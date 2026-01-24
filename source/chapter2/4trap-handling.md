# 实现特权级的切换

## RISC-V 特权级切换

### 特权级切换的起因

批处理操作系统为了建立好应用程序的执行环境，需要在执行应用程序前进行一些初始化工作，并监控应用程序的执行，具体体现在：

- 启动应用程序时，需要初始化应用程序的用户态上下文，并能切换到用户态执行应用程序；
- 应用程序发起系统调用后，需要切换到批处理操作系统中进行处理；
- 应用程序执行出错时，批处理操作系统要杀死该应用并加载运行下一个应用；
- 应用程序执行结束时，批处理操作系统要加载运行下一个应用。

这些处理都涉及到特权级切换，因此都需要硬件和操作系统协同提供的特权级切换机制。

### 特权级切换相关的控制状态寄存器

本章中我们仅考虑当 CPU 在 U 特权级运行用户程序的时候触发 Trap，并切换到 S 特权级的批处理操作系统进行处理。

| CSR 名 | 该 CSR 与 Trap 相关的功能 |
|--------|--------------------------|
| sstatus | `SPP` 等字段给出 Trap 发生之前 CPU 处在哪个特权级（S/U）等信息 |
| sepc | 当 Trap 是一个异常的时候，记录 Trap 发生之前执行的最后一条指令的地址 |
| scause | 描述 Trap 的原因 |
| stval | 给出 Trap 附加信息 |
| stvec | 控制 Trap 处理代码的入口地址 |

特权级切换的具体过程一部分由硬件直接完成，另一部分则需要由操作系统来实现。

### 特权级切换的硬件控制机制

当 CPU 执行完一条指令并准备从用户特权级陷入（Trap）到 S 特权级的时候，硬件会自动完成如下这些事情：

- `sstatus` 的 `SPP` 字段会被修改为 CPU 当前的特权级（U/S）。
- `sepc` 会被修改为 Trap 处理完成后默认会执行的下一条指令的地址。
- `scause/stval` 分别会被修改成这次 Trap 的原因以及相关的附加信息。
- CPU 会跳转到 `stvec` 所设置的 Trap 处理入口地址，并将当前特权级设置为 S，然后从 Trap 处理入口地址处开始执行。

> **注意**：**stvec 相关细节**
> 
> 在 RV64 中，`stvec` 是一个 64 位的 CSR，在中断使能的情况下，保存了中断处理的入口地址。它有两个字段：
> 
> - MODE 位于 [1:0]，长度为 2 bits；
> - BASE 位于 [63:2]，长度为 62 bits。
> 
> 当 MODE 字段为 0 的时候，`stvec` 被设置为 Direct 模式，此时进入 S 模式的 Trap 无论原因如何，处理 Trap 的入口地址都是 `BASE<<2`，CPU 会跳转到这个地方进行异常处理。本书中我们只会将 `stvec` 设置为 Direct 模式。

而当 CPU 完成 Trap 处理准备返回的时候，需要通过一条 S 特权级的特权指令 `sret` 来完成，这一条指令具体完成以下功能：

- CPU 会将当前的特权级按照 `sstatus` 的 `SPP` 字段设置为 U 或者 S；
- CPU 会跳转到 `sepc` 寄存器指向的那条指令，然后继续执行。

## LocalContext 和 Trap 处理

在本项目的代码框架中，我们使用 `kernel-context` 库提供的 `LocalContext` 来管理用户上下文和 Trap 处理。

### LocalContext 结构

`LocalContext` 结构体定义如下：

```rust
#[repr(C)]
pub struct LocalContext {
    sctx: usize,
    x: [usize; 31],      // 通用寄存器 x1~x31（x0 硬编码为 0）
    sepc: usize,         // 程序计数器
    supervisor: bool,    // 是否以特权态切换
    interrupt: bool,     // 线程中断是否开启
}
```

这个结构体保存了：
- 通用寄存器（除了 x0，它总是 0）
- 程序计数器 `sepc`
- 特权级和中断状态标志

### 创建用户上下文

在运行应用程序之前，我们需要创建一个用户上下文：

```rust
let mut ctx = LocalContext::user(app_base);
```

`LocalContext::user()` 方法会创建一个用户态上下文，其中：
- `sepc` 设置为应用程序入口地址 `app_base`
- `supervisor` 设置为 `false`（用户态）
- `interrupt` 设置为 `true`（允许中断）

### 执行用户程序

执行用户程序通过 `ctx.execute()` 方法：

```rust
unsafe { ctx.execute() };
```

`execute()` 方法会：
1. 设置 `stvec` 指向 Trap 处理入口
2. 保存当前内核上下文到 `sscratch`
3. 恢复用户上下文（通用寄存器和 `sepc`）
4. 通过 `sret` 指令切换到用户态执行

当发生 Trap 时，硬件会：
1. 保存当前状态到 CSR（`sstatus`、`sepc` 等）
2. 跳转到 `stvec` 指向的 Trap 处理入口
3. 在 Trap 处理入口，会交换 `sp` 和 `sscratch`，从而切换到内核栈

### Trap 处理

在 `ch2/src/main.rs` 中，Trap 处理逻辑如下：

