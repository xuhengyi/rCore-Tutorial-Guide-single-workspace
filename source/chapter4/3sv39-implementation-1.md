# 实现 SV39 多级页表机制（上）

> **背景知识**：
> - [地址空间](https://rcore-os.github.io/rCore-Tutorial-Book-v3/chapter4/2address-space.html)
> - [SV39 多级页表原理](https://rcore-os.github.io/rCore-Tutorial-Book-v3/chapter4/3sv39-implementation-1.html#id6)

我们将在内核实现 RV64 架构 SV39 分页机制。由于内容过多，分成两个小节。

## 虚拟地址和物理地址

### 内存控制相关的 CSR 寄存器

默认情况下 MMU 未被使能，此时无论 CPU 处于哪个特权级，访存的地址都将直接被视作物理地址。可以通过修改 S 特权级的 `satp` CSR 来启用分页模式，此后 S 和 U 特权级的访存地址会被视为虚拟地址，经过 MMU 的地址转换获得对应物理地址，再通过它来访问物理内存。

`satp` CSR 的字段分布：
- `MODE`：分页模式，设置为 8 时启用 SV39 分页机制
- `ASID`：地址空间标识符（本章暂不使用）
- `PPN`：页表根节点的物理页号

当 `MODE` 设置为 0 的时候，所有访存都被视为物理地址；而设置为 8 时，SV39 分页机制被启用，所有 S/U 特权级的访存被视为一个 39 位的虚拟地址，MMU 会将其转换成 56 位的物理地址；如果转换失败，则会触发异常。

### 地址格式与组成

我们采用分页管理，单个页面的大小设置为 4 KiB，每个虚拟页面和物理页帧都按 4 KB 对齐。4 KiB 需要用 12 位字节地址来表示，因此虚拟地址和物理地址都被分成两部分：

- 它们的低 12 位被称为 **页内偏移** (Page Offset)
- 虚拟地址的高 27 位，即 `[38:12]` 为它的虚拟页号 VPN
- 物理地址的高 44 位，即 `[55:12]` 为它的物理页号 PPN

页号可以用来定位一个虚拟/物理地址属于哪一个虚拟页面/物理页帧。

地址转换是以页为单位进行的，转换前后地址页内偏移部分不变。MMU 只是从虚拟地址中取出 27 位虚拟页号，在页表中查到其对应的物理页号，如果找到，就将得到的 44 位的物理页号与 12 位页内偏移拼接到一起，形成 56 位物理地址。

> **注意**：**RV64 架构中虚拟地址为何只有 39 位？**
> 
> 虚拟地址长度确实应该和位宽一致为 64 位，但是在启用 SV39 分页模式下，只有低 39 位是真正有意义的。SV39 分页模式规定 64 位虚拟地址的 `[63:39]` 这 25 位必须和第 38 位相同，否则 MMU 会直接认定它是一个不合法的虚拟地址。
> 
> 也就是说，所有 `2^64` 个虚拟地址中，只有最低的 256 GiB（当第 38 位为 0 时）以及最高的 256 GiB（当第 38 位为 1 时）是可能通过 MMU 检查的。

## 地址相关的数据结构抽象与类型定义

在本项目的代码框架中，地址和页号的概念由 `page_table` crate 提供抽象。这些类型包括：

- `VAddr<Sv39>`：虚拟地址
- `PAddr`：物理地址（在本项目中通常直接使用 `usize`）
- `VPN<Sv39>`：虚拟页号
- `PPN<Sv39>`：物理页号

这些类型提供了类型安全的地址和页号操作，避免了直接使用 `usize` 可能带来的错误。

## 页表项的数据结构抽象与类型定义

SV39 分页模式下的页表项（PTE）包含：
- `[53:10]` 这 44 位是物理页号
- 最低的 8 位 `[7:0]` 则是标志位

标志位的含义：
- **V (Valid)**：仅当 V 位为 1 时，页表项才是合法的
- **R/W/X**：分别控制索引到这个页表项的对应虚拟页面是否允许读/写/取指
- **U**：控制索引到这个页表项的对应虚拟页面是否在 CPU 处于 U 特权级的情况下是否被允许访问
- **G**：全局标志（本章不使用）
- **A (Accessed)**：记录自从页表项上的这一位被清零之后，页表项的对应虚拟页面是否被访问过
- **D (Dirty)**：记录自从页表项上的这一位被清零之后，页表项的对应虚拟页表是否被修改过

在本项目的代码框架中，页表项由 `page_table` crate 提供，通过 `VmFlags` 来设置标志位。

## kernel-vm 库的使用

在本项目的代码框架中，虚拟内存管理由 `kernel-vm` 库提供。主要的数据结构是 `AddressSpace`：

```rust
// kernel-vm/src/space/mod.rs

pub struct AddressSpace<Meta: VmMeta, M: PageManager<Meta>> {
    pub areas: Vec<Range<VPN<Meta>>>,  // 虚拟地址块
    page_manager: M,                    // 页管理器
}
```

`AddressSpace` 提供了以下主要功能：

1. **创建地址空间**：`AddressSpace::new()` 创建一个新的地址空间
2. **建立映射**：`map_extern()` 和 `map()` 方法建立虚拟地址到物理地址的映射
3. **地址翻译**：`translate()` 方法将虚拟地址翻译为当前地址空间中的指针
4. **获取根页表**：`root_ppn()` 和 `root()` 方法获取页表根节点的物理页号

`PageManager` trait 定义了物理页管理器的接口，需要实现：
- `new_root()`：创建新的根页表
- `allocate()`：分配物理页
- `deallocate()`：释放物理页
- `p_to_v()` 和 `v_to_p()`：物理地址和虚拟地址之间的转换

## Sv39Manager 实现

在 `ch4/src/main.rs` 的 `impls` 模块中，我们实现了 `Sv39Manager`：

```rust
// ch4/src/main.rs (impls 模块)

#[repr(transparent)]
pub struct Sv39Manager(NonNull<Pte<Sv39>>);

impl PageManager<Sv39> for Sv39Manager {
    fn new_root() -> Self {
        Self(NonNull::new(Self::page_alloc(1)).unwrap())
    }
    
    fn root_ppn(&self) -> PPN<Sv39> {
        PPN::new(self.0.as_ptr() as usize >> Sv39::PAGE_BITS)
    }
    
    fn allocate(&mut self, len: usize, flags: &mut VmFlags<Sv39>) -> NonNull<u8> {
        *flags |= Self::OWNED;
        NonNull::new(Self::page_alloc(len)).unwrap()
    }
    
    // ... 其他方法
}
```

`Sv39Manager` 使用内核堆分配器来分配物理页，并通过 `OWNED` 标志位来标记哪些页是由该地址空间管理的。

## 关键概念总结

1. **SV39 分页模式**：RISC-V 64 位架构的 39 位虚拟地址分页模式
2. **虚拟页号 (VPN)**：27 位，用于在页表中查找对应的物理页号
3. **物理页号 (PPN)**：44 位，指向实际的物理页帧
4. **页表项 (PTE)**：包含物理页号和标志位
5. **地址空间**：由页表和虚拟地址块组成的抽象

这些概念是虚拟内存管理的基础，在下一节中我们会介绍如何建立和管理页表。
