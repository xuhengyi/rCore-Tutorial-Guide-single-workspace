# 引言

## 本章导读

大多数程序员的职业生涯都从 `Hello, world!` 开始。

```rust
println!("Hello, world!");
```

然而，要用几行代码向世界问好，并不像表面上那么简单。`Hello, world!` 程序能够编译运行，靠的是以 **编译器** 为主的开发环境和以 **操作系统** 为主的执行环境。

在本章中，我们将抽丝剥茧，一步步让 `Hello, world!` 程序脱离其依赖的执行环境，编写一个能打印 `Hello, world!` 的 OS。这趟旅途将让我们对应用程序及其执行环境有更深入的理解。

> **注意**：实验指导书存在的目的是帮助读者理解框架代码。
> 
> 为便于测试，完成编程实验时，请以框架代码为基础，不必跟着文档从零开始编写内核。

为了做到这一步，首先需要让程序不依赖于标准库，并通过编译。

接下来要让脱离了标准库的程序能输出（即支持 `println!`），这对程序的开发和调试至关重要。

最后把程序移植到内核态，构建在裸机上支持输出的最小运行时环境。

## 实践体验

本章一步步实现了支持打印字符串的简单操作系统。

获取本章代码：

```bash
$ git clone https://github.com/LearningOS/rCore-Tutorial-Code
$ cd rCore-Tutorial-Code
$ git checkout ch1
```

运行本章代码：

```bash
$ cd ch1
$ cargo run
```

预期输出：

```
Hello, world!
```

程序会打印 `Hello, world!` 然后关机。

## 本章代码树

```
rCore-Tutorial-in-single-workspace/
├── ch1/
│   ├── Cargo.toml          # cargo 项目配置文件
│   ├── build.rs            # 构建脚本，包含链接脚本
│   ├── README.md           # 本章说明文档
│   └── src/
│       └── main.rs         # 内核主函数
├── sbi/                    # SBI 调用封装库
│   └── src/
│       └── lib.rs          # SBI 接口封装
└── rust-toolchain.toml     # 整个项目的工具链版本
```

本章代码非常简洁，主要包含：

- **main.rs**：内核主函数，包含入口点设置、栈初始化、打印和关机功能
- **build.rs**：构建脚本，包含链接脚本配置，将代码放置在 `0x80200000` 地址
- **sbi库**：封装了 SBI 调用接口，提供 `console_putchar` 和 `shutdown` 等功能
