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

# What are we going to talk about?

- This presentation is to showcase some advanced, low level features of Rust.
- Rust is usually viewed as a safe haven for low level engineers, but usually they don't get too low.
- This presentation is going to take us deep ;)
- We'll use the open source of mirrord as examples!

<!--

-->


---

# Quick Poll

<v-clicks>

- syscalls
- unsafe
- extern
- ABI
- Naked
- FFI
- trampoline
- TLS (not the protocol!)

</v-clicks>

---


# Quick Infomercial 📺

- mirrord lets backend engineers run their service locally in the context of the remote cluster.
- We're going to focus on the "layer" (so/dylib) component of mirrord.
- mirrord-layer is injected into the users' process(es)
- It is similar to sandbox but it is completely user mode and not a VM.
- In order to achieve that, we hook libc functions that wrap syscalls and 
    decide what happens locally and what happens remote.

---


# syscalls

- Syscalls are the way to call the kernel from user mode.
- They are the only way to do things like file IO, network IO, etc.
- Syscalls are usually done by calling the `syscall` instruction.
- Most languages/frameworks use libc as a wrapper to do syscalls.

---

# Initializing..

- First we need mirrord-layer to run when loaded as so/dylib, without the original process or anyone else calling us
- ctor to the rescue


```rs
/// The one true start of mirrord-layer.
#[ctor]
fn mirrord_layer_entry_point() {
    start()
}
```

<div v-click>

```rs
/// The one true start of mirrord-layer.
#[ctor]
fn mirrord_layer_entry_point() {
    let _ = panic::catch_unwind(|| {
        if let Err(fail) = layer_pre_initialization() {
                eprintln!("mirrord layer setup failed with {:?}", fail);
                std::process::exit(-1)  
            }
        }
    };
```

</div>

---

# Initializing++

- We are hooking the relevant libc functions
- Creating a thread (using Tokio) that will run in background and communicate with the agent.

```rs
replace!(hook_manager, "open", open_detour, FnOpen, FN_OPEN);
// sum this
#[macro_export]
macro_rules! replace {
    ($hook_manager:expr, $func:expr, $detour_function:expr, $detour_type:ty, $hook_fn:expr) => {{
        let intercept = |hook_manager: &mut $crate::hooks::HookManager,
                         symbol_name,
                         detour: $detour_type|
         -> $crate::error::Result<$detour_type> {
            let replaced =
                hook_manager.hook_export_or_any(symbol_name, detour as *mut libc::c_void)?;
            let original_fn: $detour_type = std::mem::transmute(replaced);

            tracing::trace!("hooked {symbol_name:?}");
            Ok(original_fn)
        };

        let _ = intercept($hook_manager, $func, $detour_function)
            .and_then(|hooked| Ok($hook_fn.set(hooked).unwrap()));
    }};
}
    
```

---

# Example hook

```rs
/// Hook for `libc::open`.
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

# #[naked]

- Naked functions are functions that don't have a prologue and epilogue.
- This means that we have to manually manage the registers and stack.
- The prologue/epilogue are usually done by the compiler to match the ABI.


--- 

# Bundling it together

- mirrord hooks libc calls to intercept them and redirect them to the remote cluster.
- In most scenarios, we just need to define our detour functions in a "C" ABI, same as the libc functions.
- Go doesn't use libc on Linux. It calls syscalls directly!
- We have to hook Go functions, and Go has has its own ABI.
- We have to write a trampoline to convert Go ABI to C ABI.


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

```

--- 

```rs
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

# We used all dark features in few snippets!


- syscalls - we hook those
- unsafe - everywhere!
- extern + FFI + ABI = how we call other languages/export to other languages
- Naked + Trampoline - to convert ABIs
- TLS - to bypass the hooking mechanism


---
class: 'text-center'
layout: center
---

# Questions

- Available at aviram@metalbear.co
- https://hachyderm.io/@aviram
- @aviramyh - Twitter
