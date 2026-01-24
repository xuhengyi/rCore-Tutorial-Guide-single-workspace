# 引言

## 本章导读

我们将开发一个用户 **终端** (Terminal) 或 **命令行** (Command Line Application, 俗称 **Shell**)，形成用户与操作系统进行交互的命令行界面 (Command Line Interface)。

为此，我们要对任务建立新的抽象：**进程**，并实现若干基于 **进程** 的强大系统调用。

> **注意**：**任务和进程的关系与区别**
> 
> 第三章提到的 **任务** 是这里提到的 **进程** 的初级阶段，与任务相比，进程能在运行中创建 **子进程**、用新的 **程序** 内容覆盖已有的 **程序** 内容、可管理更多物理或虚拟 **资源**。

## 实践体验

在 qemu 模拟器上运行本章代码：

```bash
$ cd ch5
$ cargo run
```

待内核初始化完毕之后，将在屏幕上打印可用的应用列表并进入 shell 程序：

```
/**** APPS ****
ch5b_initproc
ch5b_user_shell
ch5b_forktest
...
**************/
Rust user shell
>>
```

可以通过输入应用名来执行应用，例如：

```
>> ch5b_forktest_simple
```

## 本章代码树

```
rCore-Tutorial-in-single-workspace/
├── ch5/
│   ├── Cargo.toml          # 项目配置文件
│   ├── build.rs            # 构建脚本（基于应用名）
│   ├── README.md           # 本章说明文档
│   └── src/
│       ├── main.rs         # 内核主函数，包含进程调度
│       ├── process.rs      # 进程管理（fork/exec）
│       └── processor.rs     # 处理器管理
├── task-manage/            # 任务管理库
│   └── src/
│       ├── proc_manage.rs   # 进程管理器（PManager）
│       ├── proc_rel.rs      # 进程关系管理
│       └── id.rs            # 进程 ID 管理
├── kernel-vm/              # 虚拟内存管理库
├── kernel-context/         # 内核上下文管理库
├── kernel-alloc/           # 内核堆分配器
├── linker/                 # 链接脚本和应用程序管理库
├── syscall/                # 系统调用库（新增进程相关系统调用）
└── console/                # 控制台库
```

本章代码相比第四章的主要变化：

- **进程管理**：使用 `rcore-task-manage` 库管理进程和父子关系
- **进程 ID**：使用 `ProcId` 来标识进程
- **进程创建**：实现 `fork` 系统调用，创建子进程
- **程序加载**：实现 `exec` 系统调用，加载新程序
- **进程等待**：实现 `wait`/`waitpid` 系统调用，等待子进程结束
- **Shell 程序**：实现用户终端，可以执行其他程序

## 关键概念

- **进程**：拥有独立地址空间和资源的执行实体
- **进程 ID (PID)**：唯一标识一个进程的整数
- **父子进程关系**：进程之间的层次关系
- **fork**：创建子进程的系统调用
- **exec**：加载新程序替换当前进程的系统调用
- **wait/waitpid**：等待子进程结束并回收资源的系统调用
- **Shell**：用户与操作系统交互的命令行界面
