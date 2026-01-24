# 实现批处理操作系统

## 将应用程序链接到内核

在本章中，我们要把应用程序的二进制镜像文件作为数据段链接到内核里，内核需要知道应用程序的数量和它们的位置。

在本项目的代码框架中，应用程序的链接是通过 `linker` 库来完成的。在 `ch2/src/main.rs` 中能够找到这样一行：

```rust
// 用户程序内联进来。
core::arch::global_asm!(include_str!(env!("APP_ASM")));
```

这里我们引入了一段汇编代码，它是在构建时自动生成的，包含了所有应用程序的二进制数据。`APP_ASM` 环境变量指向这个生成的汇编文件。

`linker` 库提供了 `AppMeta` 结构来管理链接进来的应用程序：

```rust
// linker/src/app.rs

#[repr(C)]
pub struct AppMeta {
    base: u64,
    step: u64,
    count: u64,
    first: u64,
}
```

这个结构体记录了：
- `base`：应用程序加载的基地址
- `step`：每个应用程序之间的步长
- `count`：应用程序的数量
- `first`：第一个应用程序的起始位置

## 找到并加载应用程序二进制码

我们在 `ch2/src/main.rs` 中使用 `linker::AppMeta::locate()` 来获取应用程序元数据，然后通过迭代器遍历所有应用程序：

```rust
// ch2/src/main.rs

for (i, app) in linker::AppMeta::locate().iter().enumerate() {
    let app_base = app.as_ptr() as usize;
    log::info!("load app{i} to {app_base:#x}");
    // ...
}
```

`AppMeta::iter()` 返回一个迭代器，每次迭代会：
1. 如果 `base != 0`，将应用程序从链接位置复制到 `base + i * step` 地址，并清零剩余空间
2. 如果 `base == 0`，直接返回链接位置的应用程序数据（原地执行）

这样，批处理系统就可以依次加载和运行每个应用程序了。

## 批处理系统的核心逻辑

批处理系统的核心逻辑在 `rust_main` 函数中：

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
    // 批处理
    for (i, app) in linker::AppMeta::locate().iter().enumerate() {
        let app_base = app.as_ptr() as usize;
        log::info!("load app{i} to {app_base:#x}");
        // 初始化上下文
        let mut ctx = LocalContext::user(app_base);
        // 设置用户栈
        let mut user_stack: core::mem::MaybeUninit<[usize; 256]> = core::mem::MaybeUninit::uninit();
        let user_stack_ptr = user_stack.as_mut_ptr() as *mut usize;
        *ctx.sp_mut() = unsafe { user_stack_ptr.add(256) } as usize;
        loop {
            unsafe { ctx.execute() };
            // 处理 Trap...
        }
    }
    sbi::shutdown(false)
}
```

让我们逐步分析这段代码：

1. **初始化阶段**：
   - 清零 `.bss` 段
   - 初始化控制台和日志系统
   - 初始化系统调用处理

2. **批处理循环**：
   - 遍历所有应用程序
   - 为每个应用程序创建用户上下文 `LocalContext::user(app_base)`
   - 设置用户栈（256 个 `usize`，即 2048 字节）
   - 进入执行循环

3. **执行循环**：
   - 调用 `ctx.execute()` 执行用户程序
   - 当发生 Trap 时，`execute()` 会返回
   - 根据 Trap 类型进行处理（系统调用、异常等）
   - 如果是系统调用，处理完后继续执行；如果是异常，退出当前应用程序

## 用户栈的设置

在运行应用程序之前，我们需要为用户程序准备一个栈空间。在本章的实现中：

```rust
let mut user_stack: core::mem::MaybeUninit<[usize; 256]> = core::mem::MaybeUninit::uninit();
let user_stack_ptr = user_stack.as_mut_ptr() as *mut usize;
*ctx.sp_mut() = unsafe { user_stack_ptr.add(256) } as usize;
```

这里我们：
1. 创建一个未初始化的栈数组（256 个 `usize`，即 2048 字节）
2. 使用 `MaybeUninit` 避免在 release 模式下进行不必要的零初始化
3. 将栈指针设置为栈数组的末尾（因为 RISC-V 的栈是向下增长的）

## 指令缓存的清理

在加载新应用程序之前，我们需要清理指令缓存（i-cache）：

```rust
// 清除指令缓存
unsafe { core::arch::asm!("fence.i") };
```

这是因为 CPU 会认为程序的代码段不会发生变化，因此 i-cache 是一种只读缓存。但在这里，我们会修改会被 CPU 取指的内存区域（加载新的应用程序），使得 i-cache 中含有与内存不一致的内容，必须使用 `fence.i` 指令手动清空 i-cache，让里面所有的内容全部失效，才能够保证程序执行正确性。

> **警告**：**模拟器与真机的不同之处**
> 
> 在 Qemu 模拟器上，即使不加刷新 i-cache 的指令，大概率也能正常运行，但在物理计算机上不是这样。

## 批处理系统的特点

本章实现的批处理系统被称为"邓氏鱼"（Dunkleosteus），这是一种在泥盆纪非常成功的生物，但在环境变化后迅速灭绝。这个比喻说明了：

1. **简单有效**：批处理系统在资源受限的环境下非常有效
2. **短视设计**：这种设计可能随着环境变化而不再适用
3. **核心结构**：虽然整体设计可能过时，但某些核心结构（如系统调用和上下文切换）会保留下来

本章的代码非常简洁，主要关注两个核心概念：
- **系统调用**：用户程序和内核之间的接口
- **上下文切换**：通过 `LocalContext` 实现用户态和内核态的切换

这些核心概念在后续章节中会继续使用和发展。
