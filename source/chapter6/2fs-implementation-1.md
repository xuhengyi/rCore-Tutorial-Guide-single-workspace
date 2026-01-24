# 简易文件系统 easy-fs（上）

## 松耦合模块化设计

easy-fs 从内核中分离成独立 crate，便于开发和测试：

- **easy-fs**：简易文件系统本体，实现磁盘布局、块缓存、Inode 等。
- **easy-fs-fuse / xtask**：在开发机上打包应用、制作 easy-fs 镜像，或对 easy-fs 做测试。

easy-fs 通过 **BlockDevice** 抽象与具体块设备对接，不依赖内核进程等概念，因此可单独在用户态验证。

## easy-fs 的层次结构

自下而上分为五层：

1. **块设备接口层**：`BlockDevice` trait，以块为单位读写。
2. **块缓存层**：在内存中缓存磁盘块，减少实际读写次数。
3. **磁盘数据结构层**：超级块、位图、DiskInode、目录项等。
4. **磁盘块管理层**：`EasyFileSystem`，管理布局、分配 inode/数据块。
5. **索引节点层**：`Inode`，提供创建/查找/读写等文件操作。

本节介绍前三层，下一节介绍后两层。

## 块设备接口层

在 `easy-fs` 中，块设备抽象为：

```rust
// easy-fs 中的 BlockDevice trait（示意）

pub trait BlockDevice: Send + Sync {
    fn read_block(&self, block_id: usize, buf: &mut [u8]);
    fn write_block(&self, block_id: usize, buf: &[u8]);
}
```

- `read_block`：将 `block_id` 对应的块读入 `buf`。
- `write_block`：将 `buf` 写入 `block_id` 对应的块。

内核（或其它环境）实现该 trait，即可把具体块设备接入 easy-fs。例如 ch6 中通过 `VirtIOBlock` 实现 `BlockDevice`，访问 QEMU 的 virtio-blk 设备。

## 块缓存层

为减少频繁访问块设备，在内存中维护 **块缓存**：

- 每个缓存项对应一个块，含块号、缓冲区、脏标记等。
- 读块时先查缓存，未命中再从设备读取并加入缓存。
- 写操作在缓存中进行，根据脏标记在合适的时机写回设备。

块缓存管理器负责缓存替换（如类 FIFO 策略）、分配/释放缓存等。easy-fs 通过 `get_block_cache` 一类接口使用块缓存，上层只按块号读写，无需关心是否命中设备。

## 磁盘布局与数据结构

### 磁盘布局概览

easy-fs 将磁盘按块号顺序划分为若干连续区域：

```
+------------+--------------+--------+-------------+------+
| SuperBlock | Inode Bitmap | Inode  | Data Bitmap | Data |
+------------+--------------+--------+-------------+------+
```

- **SuperBlock**：通常占一个块，存魔数、各区域块数等，用于校验和定位。
- **Inode Bitmap**：记录哪些 inode 已分配。
- **Inode 区**：存放 `DiskInode`。
- **Data Bitmap**：记录哪些数据块已分配。
- **Data 区**：文件/目录数据块。

### 超级块

超级块中保存总块数、各区域块数等。通过魔数可校验是否为合法的 easy-fs 镜像。`EasyFileSystem::create` 会初始化超级块；`EasyFileSystem::open` 则读取超级块并解析布局。

### 位图

位图由若干块组成，每 bit 表示一个 inode 或一个数据块的分配状态。提供 `alloc` / `dealloc` 等接口，用于分配或回收 inode/数据块。

### DiskInode 与目录项

- **DiskInode**：磁盘上的索引节点，含文件大小、类型（文件/目录）、直接/间接块号等。通过多级索引支持较大文件。
- **DirEntry**：目录项，存文件名和 inode 编号。目录文件内容即目录项序列。

## 小结

- 块设备层提供 `read_block` / `write_block`，与具体驱动对接。
- 块缓存层在内存中缓存块，减少 I/O，并负责写回。
- 磁盘布局包含超级块、inode 位图、inode 区、数据位图、数据区；位图管理分配，DiskInode 与目录项描述文件与目录结构。

下一节将说明 `EasyFileSystem` 与 `Inode` 如何利用这些结构实现文件系统的创建、打开、查找、读写等操作。
