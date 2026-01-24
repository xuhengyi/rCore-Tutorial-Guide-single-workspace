# 实现 SV39 多级页表机制（下）

## 物理页帧管理

### 可用物理页的分配与回收

在本项目的代码框架中，物理页帧管理由 `Sv39Manager` 实现。它使用内核堆分配器来分配物理页：

```rust
// ch4/src/main.rs (impls 模块)

impl Sv39Manager {
    #[inline]
    fn page_alloc<T>(count: usize) -> *mut T {
        unsafe {
            alloc_zeroed(Layout::from_size_align_unchecked(
                count << Sv39::PAGE_BITS,
                1 << Sv39::PAGE_BITS,
            ))
        }
        .cast()
    }
}
```

`page_alloc` 方法使用 `alloc_zeroed` 从内核堆中分配指定数量的页，并确保页对齐。

### 分配/回收物理页帧的接口

`Sv39Manager` 实现了 `PageManager<Sv39>` trait，提供了以下接口：

- `new_root()`：创建新的根页表
- `allocate()`：分配物理页，并设置 `OWNED` 标志位
- `root_ppn()`：获取根页表的物理页号
- `p_to_v()` 和 `v_to_p()`：物理地址和虚拟地址之间的转换

## 多级页表实现

### 页表基本数据结构与访问接口

在本项目的代码框架中，多级页表由 `page_table` crate 提供。`AddressSpace` 结构体管理地址空间：

```rust
// kernel-vm/src/space/mod.rs

pub struct AddressSpace<Meta: VmMeta, M: PageManager<Meta>> {
    pub areas: Vec<Range<VPN<Meta>>>,  // 虚拟地址块
    page_manager: M,                    // 页管理器
}
```

每个地址空间都对应一个不同的多级页表，这也就意味着不同页表的起始地址（即页表根节点的地址）是不一样的。

### 建立和拆除虚实地址映射关系

`AddressSpace` 提供了 `map_extern()` 和 `map()` 方法来建立映射：

```rust
// kernel-vm/src/space/mod.rs

impl<Meta: VmMeta, M: PageManager<Meta>> AddressSpace<Meta, M> {
    /// 向地址空间增加映射关系。
    pub fn map_extern(&mut self, range: Range<VPN<Meta>>, pbase: PPN<Meta>, flags: VmFlags<Meta>) {
        self.areas.push(range.start..range.end);
        let count = range.end.val() - range.start.val();
        let mut root = self.root();
        let mut mapper = Mapper::new(self, pbase..pbase + count, flags);
        root.walk_mut(Pos::new(range.start, 0), &mut mapper);
        // ...
    }
    
    /// 分配新的物理页，拷贝数据并建立映射。
    pub fn map(
        &mut self,
        range: Range<VPN<Meta>>,
        data: &[u8],
        offset: usize,
        mut flags: VmFlags<Meta>,
    ) {
        // 分配物理页
        let page = self.page_manager.allocate(count, &mut flags);
        // 拷贝数据
        // ...
        // 建立映射
        self.map_extern(range, self.page_manager.v_to_p(page), flags)
    }
}
```

- `map_extern()`：将虚拟地址范围映射到已有的物理页范围
- `map()`：分配新的物理页，拷贝数据，然后建立映射

### 内核中访问物理页帧的方法

在内核地址空间中，我们使用恒等映射（Identical Mapping）来访问物理页帧。也就是说，对于物理内存上的每个物理页帧，我们都在多级页表中用一个与其物理页号相等的虚拟页号映射到它。

在 `Sv39Manager` 中，`p_to_v()` 方法实现了这种转换：

```rust
// ch4/src/main.rs (impls 模块)

impl PageManager<Sv39> for Sv39Manager {
    fn p_to_v<T>(&self, ppn: PPN<Sv39>) -> NonNull<T> {
        unsafe { NonNull::new_unchecked(VPN::<Sv39>::new(ppn.val()).base().as_mut_ptr()) }
    }
    
    fn v_to_p<T>(&self, ptr: NonNull<T>) -> PPN<Sv39> {
        PPN::new(VAddr::<Sv39>::new(ptr.as_ptr() as _).floor().val())
    }
}
```

这样，当我们想针对物理页号构造一个能映射到它的虚拟页号的时候，只需使用一个和该物理页号相等的虚拟页号即可。

## 地址翻译

`AddressSpace` 提供了 `translate()` 方法来将虚拟地址翻译为当前地址空间中的指针：

```rust
// kernel-vm/src/space/mod.rs

pub fn translate<T>(&self, addr: VAddr<Meta>, flags: VmFlags<Meta>) -> Option<NonNull<T>> {
    let mut visitor = Visitor::new(self);
    self.root().walk(Pos::new(addr.floor(), 0), &mut visitor);
    visitor
        .ans()
        .filter(|pte| pte.flags().contains(flags))
        .map(|pte| unsafe {
            NonNull::new_unchecked(
                self.page_manager
                    .p_to_v::<u8>(pte.ppn())
                    .as_ptr()
                    .add(addr.offset())
                    .cast(),
            )
        })
}
```

这个方法会：
1. 遍历页表找到虚拟地址对应的页表项
2. 检查页表项的标志位是否满足要求
3. 将物理页号转换为虚拟地址（在当前地址空间中）
4. 加上页内偏移，返回指向数据的指针

## 关键概念总结

1. **物理页帧管理**：通过 `PageManager` trait 管理物理页的分配和回收
2. **多级页表**：SV39 使用三级页表进行地址转换
3. **恒等映射**：内核地址空间使用恒等映射来访问物理内存
4. **地址翻译**：通过遍历页表将虚拟地址转换为物理地址
5. **映射建立**：通过 `map()` 和 `map_extern()` 方法建立虚拟地址到物理地址的映射

这些机制是虚拟内存管理的基础，在下一节中我们会介绍如何为内核和应用程序创建地址空间。
