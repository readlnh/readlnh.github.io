---
title: 笔记1-独立的Rust应用
date: 2020-9-22 21:11
tags: 
    - Rust
    - os
---


第一步，先通过cargo创建一个新项目：
```
cargo new rtoyos
```
项目结构如下：
```
├── Cargo.toml
├── src
│   ├── main.rs
```
但是Rust项目默认都会链接到`std`标准库，而标准库会用到很多操作系统的功能，诸如线程，文件，网络等。所以我们要做的第一步就是实现一个不依赖于任何操作系统功能的Rust程序（裸机程序）。

## 禁用标准库
我们可以通过`![no_std]`来禁用`std`库。

```rust
#![no_std]

fn main() {
    println!("Hello, world!");
}
```

运行`cargo build`，可以看到如下错误：
```
error: cannot find macro `println` in this scope
 --> src/main.rs:4:5
  |
4 |     println!("Hello, world!");
  |     ^^^^^^^

error: `#[panic_handler]` function required, but not found

error: language item required, but not found: `eh_personality`

error: aborting due to 3 previous errors
```

### println! 宏
`println!`宏依赖于`std`库，这里我们就先不使用它了。

### panic 处理函数
第二个错误说需要一个`#[panic_handler]`函数，这个函数会在程序`panic`的时候被调用。默认情况下，`std`中有`panic`的实现，然而由于我们是在`[no_std]`环境中，只能自己实现一个`panic`函数了。

```rust
use core::panic::PanicInfo;

#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    loop {}
}
```

这里我们用了`core`库，这个库不需要操作系统支持。`PanicInfo`类型的参数会包含`panic`发生的文件，代码行数等错误信息。另外`!`标记表示这个函数的返回类型为`never type`，即永远不会返回。

### `eh_personality` 语义项
语义项是编译器内部所需的特殊函数或类型，例如`Copy` trait（`#[lang = "copy"]`），又或是之前的`panic_handler`。`eh_personality`是用来标记函数实现堆栈展开的语义项，该语义与`panic`有关。

> 堆栈展开 (Stack Unwinding) 
>
>通常当程序出现了异常时，从异常点开始会沿着 caller 调用栈一层一层回溯，直到找到某个函数能够捕获这个异常或终止程序。这个过程称为堆栈展开。
>
>当程序出现异常时，我们需要沿着调用栈一层层回溯上去回收每个 caller 中定义的局部变量（这里的回收包括 C++ 的 RAII 的析构以及 Rust 的 drop 等）避免造成捕获异常并恢复后的内存溢出。
>
>而在 Rust 中，panic 证明程序出现了错误，我们则会对于每个 caller 函数调用依次这个被标记为堆栈展开处理函数的函数进行清理。
>
>这个处理函数是一个依赖于操作系统的复杂过程，在标准库中实现。但是我们禁用了标准库使得编译器找不到该过程的实现函数了。

为了简单起见，我们将堆栈展开禁用，在`panic`发生时直接`abort`而不是依次获取堆栈信息。

在`Cargo.toml`中进行配置：

```toml
[profile.dev]
panic = "abort"

[profile.release]
panic = "abort"
```

现在,错误信息变成了：
```bash
error: requires `start` lang_item
```

## 移除C运行时依赖
大部分语言都有一个运行时（Runtime），这个运行时会在`main`函数之前被调用。以`Rust`为例，一个典型的链接了标准库的Rust程序会先跳转到C语言运行时环境`crt0`（C runtime zero），`crt0`会接着跳转到Rust运行时的入口点，这个入口点是被`start`语义所标记的。最后，Rust的运行时会调用`main`函数。

由于我们的程序无法访问标准库也就无法访问`crt0`和Rust运行时，所以我们需要定义我们自己的入口点。这里即使覆写`start`语义也是没用的，因为它仍然需要`crt0`的支持，所以我们要做的是直接覆写整个`ctr0`入口点。

```rust
#![no_std]
#![no_main]

use core::panic::PanicInfo;

#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    loop {}
}


#[no_mangle]
pub extern "C" fn _start() -> ! {
    loop {}
}
```

我们使用`#![no_main]`属性来告诉编译器我们不使用常规入口点，并使用`_start`函数作为新的入口点（`_start`是大部分系统的默认入口点）。这里我们使用`#[no_mangle]`标记来禁用编译时的重命名，保证编译器生成的函数名仍然为`_start`，并使用`extern "C"`表示这是一个C语言函数。

此时我们再编译，发现错误变成了链接错误。

## 解决链接错误
链接器（Linker）是一个程序，它将生成的目标文件组合为一个可执行文件。不同的操作系统如 Windows、macOS 或 Linux，规定了不同的可执行文件格式，因此也各有自己的链接器，抛出不同的错误；但这些错误的根本原因还是相同的：链接器的默认配置假定程序依赖于 C 语言的运行时环境，但我们的程序并不依赖于它。以x86-64-linux为例，错误如下：

```bash
error: linking with `cc` failed: exit code: 1
  |
  = note: "cc" "-Wl,--as-needed" "-Wl,-z,noexecstack" "-m64" 
  ...
          collect2: error: ld returned 1 exit status
```

我们只需要使用`-C link-arg`flag（`cargo rustc -- -C link-arg=-nostartfiles`）或是干脆选择编译为裸机目标（例如：`cargo build --target thumbv7em-none-eabihf`）即可。
