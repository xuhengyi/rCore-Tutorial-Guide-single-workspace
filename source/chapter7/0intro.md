# 引言

## 本章导读

本章将基于文件描述符实现父子进程之间的通信机制——**管道**（pipe）。我们还将扩展 `exec` 系统调用，使之能传递运行参数，并进一步改进 shell 程序，使其支持重定向符号 `>` 和 `<`。

## 实践体验

在 qemu 模拟器上运行本章代码：

```bash
$ cd ch7
$ cargo run
```

进入 shell 程序后，可以运行管道机制的简单测例 `ch7b_pipetest`，`ch7b_pipetest` 需要保证父进程通过管道传输给子进程的字符串不会发生变化。

测例输出大致如下：

```
>> ch7b_pipetest
Read OK, child process exited!
pipetest passed!
Shell: Process 2 exited with code 0
>>
```

同样的，也可以运行较为复杂的测例 `ch7b_pipe_large_test`，体验通过两个管道实现双向通信。

此外，在本章我们为 shell 程序支持了输入/输出重定向功能，可以将一个应用的输出保存到一个指定的文件。例如，下面的命令可以将 `ch7b_yield` 应用的输出保存在文件 `fileb` 当中，并在应用执行完毕之后确认它的输出：

```
>> ch7b_yield > fileb
Shell: Process 2 exited with code 0
>> ch7b_cat fileb
Hello, I am process 2.
Back in process 2, iteration 0.
Back in process 2, iteration 1.
Back in process 2, iteration 2.
Back in process 2, iteration 3.
Back in process 2, iteration 4.
yield pass.

Shell: Process 2 exited with code 0
>>
```

## 本章代码树

```
rCore-Tutorial-in-single-workspace/
├── ch7/
│   ├── Cargo.toml          # 项目配置（含 signal、signal-impl）
│   ├── build.rs
│   └── src/
│       ├── main.rs         # 内核主函数（含信号初始化）
│       ├── fs.rs           # 文件系统封装
│       ├── process.rs      # 进程管理（含信号处理）
│       ├── processor.rs    # 处理器管理
│       └── virtio_block.rs # VirtIO 块设备驱动
├── signal/                 # 信号模块定义
├── signal-impl/            # 信号模块实现
├── signal-defs/            # 信号定义（用户与内核共享）
├── syscall/                # 系统调用（含信号相关 syscall）
└── ...
```

本章相比第六章的主要变化：

- **管道机制**：实现 `sys_pipe`，支持父子进程间单向通信
- **命令行参数**：扩展 `sys_exec`，支持传递命令行参数（`argc`、`argv`）
- **I/O 重定向**：实现 `sys_dup`，支持标准输入/输出重定向（`<`、`>`）
- **信号机制**：引入信号模块，支持进程间信号通信（`kill`、`sigaction`、`sigprocmask`、`sigreturn`）

## 关键概念

- **管道（Pipe）**：基于循环缓冲区的单向通信机制，通过文件描述符访问
- **命令行参数**：通过 `exec` 传递，在用户栈上组织为 `argv` 数组
- **文件描述符复制**：`dup` 系统调用，用于实现重定向
- **信号（Signal）**：进程间异步通知机制
