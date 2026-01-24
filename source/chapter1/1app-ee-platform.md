# 应用程序执行环境与平台支持

## 执行应用程序

我们先从最简单的 Rust `Hello, world` 程序开始，用 Cargo 工具创建 Rust 项目。

```bash
$ cargo new os
```

此时，项目的文件结构如下：

```
os
├── Cargo.toml
└── src
    └── main.rs
```

其中 `Cargo.toml` 中保存了项目的库依赖、作者信息等。

cargo 为我们准备好了 `Hello world!` 源代码：

```rust
fn main() {
    println!("Hello, world!");
}
```

输入 `cargo run` 构建并运行项目：

```bash
$ cargo run
   Compiling os v0.1.0 (/path/to/os)
    Finished dev [unoptimized + debuginfo] target(s) in 1.15s
     Running `target/debug/os`
Hello, world!
```

我们在屏幕上看到了一行 `Hello, world!`，但为了打印出 `Hello, world!`，我们需要的不止几行源代码。

## 理解应用程序执行环境

在现代通用操作系统（如 Linux）上运行应用程序，需要多层次的执行环境栈支持：

```
┌─────────────────────────────────────┐
│        应用程序 (Application)        │
├─────────────────────────────────────┤
│   标准库/第三方库 (Library)         │
├─────────────────────────────────────┤
│   系统调用接口 (System Call)        │
├─────────────────────────────────────┤
│      操作系统内核 (OS Kernel)       │
├─────────────────────────────────────┤
│    硬件平台 (Hardware Platform)     │
└─────────────────────────────────────┘
```

我们的应用程序通过调用标准库或第三方库提供的接口，仅需少量源代码就能完成复杂的功能；`Hello, world!` 程序调用的 `println!` 宏就是由 Rust 标准库 std 和 GNU Libc 等提供的。这些库属于应用程序的 **执行环境** (Execution Environment)，而它们的实现又依赖于操作系统提供的系统调用。

## 平台与目标三元组

编译器在编译、链接得到可执行文件时需要知道，程序要在哪个 **平台** (Platform) 上运行，**目标三元组** (Target Triplet) 描述了目标平台的 CPU 指令集、操作系统类型和标准运行时库。

我们研究一下现在 `Hello, world!` 程序的目标三元组是什么：

```bash
$ rustc --version --verbose
   rustc 1.xx.0
   host: x86_64-unknown-linux-gnu
```

其中 host 一项表明默认目标平台是 `x86_64-unknown-linux-gnu`，CPU 架构是 x86_64，CPU 厂商是 unknown，操作系统是 linux，运行时库是 gnu libc。

接下来，我们希望把 `Hello, world!` 移植到 RISC-V 目标平台 `riscv64gc-unknown-none-elf` 上运行。

> **注意**：`riscv64gc-unknown-none-elf` 的 CPU 架构是 riscv64gc，厂商是 unknown，操作系统是 none，elf 表示没有标准的运行时库。没有任何系统调用的封装支持，但可以生成 ELF 格式的执行程序。
> 
> 我们不选择有 linux-gnu 支持的 `riscv64gc-unknown-linux-gnu`，是因为我们的目标是开发操作系统内核，而非在 linux 系统上运行的应用程序。

## 修改目标平台

将程序的目标平台换成 `riscv64gc-unknown-none-elf`，试试看会发生什么：

```bash
$ cargo run --target riscv64gc-unknown-none-elf
   Compiling os v0.1.0 (/path/to/os)
error[E0463]: can't find crate for `std`
  |
  = note: the `riscv64gc-unknown-none-elf` target may not be installed
```

报错的原因是目标平台上确实没有 Rust 标准库 std，也不存在任何受 OS 支持的系统调用。这样的平台被我们称为 **裸机平台** (bare-metal)。

幸运的是，除了 std 之外，Rust 还有一个不需要任何操作系统支持的核心库 core，它包含了 Rust 语言相当一部分核心机制，可以满足本门课程的需求。有很多第三方库也不依赖标准库 std，而仅仅依赖核心库 core。

为了以裸机平台为目标编译程序，我们要将对标准库 std 的引用换成核心库 core。
