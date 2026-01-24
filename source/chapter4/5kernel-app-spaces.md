# 内核与应用的地址空间

## 概述

本节我们就在内核中通过基于页表的各种数据结构实现地址空间的抽象。

## 实现地址空间抽象

在本项目的代码框架中，地址空间由 `AddressSpace` 结构体表示：

```rust
// kernel-vm/src/space/mod.rs

pub struct AddressSpace<Meta: VmMeta, M: PageManager<Meta>> {
    pub areas: Vec<Range<VPN<Meta>>>,  // 虚拟地址块
    page_manager: M,                    // 页管理器
}
```

`AddressSpace` 包含：
- `areas`：虚拟地址块的列表，记录哪些虚拟地址范围被映射
- `page_manager`：页管理器，负责分配和回收物理页

## 内核地址空间

在 `ch4/src/main.rs` 的 `rust_main` 函数中，我们创建内核地址空间：

```rust
// ch4/src/main.rs

fn kernel_space(
    layout: linker::KernelLayout,
    memory: usize,
    portal: usize,
) -> AddressSpace<Sv39, Sv39Manager> {
    let mut space = AddressSpace::<Sv39, Sv39Manager>::new();
    // 映射内核各个段
    for region in layout.iter() {
        use linker::KernelRegionTitle::*;
        let flags = match region.title {
            Text => "X_RV",      // 可执行、可读
            Rodata => "__RV",    // 只读
            Data | Boot => "_WRV", // 可读写
        };
        let s = VAddr::<Sv39>::new(region.range.start);
        let e = VAddr::<Sv39>::new(region.range.end);
        space.map_extern(
            s.floor()..e.ceil(),
            PPN::new(s.floor().val()),
            VmFlags::build_from_str(flags),
        )
    }
    // 映射堆空间
    // ...
    // 映射传送门
    space.map_extern(
        PROTAL_TRANSIT..PROTAL_TRANSIT + 1,
        PPN::new(portal >> Sv39::PAGE_BITS),
        VmFlags::build_from_str("__G_XWRV"),
    );
    // 启用分页
    unsafe { satp::set(satp::Mode::Sv39, 0, space.root_ppn().val()) };
    space
}
```

内核地址空间的布局：
- **代码段 (.text)**：可执行、可读，使用恒等映射
- **只读数据段 (.rodata)**：只读，使用恒等映射
- **数据段 (.data/.bss)**：可读写，使用恒等映射
- **堆空间**：可读写，使用恒等映射
- **传送门 (Portal)**：位于最高虚拟页面，用于跨地址空间切换

## 应用地址空间

在 `ch4/src/process.rs` 中，我们通过解析 ELF 文件来创建应用地址空间：

```rust
// ch4/src/process.rs

impl Process {
    pub fn new(elf: ElfFile) -> Option<Self> {
        let entry = // ... 获取入口点
        
        let mut address_space = AddressSpace::new();
        // 解析 ELF 程序段
        for program in elf.program_iter() {
            if !matches!(program.get_type(), Ok(program::Type::Load)) {
                continue;
            }
            
            let off_mem = program.virtual_addr() as usize;
            let end_mem = off_mem + program.mem_size() as usize;
            
            // 根据 ELF 标志设置页表标志
            let mut flags: [u8; 5] = *b"U___V";
            if program.flags().is_execute() { flags[1] = b'X'; }
            if program.flags().is_write() { flags[2] = b'W'; }
            if program.flags().is_read() { flags[3] = b'R'; }
            
            // 建立映射并拷贝数据
            address_space.map(
                VAddr::new(off_mem).floor()..VAddr::new(end_mem).ceil(),
                &elf.input[off_file..][..len_file],
                off_mem & PAGE_MASK,
                VmFlags::from_str(unsafe { core::str::from_utf8_unchecked(&flags) }).unwrap(),
            );
        }
        
        // 映射用户栈
        let stack = // ... 分配栈空间
        address_space.map_extern(
            VPN::new((1 << 26) - 2)..VPN::new(1 << 26),
            PPN::new(stack as usize >> Sv39::PAGE_BITS),
            VmFlags::build_from_str("U_WRV"),
        );
        
        // 创建进程上下文
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

应用地址空间的布局：
- **ELF 程序段**：根据 ELF 文件中的程序头（Program Header）映射各个段
- **用户栈**：位于虚拟地址空间的较高位置
- **传送门**：与内核地址空间共享，用于 Trap 处理

## 地址空间隔离

地址空间抽象的重要意义在于 **隔离** (Isolation)。当我们执行每个应用的代码的时候，内核需要控制 MMU 使用这个应用地址空间的多级页表进行地址转换。由于每个应用地址空间在创建的时候也顺带设置好了多级页表使得只有那些存放了它的数据的物理页帧能够通过该多级页表被映射到，这样它就只能访问自己的数据而无法触及其他应用或是内核的数据。

在本章中，我们采用了内核和应用地址空间隔离的设计：
- 内核拥有一个独立的地址空间
- 每个应用程序拥有自己的地址空间
- 在 Trap 的时候需要进行地址空间切换

这种设计的优点：
- 可以应对处理器的 Meltdown 漏洞
- 内核数据不会暴露给应用程序
- 更符合现代操作系统的设计

缺点：
- Trap 时需要切换地址空间，有一定的性能开销
- 实现复杂度较高

## 跨地址空间访问

由于内核和应用程序的地址空间隔离，内核无法直接访问应用程序的数据。在系统调用处理中，我们需要手动查页表来访问应用程序的数据：

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
                    log::error!("ptr not readable");
                    -1
                }
            }
            // ...
        }
    }
}
```

`address_space.translate()` 方法会：
1. 在应用程序的地址空间中查找虚拟地址 `buf` 对应的页表项
2. 检查页表项的标志位是否满足要求（可读）
3. 将物理地址转换为内核地址空间中的虚拟地址
4. 返回指向数据的指针

## 关键概念总结

1. **地址空间**：由页表和虚拟地址块组成的抽象
2. **地址空间隔离**：内核和应用程序拥有独立的地址空间
3. **恒等映射**：内核地址空间使用恒等映射来访问物理内存
4. **ELF 解析**：通过解析 ELF 文件来创建应用地址空间
5. **跨地址空间访问**：内核需要手动查页表来访问应用程序的数据

这些机制为每个应用程序提供了安全隔离的执行环境，是操作系统安全性的重要保障。
