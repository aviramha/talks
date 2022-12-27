---
# try also 'default' to start simple
theme: academic
# background: backgrounds/julian-hochgesang-SiflIx5IlRI-unsplash.jpg
# apply any windi css classes to the current slide
class: 'text-center'
# https://sli.dev/custom/highlighters.html
highlighter: shiki
# show line numbers in code blocks
lineNumbers: false
# some information about the slides, markdown enabled
info: |
  Talk about introducing Rust to the workplace.
# persist drawings in exports and build
drawings:
  persist: false
author: Aviram Hassan
authorUrl: https://github.com/aviramha/
hideInToc: ture
---

# Dark Arts of Rust <mdi-language-rust/>

Illuminating guide to the dark arts of Rust 🦀

---
layout: center
class: text-center
---

# About Me 🤓 

Hey! Nice to meet you 👋🏽. I am Aviram Hassan (he/him).

I am CEO of [MetalBear](https://metalbear.co)

Currently working on [mirrord.dev](https://mirrord.dev)


---

# Quick Poll

<v-clicks>

- syscalls
- unsafe
- extern
- ABI
- Naked
- ffi
- trampoline
- TLS (not the protocol!)

</v-clicks>


---

# What are we going to talk about?

- This presentation is to showcase a bit of advanced, low level features of Rust.
- Rust is usually viewed as a safe haven for low level engineers, but usually they don't get too low.
- This presentation is going to take us deep ;)
- We'll use the open source of mirrord as examples!

<!--

-->

---

# Quick Infomercial 📺

- mirrord lets backend engineers run their service locally in the context of the remote cluster.
- It is similar to sandbox/development environment but it is completely user mode and not a VM.
- It is a Rust application that uses a lot of low level features of Rust.

---


# Unsafe

- Unsafe is a famous keyword in Rust, you know you need to use it when doing "unsafe operations". 
- What does that mean?
  - Pointer access
  - Dereferencing (with exceptions)
  - Calling unsafe functions (usually FFI)
- What it doesn't do:
  - Anything in runtime - it is just a hint to the compiler that you know what you are doing.

<!--
Maybe talk about SEH/try except that would be a good example of how I viewed unsafe when I first heard about it.
-->

---

# syscalls

- Syscalls are the way to call the kernel from user mode.
- They are the only way to do things like file IO, network IO, etc.
- Syscalls are usually done by calling the `syscall` instruction.

---

# ABI

- Application Binary Interface
- How parameters are passed to functions and return value is returned
- Which registers are used, how the stack is used.
- For example SystemV AMD64 ABI (C calling convention on Linux)
<img src="/abi.png" style="height: 100%; width: 100%; margin-left: auto; margin-right: auto;" class="center"/>

--- 

# FFI

- Foreign Function Interface
- How to call functions from other languages
- Calling from X ABI to Y ABI

---

# extern

- extern keyword is for specifying a symbol to be imported from an external resource (dylib/so/dll) or that it is externally
  visible (when we want other "code" to use it)

- when exporting, would be used with `no_mangle`

```rs
#[link(name = "my_c_library")]
extern "C" {
    fn my_c_function(x: i32) -> bool;
}
```

```rs
#[no_mangle]
pub extern "C" fn callable_from_c(x: i32) -> bool {
    x % 3 == 0
}
```


---

# TLS

- Thread Local Storage
- Store data per thread
- We use it for bypassing the hooking mechanism in mirrord


---

```rs

thread_local!(pub(crate) static DETOUR_BYPASS: RefCell<bool> = RefCell::new(false));

pub(crate) fn detour_bypass_on() {
    DETOUR_BYPASS.with(|enabled| *enabled.borrow_mut() = true);
}

pub(crate) fn detour_bypass_off() {
    DETOUR_BYPASS.with(|enabled| *enabled.borrow_mut() = false);
}

pub(crate) struct DetourGuard;

impl DetourGuard {
    /// Create a new DetourGuard if it's not already enabled.
    pub(crate) fn new() -> Option<Self> {
        DETOUR_BYPASS.with(|enabled| {
            if *enabled.borrow() {
                None
            } else {
                *enabled.borrow_mut() = true;
                Some(Self)
            }
        })
    }
}
```

---

```rs
impl Drop for DetourGuard {
    fn drop(&mut self) {
        detour_bypass_off();
    }
}

/// Hook for `libc::open`.
///
/// **Bypassed** by `raw_path`s that match `IGNORE_FILES` regex.
#[hook_fn]
pub(super) unsafe extern "C" fn open_detour(
    raw_path: *const c_char,
    open_flags: c_int,
    mut args: ...
) -> RawFd {
    let mode: c_int = args.arg();
    let guard = DetourGuard::new();
    if guard.is_none() {
        FN_OPEN(raw_path, open_flags, mode)
    } else {
        open_logic(raw_path, open_flags, mode)
    }
}
```
---

# Trampoline

- What happens if we want to convert ABI x to ABI y?
- We call such code a trampoline - a function that calls another function with different ABI.
- This is a very common pattern in FFI.
- Unfortunately, Rust doesn't let you extend and add ABIs to the language, so we have to use assembly to build our trampoline!

---

# Naked

- Naked functions are functions that don't have a prologue and epilogue.
- This means that we have to manually push/pop registers and adjust the stack pointer.
- This is useful when we want to write assembly code in Rust.


--- 

# Bundling it together

- mirrord hooks libc calls to intercept them and redirect them to the remote cluster.
- In most scenarios, we just need to define our detour functions in a "C" ABI, same as the libc functions.
- Go doesn't use libc on Linux. It calls syscalls directly!
- We have to hook Go functions, and Go has has its own ABI.
- We have to write a trampoline to convert Go ABI to C ABI.


---

# Finding which Go version is used

```rs
    if let Some(version_symbol) =
        hook_manager.resolve_symbol_main_module("runtime.buildVersion.str")
    {
        // Version str is `go1.xx` - take only last 4 characters.
        let version = unsafe {
            std::str::from_utf8_unchecked(std::slice::from_raw_parts(
                version_symbol.0.add(2) as *const u8,
                4,
            ))
        };
    }
```

---

# Our simple trampoline

- This example lacks some before-after-op (in the real code, we have to save stack, call some prep functions)

```rs
#[cfg(target_os = "linux")]
#[cfg(target_arch = "x86_64")]
#[naked]
unsafe extern "C" fn go_raw_syscall_detour() {
    asm!(
        "mov rsi, QWORD PTR [rsp+0x10]",
        "mov rdx, QWORD PTR [rsp+0x18]",
        "mov rcx, QWORD PTR [rsp+0x20]",
        "mov rdi, QWORD PTR [rsp+0x8]",
        "call c_abi_syscall_handler",
        "mov  QWORD PTR [rsp+0x28],rax",
        "mov  QWORD PTR [rsp+0x30],rdx",
        "mov  QWORD PTR [rsp+0x38],0x0",
        "ret",
        options(noreturn),
    );
}

#[no_mangle]
unsafe extern "C" fn c_abi_syscall_handler(
    syscall: i64,
    param1: i64,
    param2: i64,
    param3: i64,
) -> i32 {    
    debug!("c_abi_sycall_handler received syscall: {syscall:?}");
    let res = match syscall {        
        libc::SYS_socket => {
            let sock = libc::socket(param1 as i32, param2 as i32, param3 as i32);            
            sock
        }
        _ => libc::syscall(syscall, param1, param2, param3) as i32,
    };
    return res;
}

```


---
class: 'text-center'
layout: center
---

# Questions

- Available at aviram@metalbear.co
- aviramyh - @Twitter
