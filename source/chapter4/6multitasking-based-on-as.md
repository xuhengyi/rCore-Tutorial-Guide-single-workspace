# 基于地址空间的分时多任务

## 概述

本节我们介绍如何基于地址空间抽象来实现分时多任务系统。

## 建立并开启基于分页模式的虚拟地址空间

当 SBI 实现初始化完成后，CPU 将跳转到内核入口点并在 S 特权级上执行，此时还并没有开启分页模式，内核的每一次访存仍被视为一个物理地址直接访问物理内存。而在开启分页模式之后，内核的代码在访存的时候只能看到内核地址空间，此时每次访存将被视为一个虚拟地址且需要通过 MMU 基于内核地址空间的多级页表的地址转换。

### 创建内核地址空间

在 `ch4/src/main.rs` 的 `rust_main` 函数中，我们创建内核地址空间并启用分页：

```rust
// ch4/src/main.rs

extern "C" fn rust_main() -> ! {
    let layout = linker::KernelLayout::locate();
    // bss 段清零
    unsafe { layout.zero_bss() };
    // 初始化 `console`
    rcore_console::init_console(&Console);
    rcore_console::set_log_level(option_env!("LOG"));
    rcore_console::test_log();
    // 初始化内核堆
    kernel_alloc::init(layout.start() as _);
    unsafe {
        kernel_alloc::transfer(core::slice::from_raw_parts_mut(
            layout.end() as _,
            MEMORY - layout.len(),
        ))
    };
    // 建立异界传送门
    let portal_size = MultislotPortal::calculate_size(1);
    let portal_layout = Layout::from_size_align(portal_size, 1 << Sv39::PAGE_BITS).unwrap();
    let portal_ptr = unsafe { alloc(portal_layout) };
    // 建立内核地址空间
    let mut ks = kernel_space(layout, MEMORY, portal_ptr as _);
    // ...
}
```

`kernel_space()` 函数会：
1. 创建新的地址空间
2. 映射内核的各个段（代码段、数据段等）
3. 映射堆空间
4. 映射传送门
5. 设置 `satp` CSR 启用分页模式

### 启用分页模式

在 `kernel_space()` 函数的最后，我们设置 `satp` CSR：

```rust
// ch4/src/main.rs

unsafe { satp::set(satp::Mode::Sv39, 0, space.root_ppn().val()) };
```

这行代码会：
1. 将 `satp` 的 `MODE` 字段设置为 `Sv39`（值为 8）
2. 将 `PPN` 字段设置为页表根节点的物理页号
3. 从这一刻开始，SV39 分页模式就被启用了

在设置 `satp` 之后，我们还需要使用 `sfence.vma` 指令清空 TLB（快表），以确保地址转换能够及时与 `satp` 的修改同步。

## 跨地址空间的上下文切换

由于内核和应用程序拥有独立的地址空间，在 Trap 和任务切换时需要进行地址空间切换。在本项目的代码框架中，这通过 `ForeignContext` 来实现：

```rust
// ch4/src/process.rs

pub struct Process {
    pub context: ForeignContext,
    pub address_space: AddressSpace<Sv39, Sv39Manager>,
}
```

`ForeignContext` 包含：
- `context`：`LocalContext`，保存了通用寄存器和程序计数器
- `satp`：地址空间的 token（用于设置 `satp` CSR）

### 执行进程

在 `ch4/src/main.rs` 的 `schedule()` 函数中，我们使用 `ForeignContext::execute()` 来执行进程：

```rust
// ch4/src/main.rs

extern "C" fn schedule() -> ! {
    // 初始化异界传送门
    let portal = unsafe { MultislotPortal::init_transit(PROTAL_TRANSIT.base().val(), 1) };
    // 初始化 syscall
    // ...
    while !unsafe { PROCESSES.get_mut().is_empty() } {
        let ctx = unsafe { &mut PROCESSES.get_mut()[0].context };
        unsafe { ctx.execute(portal, ()) };
        // 处理 Trap...
    }
}
```

