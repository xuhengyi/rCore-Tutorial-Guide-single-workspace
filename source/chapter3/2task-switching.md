# 任务切换

## 任务切换的概念

本节我们将见识操作系统的核心机制——**任务切换**，即应用在运行中主动或被动地交出 CPU 的使用权，内核可以选择另一个程序继续执行。内核需要保证用户程序两次运行期间，任务上下文（如寄存器、栈等）保持一致。

## 任务切换的设计与实现

任务切换与上一章提及的 Trap 控制流切换相比，有如下异同：

- 与 Trap 切换不同，它不涉及特权级切换，部分由编译器完成；
- 与 Trap 切换相同，它对应用是透明的。

事实上，任务切换是来自两个不同应用在内核中的 Trap 控制流之间的切换。当一个应用 Trap 到 S 态 OS 内核中进行进一步处理时，其 Trap 控制流可以调用一个特殊的函数来切换到另一个应用的 Trap 控制流。

在本项目的代码框架中，任务切换通过 `LocalContext::execute()` 和 `LocalContext` 的保存/恢复来实现。

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

## 任务执行

任务的执行通过 `TaskControlBlock::execute()` 方法：

```rust
// ch3/src/task.rs

pub unsafe fn execute(&mut self) {
    self.ctx.execute();
}
```

这个方法会调用 `LocalContext::execute()`，它会：
1. 设置 `stvec` 指向 Trap 处理入口
2. 保存当前内核上下文
3. 恢复用户上下文
4. 通过 `sret` 指令切换到用户态执行

当发生 Trap 时，硬件会：
1. 保存当前状态到 CSR（`sstatus`、`sepc` 等）
2. 跳转到 `stvec` 指向的 Trap 处理入口
3. 在 Trap 处理入口，会交换 `sp` 和 `sscratch`，从而切换到内核栈

## 任务切换流程

在 `ch3/src/main.rs` 中，任务切换的流程如下：

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
            
            // 处理 Trap...
        }
    }
    i = (i + 1) % index_mod;
}
```

这个循环会：
1. 遍历所有任务控制块
2. 如果任务未完成，执行该任务
3. 当发生 Trap 时，根据 Trap 类型进行处理
4. 如果是系统调用或时钟中断，可能会切换到下一个任务

## 任务切换的时机

任务切换可以在以下时机发生：

1. **协作式切换**：任务主动调用 `sys_yield` 系统调用
2. **抢占式切换**：时钟中断触发，强制切换任务
3. **任务退出**：任务调用 `sys_exit` 系统调用或发生异常

## 上下文保存和恢复

任务切换的关键是保存和恢复上下文。在本项目的代码框架中，这由 `LocalContext` 自动处理：

- **保存**：当发生 Trap 时，`LocalContext::execute()` 返回，此时上下文已经保存在 `LocalContext` 结构体中
- **恢复**：下次调用 `LocalContext::execute()` 时，会从保存的上下文恢复执行

## 与第二章的对比

| 特性 | 第二章 | 第三章 |
|------|--------|--------|
| 任务管理 | 无，每次只运行一个应用 | 使用 `TaskControlBlock` 管理多个任务 |
| 切换方式 | 重新加载应用程序 | 保存/恢复上下文，快速切换 |
| 并发性 | 串行执行 | 多任务并发执行 |
| 切换开销 | 需要重新加载程序 | 只需切换上下文，开销很小 |

通过任务切换机制，我们可以实现真正的多任务并发执行，大大提高系统的效率和响应性。
