# 管理多道程序

## 概述

内核为了管理任务，需要维护任务信息，相关内容包括：

- 任务运行状态：未初始化、准备执行、正在执行、已退出
- 任务控制块：维护任务状态和任务上下文
- 任务相关系统调用：程序主动暂停 `sys_yield` 和主动退出 `sys_exit`

## yield 系统调用

在多道程序执行中，一个典型的场景是：应用程序需要等待某个事件（如 I/O 操作）完成，在等待期间可以主动让出 CPU，让其他程序执行。

例如，蓝色应用向外设提交了一个请求，外设随即开始工作，但是它要一段时间后才能返回结果。蓝色应用于是调用 `sys_yield` 交出 CPU 使用权，内核让绿色应用继续执行。一段时间后 CPU 切换回蓝色应用，发现外设仍未返回结果，于是再次 `sys_yield`。直到第二次切换回蓝色应用，外设才处理完请求，于是蓝色应用终于可以向下执行了。

我们还会遇到很多其他需要等待其完成才能继续向下执行的事件，调用 `sys_yield` 可以避免等待过程造成的资源浪费。

```rust
/// 功能：应用主动交出 CPU 所有权并切换到其他应用。
/// 返回值：总是返回 0。
/// syscall ID：124
fn sys_yield() -> isize;
```

## TaskControlBlock 结构

在本章中，我们使用 `TaskControlBlock` 来管理每个任务：

```rust
// ch3/src/task.rs

pub struct TaskControlBlock {
    ctx: LocalContext,        // 任务上下文
    pub finish: bool,         // 任务是否已完成
    stack: [usize; 256],      // 用户栈（256 个 usize，即 2048 字节）
}
```

这个结构体包含：
- `ctx`：任务的执行上下文（`LocalContext`），保存了通用寄存器、程序计数器等
- `finish`：任务是否已经完成（退出或出错）
- `stack`：任务的用户栈空间

## 调度事件

`TaskControlBlock` 还定义了 `SchedulingEvent` 枚举来表示调度事件：

```rust
// ch3/src/task.rs

pub enum SchedulingEvent {
    None,                      // 无事件，继续执行
    Yield,                     // 任务主动让出 CPU
    Exit(usize),               // 任务退出，带退出码
    UnsupportedSyscall(SyscallId),  // 不支持的系统调用
}
```

## 系统调用处理

`TaskControlBlock::handle_syscall()` 方法处理系统调用：

```rust
// ch3/src/task.rs

pub fn handle_syscall(&mut self) -> SchedulingEvent {
    use syscall::{SyscallId as Id, SyscallResult as Ret};
    use SchedulingEvent as Event;

    let id = self.ctx.a(7).into();
    let args = [
        self.ctx.a(0),
        self.ctx.a(1),
        self.ctx.a(2),
        self.ctx.a(3),
        self.ctx.a(4),
        self.ctx.a(5),
    ];
    match syscall::handle(Caller { entity: 0, flow: 0 }, id, args) {
        Ret::Done(ret) => match id {
            Id::EXIT => Event::Exit(self.ctx.a(0)),
            Id::SCHED_YIELD => {
                *self.ctx.a_mut(0) = ret as _;
                self.ctx.move_next();
                Event::Yield
            }
            _ => {
                *self.ctx.a_mut(0) = ret as _;
                self.ctx.move_next();
                Event::None
            }
        },
        Ret::Unsupported(_) => Event::UnsupportedSyscall(id),
    }
}
```

这个方法会：
1. 从上下文读取 syscall ID 和参数
2. 调用 `syscall::handle()` 进行系统调用分发和处理
3. 对于 `EXIT` 系统调用，返回 `Exit` 事件
4. 对于 `SCHED_YIELD` 系统调用，返回 `Yield` 事件
5. 对于其他系统调用，返回 `None` 事件

## 多道程序执行循环

在 `ch3/src/main.rs` 中，多道程序的执行循环如下：

```rust
// 多道执行
let mut remain = index_mod;
let mut i = 0usize;
while remain > 0 {
    let tcb = &mut tcbs[i];
    if !tcb.finish {
        loop {
            #[cfg(not(feature = "coop"))]
            sbi::set_timer(time::read64() + 12500);
            unsafe { tcb.execute() };

            use scause::*;
            let finish = match scause::read().cause() {
                Trap::Interrupt(Interrupt::SupervisorTimer) => {
                    sbi::set_timer(u64::MAX);
                    log::trace!("app{i} timeout");
                    false
                }
                Trap::Exception(Exception::UserEnvCall) => {
                    use task::SchedulingEvent as Event;
                    match tcb.handle_syscall() {
                        Event::None => continue,
                        Event::Exit(code) => {
                            log::info!("app{i} exit with code {code}");
                            true
                        }
                        Event::Yield => {
                            log::debug!("app{i} yield");
                            false
                        }
                        Event::UnsupportedSyscall(id) => {
                            log::error!("app{i} call an unsupported syscall {}", id.0);
                            true
                        }
                    }
                }
                Trap::Exception(e) => {
                    log::error!("app{i} was killed by {e:?}");
                    true
                }
                Trap::Interrupt(ir) => {
                    log::error!("app{i} was killed by an unexpected interrupt {ir:?}");
                    true
                }
            };
            if finish {
                tcb.finish = true;
                remain -= 1;
            }
            break;
        }
    }
    i = (i + 1) % index_mod;
}
```

这个循环会：
1. 遍历所有任务控制块（轮询方式）
2. 如果任务未完成，执行该任务
3. 当发生 Trap 时：
   - **时钟中断**：如果是抢占式调度，记录超时并退出循环，切换到下一个任务
   - **系统调用**：调用 `handle_syscall()` 处理系统调用
     - `None`：继续执行当前任务
     - `Exit`：标记任务完成
     - `Yield`：退出循环，切换到下一个任务
     - `UnsupportedSyscall`：标记任务完成（出错）
   - **异常**：标记任务完成（出错）
4. 如果任务完成，减少剩余任务数
5. 切换到下一个任务（通过 `i = (i + 1) % index_mod` 实现轮询）

## 任务状态转换

任务的执行状态转换如下：

```
UnInit (未初始化)
    ↓ init()
Ready (准备运行)
    ↓ execute()
Running (正在运行)
    ↓ sys_yield() / 时钟中断
Ready (准备运行)
    ↓ execute()
Running (正在运行)
    ↓ sys_exit() / 异常
Exited (已退出)
```

## 协作式调度

当启用 `coop` feature 时，系统只使用协作式调度：

```rust
#[cfg(not(feature = "coop"))]
sbi::set_timer(time::read64() + 12500);
```

这行代码会被编译掉，不会设置时钟中断。任务只能通过 `sys_yield` 主动让出 CPU。

协作式调度的优点：
- 实现简单
- 任务可以在合适的时机让出 CPU
- 不需要处理时钟中断

协作式调度的缺点：
- 如果任务不主动让出 CPU，其他任务无法执行
- 不适合长时间运行的任务

## 关键概念总结

1. **多道程序**：多个程序同时驻留在内存中
2. **任务控制块**：管理每个任务的执行状态和上下文
3. **协作式调度**：任务主动让出 CPU（通过 `sys_yield`）
4. **任务切换**：从一个任务切换到另一个任务
5. **调度事件**：表示任务执行过程中的各种事件

这些概念是分时多任务系统的基础，在下一节中我们会介绍抢占式调度。