`ForeignContext::execute()` 会：
1. 切换到进程的地址空间（设置 `satp`）
2. 恢复进程的上下文
3. 通过 `sret` 指令返回到用户态执行

当发生 Trap 时，硬件会：
1. 保存当前状态到 CSR
2. 跳转到 `stvec` 指向的 Trap 处理入口
3. 在 Trap 处理入口，会切换到内核地址空间

## 传送门（Portal）机制

传送门是用于跨地址空间切换的特殊机制。在本章中，我们使用 `MultislotPortal` 来实现：

```rust
// ch4/src/main.rs

// 建立异界传送门
let portal_size = MultislotPortal::calculate_size(1);
let portal_layout = Layout::from_size_align(portal_size, 1 << Sv39::PAGE_BITS).unwrap();
let portal_ptr = unsafe { alloc(portal_layout) };
```

传送门被映射到内核和应用地址空间的最高虚拟页面，这样在执行地址空间切换的代码时，无论当前在哪个地址空间，都能访问到这段代码。

## 加载和执行应用程序

### 进程创建

在 `ch4/src/process.rs` 中，我们通过解析 ELF 文件来创建进程：

```rust
// ch4/src/process.rs

impl Process {
    pub fn new(elf: ElfFile) -> Option<Self> {
        // 解析 ELF 入口点
        let entry = // ...
        
        // 创建地址空间并映射 ELF 段
        let mut address_space = AddressSpace::new();
        for program in elf.program_iter() {
            // 映射程序段
        }
        
        // 映射用户栈
        // ...
        
        // 创建上下文
        let mut context = LocalContext::user(entry);
        let satp = (8 << 60) | address_space.root_ppn().val();
        *context.sp_mut() = 1 << 38;
        
        Some(Self {
            context: ForeignContext { context, satp },
            address_space,
        })
    }
}
```

### 进程调度

在 `schedule()` 函数中，我们实现了简单的轮询调度：

```rust
// ch4/src/main.rs

extern "C" fn schedule() -> ! {
    // ...
    while !unsafe { PROCESSES.get_mut().is_empty() } {
        let ctx = unsafe { &mut PROCESSES.get_mut()[0].context };
        unsafe { ctx.execute(portal, ()) };
        
        match scause::read().cause() {
            scause::Trap::Exception(scause::Exception::UserEnvCall) => {
                // 处理系统调用
                // ...
            }
            e => {
                // 处理异常
                unsafe { PROCESSES.get_mut().remove(0) };
            }
        }
    }
    sbi::shutdown(false)
}
```

## 系统调用处理

由于地址空间隔离，系统调用处理需要特别注意跨地址空间访问。例如，`sys_write` 需要手动查页表来访问应用程序的缓冲区：

```rust
// ch4/src/main.rs (impls 模块)

impl IO for SyscallContext {
    fn write(&self, caller: Caller, fd: usize, buf: usize, count: usize) -> isize {
        match fd {
            STDOUT | STDDEBUG => {
                const READABLE: VmFlags<Sv39> = VmFlags::build_from_str("RV");
                if let Some(ptr) = unsafe { PROCESSES.get_mut() }
                    .get_mut(caller.entity)
                    .unwrap()
                    .address_space
                    .translate(VAddr::new(buf), READABLE)
                {
                    print!("{}", unsafe {
                        core::str::from_utf8_unchecked(core::slice::from_raw_parts(
                            ptr.as_ptr(),
                            count,
                        ))
                    });
                    count as _
                } else {
                    -1
                }
            }
            // ...
        }
    }
}
```

## 关键概念总结

1. **地址空间切换**：在 Trap 和任务切换时需要切换地址空间
2. **ForeignContext**：用于跨地址空间的上下文切换
3. **传送门机制**：用于在地址空间切换时执行代码的特殊机制
4. **ELF 解析**：通过解析 ELF 文件来创建应用地址空间
5. **跨地址空间访问**：内核需要手动查页表来访问应用程序的数据

通过这些机制，我们实现了基于地址空间的分时多任务系统，为每个应用程序提供了安全隔离的执行环境。
