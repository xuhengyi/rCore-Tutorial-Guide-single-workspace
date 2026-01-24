# 第一章：应用程序与基本执行环境

## 目录

1. [引言](0intro.md) - 本章导读和实践体验
2. [应用程序执行环境与平台支持](1app-ee-platform.md) - 理解应用程序执行环境和目标平台
3. [移除标准库依赖](2remove-std.md) - 如何移除 Rust 标准库依赖
4. [构建裸机执行环境](4mini-rt-baremetal.md) - 在裸机上实现 Hello, world!
5. [练习](5exercise.md) - 编程作业和问答作业（已废弃，仅供参考）

## 本章目标

本章的目标是让 `Hello, world!` 程序脱离其依赖的执行环境，编写一个能打印 `Hello, world!` 的 OS。

## 主要内容

1. **理解执行环境**：了解应用程序如何在多层执行环境栈上运行
2. **目标平台**：学习如何将程序编译到 RISC-V 裸机平台
3. **移除标准库**：移除 Rust 标准库依赖，使用核心库 core
4. **裸机环境**：在裸机上设置栈、内存布局，实现基本的打印和关机功能

## 代码位置

本章的代码位于 `rCore-Tutorial-in-single-workspace/ch1/` 目录下。

主要文件：
- `src/main.rs` - 内核主函数，包含入口点、栈设置、打印和关机
- `build.rs` - 构建脚本，包含链接脚本配置
- `Cargo.toml` - 项目配置文件

## 运行方法

```bash
$ cd ch1
$ cargo build --release
$ qemu-system-riscv64 \
    -machine virt \
    -nographic \
    -bios ../rustsbi-qemu.bin \
    -device loader,file=target/riscv64gc-unknown-none-elf/release/ch1,addr=0x80200000
```

## 预期输出

```
Hello, world!
```

程序会打印 `Hello, world!` 然后关机。

## 关键概念

- **执行环境**：应用程序运行所需的软件栈
- **目标三元组**：描述目标平台的 CPU 指令集、操作系统类型和标准运行时库
- **裸机平台**：没有操作系统支持的平台
- **链接脚本**：控制程序内存布局的脚本
- **SBI**：RISC-V Supervisor Binary Interface，为操作系统提供底层服务
- **Naked 函数**：不包含序言和尾声的函数，可以在没有栈的情况下执行

## 注意事项

- 实验指导书存在的目的是帮助读者理解框架代码
- 为便于测试，完成编程实验时，请以框架代码为基础，不必跟着文档从零开始编写内核
- 本章代码非常简洁，主要目的是理解基本的执行环境和裸机启动过程
