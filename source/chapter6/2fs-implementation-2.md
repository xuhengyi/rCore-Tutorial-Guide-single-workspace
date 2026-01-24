# 简易文件系统 easy-fs（下）

## 磁盘块管理器（EasyFileSystem）

`EasyFileSystem` 负责整个磁盘布局与块分配：

- 持有块设备引用、inode 位图、数据块位图，以及各区域的起始块号。
- **create**：在块设备上创建并初始化一个新的 easy-fs（写超级块、根目录 inode 等）。
- **open**：从已存在的镜像打开 easy-fs（读超级块，初始化位图与区域信息）。
- **alloc_inode** / **alloc_data**：在位图中分配 inode 或数据块，返回块号。
- **dealloc_data**：回收数据块（本节简化实现中可能未实现 `dealloc_inode`，即不支持删除文件）。

通过 `get_disk_inode_pos`、`get_data_block_id` 等，可根据 inode 编号或数据块编号计算出在块设备上的实际块号，供下层块缓存访问。

## 索引节点层（Inode）

`Inode` 是 easy-fs 对上暴露的文件/目录抽象，封装了 `DiskInode` 在磁盘上的位置以及 `EasyFileSystem` 的引用。

### 根目录

`EasyFileSystem::root_inode()` 返回根目录的 `Inode`。根目录对应 inode 编号 0，所有文件均置于根目录之下。

### 文件查找、创建、列举

- **find(name)**：在根目录的目录项中按文件名查找，返回对应文件的 `Inode`。
- **create(name)**：在根目录下创建新文件，分配 inode 并添加目录项，返回新 `Inode`。
- **readdir**：列举根目录下所有文件名（即遍历根目录的目录项）。

### 文件读写与清空

- **read_at(offset, buf)**：从文件 `offset` 处读取若干字节到 `buf`，返回实际读取长度。
- **write_at(offset, buf)**：向文件 `offset` 处写入 `buf` 内容；若超出当前大小，则先扩展文件（分配新数据块、更新 DiskInode）。
- **clear**：清空文件内容，释放数据块并重置大小。

实现时通过 `EasyFileSystem` 分配数据块，通过 `DiskInode` 的多级索引解析块号，再经块缓存读写块内容。

## 将应用打包为 easy-fs 镜像

在开发环境中，通过 `easy-fs-fuse` 或项目内的 **xtask** 等工具：

1. 在宿主机上创建一个“块设备”（如一个大文件）。
2. 在其上 `EasyFileSystem::create` 创建 easy-fs。
3. 获取根目录 `Inode`，将各应用 ELF 以文件形式 `create` 并 `write_at` 写入。
4. 将结果保存为 `fs.img` 等镜像文件。

QEMU 启动时将该镜像作为 virtio-blk 的后端，内核中的 `BlockDevice` 实现即可读写该镜像，从而挂载 easy-fs。

## 小结

- **EasyFileSystem** 管理磁盘布局、位图与块分配，提供 `create` / `open`、`root_inode`、`alloc_inode` / `alloc_data` 等。
- **Inode** 提供基于根目录的 `find` / `create` / `readdir`，以及 `read_at` / `write_at` / `clear`，实现文件与目录的抽象。
- 应用通过 xtask/easy-fs-fuse 等工具打包进 `fs.img`，内核从块设备挂载 easy-fs 并借由其接口加载、执行用户程序。
