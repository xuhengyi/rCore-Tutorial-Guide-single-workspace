# 第六章：文件系统与 I/O

## 目录

1. [引言](0intro.md)
2. [文件与文件描述符](1file-descriptor.md)
3. [文件系统接口](1fs-interface.md)
4. [简易文件系统 easy-fs（上）](2fs-implementation-1.md)
5. [简易文件系统 easy-fs（下）](2fs-implementation-2.md)
6. [在内核中使用 easy-fs](3using-easy-fs-in-kernel.md)
7. [练习](4exercise.md) - 编程作业和问答作业

## 本章目标

- 实现简易文件系统 **easy-fs**，管理持久存储（块设备）。
- 接入 **VirtIO 块设备**，在 QEMU 上挂载 `fs.img`。
- 为进程增加 **文件描述符表**，实现 `open` / `close` / `read` / `write`。
- **移除 loader**，从文件系统加载 initproc 与 `exec` 的程序。

## 代码结构概要

- **ch6**：内核入口、MMIO、FS 初始化、进程与 fd、VirtIO 块驱动。
- **easy-fs**：块设备抽象、块缓存、磁盘布局、`EasyFileSystem`、`Inode`。
- **xtask**：制作 `fs.img`、打包应用等构建流程。

## 运行与测试

```bash
$ cd ch6
$ cargo run
```

在 shell 中运行 `ch6_file0`、`ch6_file1`（或 `filetest_simple`、`cat_filea`）等测例，验证创建、读写、关闭文件以及从文件系统加载程序。

## 概念速览

- **BlockDevice**：块设备抽象；**块缓存**：内存缓存磁盘块。
- **easy-fs 布局**：SuperBlock | Inode Bitmap | Inode 区 | Data Bitmap | Data 区。
- **Inode / DiskInode**：文件与目录的磁盘表示；**FileHandle**：内核中已打开文件。
- **文件描述符**：fd_table 下标；**open/close/read/write**：基于 fd 的文件 I/O。
