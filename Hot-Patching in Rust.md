## Hot-Patching in Rust

Is hot-patching supported by Rust? How does one hot-patch Rust programs? How is hot-patching in Rust compared to that in C/C++? These are the questions we're hopefully able to answer in this report.

### Function Detour in C/C++

Start with a simple example so we're on the same page - *function detour* in C/C++:

```c++
#include <iostream>
#include <Windows.h>

void func1() { std::cout << "func1" << std::endl; }
void func2() { std::cout << "func2" << std::endl; }

void detour(void* sourceFunc, void* targetFunc) {
    DWORD originProtect;
    char* source = (char*)sourceFunc;
    char* target = (char*)targetFunc;
    int offset = target - source - 5;

    VirtualProtect(source, 5, PAGE_EXECUTE_READWRITE, &originProtect);
    *source = 0xE9;
    *(int*)(source + 1) = offset;
    VirtualProtect(source, 5, originProtect, &originProtect);
}

int main() {
    func1();
    detour(func1, func2);
    func1();
    return 0;
}
```

This is a kind of old-fashioned instruction injection on 32-bit Windows. In the example, what happens is that `func1` gets patched to jump to `func2` in substitution to executing its own logics, thus effectively redirecting the call of `func1` to `func2`. Fire up the program in debug mode and it outputs as intended:

```
func1
func2
```

### Function Detour in Rust

So how's *function detour* supposed to be in Rust? Let's try to replicate the above example in Rust. Coming in at first I wasn't sure about how raw pointer manipulation and system API calls are presented in Rust. Hopefully, as a system-level programming language, Rust doesn't come short in terms of low-level operations.

Rust supports function address peeking through `usize` conversions. We can calculate function address offset for the relative jump by converting function pointers to `usize`:

```rust
fn caclPatchOffset(source: fn(), target: fn()) -> usize{
	target as usize - source as usize - 5
}
```

Raw pointer manipulation is supported as well:

```rust
fn detour(source: fn(), target: fn()) {
	unsafe { *(source as *mut u8) = 0xE9; }
}
```

Rust also has bindings to system APIs. For Windows API, `winapi` crate is provided:

```rust
use winapi;

fn detour(source: fn(), target: fn()) {
	winapi::um::memoryapi::VirtualProtect(
	    source as winapi::um::winnt::PVOID,
	    5 as winapi::shared::basetsd::SIZE_T,
	    winapi::um::winnt::PAGE_EXECUTE_READWRITE,
	    &mut originalProtect,
	);
}
```

Piece these together - it turns out we can have a direct translation of the example written in C++:

```rust
use winapi;

fn func1() { println!("func1") }
fn func2() { println!("func2") }

fn detour(source: fn(), target: fn()) {
    let mut originalProtect = 0;
    let offset = target as usize - source as usize - 5;
    unsafe {
        winapi::um::memoryapi::VirtualProtect(
            source as winapi::um::winnt::PVOID,
            5,
            winapi::um::winnt::PAGE_EXECUTE_READWRITE,
            &mut originalProtect,
        );
        *(source as *mut u8) = 0xE9;
        *((source as *mut u8).offset(1) as *mut usize) = offset;
        winapi::um::memoryapi::VirtualProtect(
            source as winapi::um::winnt::PVOID,
            5,
            originalProtect,
            &mut originalProtect,
        );
    }
}

fn main() {
    func1();
    detour(func1, func2);
    func1();
}
```

Compiled in debug mode, it outputs as expected:

```
func1
func2
```

### A Deeper Dive into The Function Detour in Rust

The function detour in Rust works, but how? There is a catch I only found out after a deeper dive into the assembly later in the experiment.

The `main()` in the Rust example looks like this in assembly:

```asm
00841450 | sub esp,8
00841453 | call rust-winapi.8412A0
00841458 | lea eax,dword ptr ds:[8412A0]
0084145E | mov dword ptr ss:[esp],eax
00841461 | lea eax,dword ptr ds:[8412F0]
00841467 | mov dword ptr ss:[esp+4],eax
0084146B | call rust-winapi.841340
00841470 | call rust-winapi.8412A0
00841475 | add esp,8
00841478 | ret
```

`call rust-winapi.8412A0` at line 3 and 8 is the call to `func1`.

`call rust-winapi.841340` at line 7 is the call to `detour`.

`func2` sets in address `rust-winapi.8412F0`.

`func1` before `detour` is called:

```assembly
008412A0 | push esi
008412A1 | sub esp,30
008412A4 | xor eax,eax
008412A6 | mov ecx,dword ptr ds:[85C1B8]
008412AC | mov edx,dword ptr ds:[85C1BC]
........ | ...
008412EA | ret
```

`func1` after `detour`:

```assembly
008412A0 | jmp rust-winapi.8412F0
008412A5 | ror byte ptr ds:[ebx-7A3E47F3],0
008412AC | mov edx,dword ptr ds:[85C1BC]
........ | ...
008412EA | ret
```

As we can see, `detour` changed the 5 bytes from `008412A0` to `008412A4` into a `jmp` instruction, which is a direct substitution to the function body of `func1`. This is the one of the oldest tricks in the book of function detours.

However this particular type of function detour has multi-threaded issues and it just happens to be present in the example - original instruction `xor eax,eax` at `008412A4` gets cut in half, rendering it corrupted.

I carried out this investigation because I cannot find any Rustc arguments for hot-patchable function.  Thus to ensure multi-thread safety, we need to use mutex for function detours. Rust actually has a crate named `detour` particularly designed for this purpose. Crate `detour` has a nice capsulation which makes the function detour logic less verbose:

```rust
use std::error::Error;
use detour::GenericDetour;

fn func1() { println!("func1") }
fn func2() { println!("func2") }

fn main() -> Result<(), Box<dyn Error>> {
    func1();
    let hook = unsafe { GenericDetour::<fn()>::new(func1, func2)? };
    unsafe { hook.enable()? };
    func1();
    Ok(())
}
```

It has recoverable mechanics and uses mutexes to ensure multi-thread safety:

```rust
unsafe fn toggle(&self, enabled: bool) -> Result<()> {
    let _guard = memory::POOL.lock().unwrap();

    if self.enabled.load(Ordering::SeqCst) == enabled {
      return Ok(());
    }

    // Runtime code is by default only read-execute
    let _handle = {
      let area = (*self.patcher.get()).area();
      region::protect_with_handle(
        area.as_ptr(),
        area.len(),
        region::Protection::ReadWriteExecute,
      )
    }?;

    // Copy either the detour or the original bytes of the function
    (*self.patcher.get()).toggle(enabled);
    self.enabled.store(enabled, Ordering::SeqCst);
    Ok(())
}
```

Thus crate `detour` should be taken into consideration when function detour is necessary in Rust.

### Hot-Patching in Rust
