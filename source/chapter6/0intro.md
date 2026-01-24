# 引言

## 本章导读

本章我们将实现一个简单的文件系统，能够对 **持久存储设备** (Persistent Storage) I/O 资源进行管理；将设计两种文件：常规文件和目录文件，它们均以文件系统所维护的 **磁盘文件** 形式被组织并保存在持久存储设备上。

本章使用 **easy-fs** 作为简易文件系统实现，并通过 VirtIO 块设备访问持久存储。

## 实践体验

在 qemu 模拟器上运行本章代码：

```bash
$ cd ch6
$ cargo run
```

内核初始化完成之后就会进入 shell 程序。运行本章测例 `ch6_file0`（或 `filetest_simple`）：

```
>> ch6_file0
file_test passed!
Shell: Process 2 exited with code 0
>>
```

它会将 `Hello, world!` 输出到另一个文件 `filea`，并读取里面的内容确认输出正确。也可以通过 `ch6_file1`（或 `cat_filea`）查看 `filea` 中的内容：

```
>> ch6_file1
Hello, world!
Shell: Process 2 exited with code 0
>>
```

## 本章代码树

```
rCore-Tutorial-in-single-workspace/
├── ch6/
│   ├── Cargo.toml          # 项目配置（含 easy-fs、virtio-drivers）
│   ├── build.rs            # 构建脚本
│   └── src/
│       ├── main.rs         # 内核主函数，含 MMIO、文件系统初始化
│       ├── fs.rs           # 文件系统封装（FS、read_all）
│       ├── process.rs      # 进程（含 fd_table、从 FS 加载 ELF）
│       ├── processor.rs    # 处理器管理
│       └── virtio_block.rs # VirtIO 块设备驱动
├── easy-fs/                # 简易文件系统库
│   └── src/
│       ├── block_dev.rs    # BlockDevice trait
│       ├── block_cache.rs  # 块缓存
│       ├── layout.rs       # 磁盘布局与数据结构
│       ├── efs.rs          # EasyFileSystem
│       ├── vfs.rs          # Inode 等 VFS 抽象
│       └── ...
├── xtask/                  # 构建与打包（制作 fs 镜像等）
├── kernel-vm/
├── kernel-context/
├── kernel-alloc/
├── syscall/                # 含 open/close/read/write 等
└── ...
```

本章相比第五章的主要变化：

- **移除 loader**：不再将用户程序链接进内核，改为从文件系统加载
- **easy-fs**：接入 easy-fs，从块设备上的文件系统镜像读写文件
- **VirtIO 块设备**：实现 BlockDevice，对接 QEMU 的 virtio-blk
- **MMIO**：内核地址空间映射 VirtIO MMIO 区间（如 `0x1000_1000`）
- **文件描述符表**：进程增加 `fd_table`，支持 `open/close/read/write`
- **exec 从文件系统加载**：通过 `FS.open` 等接口读取 ELF，再 `Process::from_elf`

## 关键概念

- **持久存储**：块设备上的数据在断电后仍保留
- **块设备**：以固定大小的块为单位读写的存储设备
- **easy-fs**：简化的磁盘布局与 Inode 文件系统
- **文件描述符**：非负整数，表示进程 `fd_table` 中的一项
- **VirtIO 块设备**：QEMU virt 机器上的虚拟块设备，通过 MMIO 访问