```rust
loop {
    unsafe { ctx.execute() };

    use scause::{Exception, Trap};
    match scause::read().cause() {
        Trap::Exception(Exception::UserEnvCall) => {
            use SyscallResult::*;
            match handle_syscall(&mut ctx) {
                Done => continue,
                Exit(code) => log::info!("app{i} exit with code {code}"),
                Error(id) => log::error!("app{i} call an unsupported syscall {}", id.0),
            }
        }
        trap => log::error!("app{i} was killed because of {trap:?}"),
    }
    // 清除指令缓存
    unsafe { core::arch::asm!("fence.i") };
    break;
}
```

当 `execute()` 返回后，我们检查 `scause` 寄存器来确定 Trap 的原因：

1. **系统调用**（`UserEnvCall`）：
   - 调用 `handle_syscall()` 处理系统调用
   - 如果系统调用正常完成，继续执行用户程序
   - 如果是 `EXIT` 系统调用，记录退出码并退出循环
   - 如果是不支持的系统调用，记录错误并退出循环

2. **其他异常**：
   - 记录错误信息
   - 退出循环，加载下一个应用程序

### 系统调用处理

`handle_syscall()` 函数处理系统调用：

```rust
fn handle_syscall(ctx: &mut LocalContext) -> SyscallResult {
    use syscall::{SyscallId as Id, SyscallResult as Ret};

    let id = ctx.a(7).into();  // 从 a7 寄存器读取 syscall ID
    let args = [ctx.a(0), ctx.a(1), ctx.a(2), ctx.a(3), ctx.a(4), ctx.a(5)];
    match syscall::handle(Caller { entity: 0, flow: 0 }, id, args) {
        Ret::Done(ret) => match id {
            Id::EXIT => SyscallResult::Exit(ctx.a(0)),  // EXIT 系统调用
            _ => {
                *ctx.a_mut(0) = ret as _;  // 将返回值写入 a0
                ctx.move_next();            // 将 sepc 移到下一条指令
                SyscallResult::Done
            }
        },
        Ret::Unsupported(id) => SyscallResult::Error(id),
    }
}
```

系统调用处理流程：

1. 从上下文读取 syscall ID（`a7` 寄存器）和参数（`a0~a5` 寄存器）
2. 调用 `syscall::handle()` 进行系统调用分发和处理
3. 对于 `EXIT` 系统调用，返回退出码
4. 对于其他系统调用，将返回值写入 `a0`，并将 `sepc` 移到下一条指令（跳过 `ecall` 指令）
5. 返回 `Done` 表示继续执行

### 系统调用的实现

系统调用的具体实现在 `impls` 模块中：

```rust
mod impls {
    pub struct SyscallContext;

    impl syscall::IO for SyscallContext {
        fn write(&self, _caller: syscall::Caller, fd: usize, buf: usize, count: usize) -> isize {
            match fd {
                STDOUT | STDDEBUG => {
                    print!("{}", unsafe {
                        core::str::from_utf8_unchecked(core::slice::from_raw_parts(
                            buf as *const u8,
                            count,
                        ))
                    });
                    count as _
                }
                _ => {
                    rcore_console::log::error!("unsupported fd: {fd}");
                    -1
                }
            }
        }
    }

    impl syscall::Process for SyscallContext {
        #[inline]
        fn exit(&self, _caller: syscall::Caller, _status: usize) -> isize {
            0
        }
    }
}
```

- `sys_write`：将传入的位于应用程序内的缓冲区的开始地址和长度转化为一个字符串 `&str`，然后使用批处理操作系统已经实现的 `print!` 宏打印出来。
- `sys_exit`：返回 0，实际的退出处理在 `handle_syscall` 中完成。

## 执行应用程序的完整流程

当批处理操作系统初始化完成，或者是某个应用程序运行结束或出错的时候，我们要切换到下一个应用程序。此时 CPU 运行在 S 特权级，而它希望能够切换到 U 特权级。

在 RISC-V 架构中，唯一一种能够使得 CPU 特权级下降的方法就是通过 Trap 返回系列指令，比如 `sret`。

运行应用程序之前要完成如下这些工作：

1. 创建用户上下文 `LocalContext::user(app_base)`，其中 `sepc` 设置为应用程序入口点
2. 设置用户栈指针
3. 调用 `ctx.execute()`，它会：
   - 设置 `stvec` 指向 Trap 处理入口
   - 保存内核上下文到 `sscratch`
   - 恢复用户上下文
   - 通过 `sret` 指令切换到用户态执行

当用户程序执行 `ecall` 指令时：

1. 硬件自动保存状态到 CSR（`sstatus`、`sepc`、`scause`、`stval`）
2. 跳转到 `stvec` 指向的 Trap 处理入口
3. Trap 处理入口交换 `sp` 和 `sscratch`，切换到内核栈
4. 保存用户上下文到内核栈
5. 调用 Rust 的 Trap 处理函数
6. 处理完成后，恢复用户上下文
7. 通过 `sret` 返回到用户态继续执行

## 关键概念总结

1. **特权级**：RISC-V 定义了多个特权级（M/S/U），本章主要使用 S 和 U 特权级
2. **Trap**：从用户态切换到内核态的过程，由硬件和软件协同完成
3. **上下文切换**：保存和恢复寄存器状态，实现不同特权级之间的切换
4. **系统调用**：用户程序请求内核服务的接口，通过 `ecall` 指令触发
5. **LocalContext**：管理用户程序执行上下文的数据结构

这些概念是操作系统实现的基础，在后续章节中会继续使用和发展。
