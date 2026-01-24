# 多道程序放置与加载

## 多道程序放置

在第二章中，内核让所有应用都共享同一个固定的起始地址。正因如此，内存中同时最多只能驻留一个应用。

要一次加载运行多个程序，就要求每个用户程序被内核加载到内存中的起始地址都不同。在本项目的代码框架中，应用程序的链接是通过 `linker` 库来完成的，它会自动为每个应用程序分配不同的起始地址。

每个应用程序会被加载到不同的内存地址，地址间隔为 `APP_SIZE_LIMIT`（通常为 `0x20000`，即 128KB）。这样，多个应用程序可以同时驻留在内存中，而不会相互覆盖。

> **注意**：qemu 预留的内存空间是有限的，如果加载的程序过多，程序地址超出内存空间，可能出现 `core dumped`。

## 多道程序加载

在本项目的代码框架中，应用程序的加载在 `ch3/src/main.rs` 的 `rust_main` 函数中完成：

```rust
extern "C" fn rust_main() -> ! {
    // bss 段清零
    unsafe { linker::KernelLayout::locate().zero_bss() };
    // 初始化 `console`
    rcore_console::init_console(&Console);
    rcore_console::set_log_level(option_env!("LOG"));
    rcore_console::test_log();
    // 初始化 syscall
    syscall::init_io(&SyscallContext);
    syscall::init_process(&SyscallContext);
    syscall::init_scheduling(&SyscallContext);
    syscall::init_clock(&SyscallContext);
    syscall::init_trace(&SyscallContext);
    // 任务控制块
    let mut tcbs = [TaskControlBlock::ZERO; APP_CAPACITY];
    let mut index_mod = 0;
    // 初始化
    for (i, app) in linker::AppMeta::locate().iter().enumerate() {
        let entry = app.as_ptr() as usize;
        log::info!("load app{i} to {entry:#x}");
        tcbs[i].init(entry);
        index_mod += 1;
    }
    // ...
}
```

让我们逐步分析这段代码：

1. **初始化阶段**：
   - 清零 `.bss` 段
   - 初始化控制台和日志系统
   - 初始化系统调用处理（包括新增的调度、时钟和跟踪相关系统调用）

2. **任务控制块数组**：
   - 创建一个 `TaskControlBlock` 数组，大小为 `APP_CAPACITY`（32）
   - 使用 `TaskControlBlock::ZERO` 进行初始化

3. **加载应用程序**：
   - 遍历所有链接进来的应用程序
   - 对每个应用程序调用 `tcb.init(entry)` 进行初始化
   - `entry` 是应用程序在内存中的起始地址，由 `linker` 库自动分配

## TaskControlBlock 初始化

`TaskControlBlock::init()` 方法的实现：

```rust
// ch3/src/task.rs

pub fn init(&mut self, entry: usize) {
    self.stack.fill(0);
    self.finish = false;
    self.ctx = LocalContext::user(entry);
    *self.ctx.sp_mut() = self.stack.as_ptr() as usize + core::mem::size_of_val(&self.stack);
}
```

这个方法会：
1. 清零用户栈
2. 设置 `finish` 标志为 `false`
3. 创建用户上下文 `LocalContext::user(entry)`，其中 `sepc` 设置为应用程序入口地址
4. 设置用户栈指针为栈数组的末尾（因为 RISC-V 的栈是向下增长的）

## 应用程序地址分配

在本项目的代码框架中，应用程序的地址分配由 `linker` 库自动处理。`AppMeta::iter()` 方法会：

1. 如果 `base != 0`，将应用程序从链接位置复制到 `base + i * step` 地址
2. 如果 `base == 0`，直接返回链接位置的应用程序数据（原地执行）

这样，每个应用程序都会被加载到不同的内存地址，可以同时驻留在内存中。

## 与第二章的对比

| 特性 | 第二章 | 第三章 |
|------|--------|--------|
| 应用程序加载 | 每次只加载一个，运行完再加载下一个 | 启动时一次性加载所有应用程序 |
| 内存使用 | 所有应用共享同一个起始地址 | 每个应用有独立的起始地址 |
| 切换开销 | 需要重新加载应用程序 | 只需切换上下文，开销更小 |
| 并发性 | 串行执行 | 可以快速切换，实现并发效果 |

通过一次性加载所有应用程序，我们可以大大减少任务切换的开销，实现真正的多道程序并发执行。
