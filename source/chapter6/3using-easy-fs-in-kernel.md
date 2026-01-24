# 在内核中使用 easy-fs

## 块设备驱动层

### VirtIO 块设备

在 QEMU `virt` 机器上，使用 **VirtIO 块设备** 作为持久存储。运行 QEMU 时通过 `-drive` 指定镜像文件，并通过 `-device virtio-blk-device` 将该镜像作为块设备挂载。例如：

```bash
-drive file=fs.img,if=none,format=raw,id=x0
-device virtio-blk-device,drive=x0,bus=virtio-mmio-bus.0
```

VirtIO 设备通过 **MMIO**（内存映射 I/O）访问。常见配置下，virtio-blk 的 MMIO 基址为 `0x1000_1000`，范围 4KiB。内核需要在创建地址空间时映射该区间，才能访问设备寄存器。

### ch6 中的实现

在 `ch6/src/virtio_block.rs` 中：

- 定义 `VIRTIO0 = 0x10001000`，实现 `Hal` trait（如 `dma_alloc`、`virt_to_phys` 等），供 `virtio-drivers` 使用。
- 实现 `BlockDevice` trait，内部通过 `VirtIOBlk` 完成 `read_block` / `write_block`。
- 将 `VirtIOBlock` 封装为 `BLOCK_DEVICE` 全局实例，作为 easy-fs 的底层块设备。

在 `ch6/src/main.rs` 中，`kernel_space` 会映射 `MMIO` 列表（含 `(0x1000_1000, 0x1000)`），从而访问 VirtIO 块设备。

## 文件系统初始化与 FS 全局

### 打开文件系统与根 Inode

内核启动时，从 `BLOCK_DEVICE` 上 **打开** 已存在的 easy-fs 镜像：

```rust
// ch6/src/fs.rs（示意）

pub static FS: Lazy<FileSystem> = Lazy::new(|| {
    let efs = EasyFileSystem::open(BLOCK_DEVICE.clone());
    FileSystem {
        root: EasyFileSystem::root_inode(&efs),
    }
});
```

`FS` 提供 `open`、`find`、`readdir` 等，内部基于 `root` 的 `Inode` 完成根目录下的创建、查找、列举。

### 加载初始进程

不再从链接进内核的二进制加载 initproc，而是 **从文件系统读取**：

```rust
// ch6/src/main.rs（示意）

let initproc = read_all(FS.open("initproc", OpenFlags::RDONLY).unwrap());
if let Some(process) = Process::from_elf(ElfFile::new(initproc.as_slice()).unwrap()) {
    // 加入 PROCESSOR，启动调度
}
```

`FS.open` 返回 `FileHandle`，`read_all` 将该文件整体读入内存，得到 ELF 字节流，再交给 `Process::from_elf`。这样，initproc 以及后续用户程序都存放在 `fs.img` 中，由 easy-fs 管理。

## 文件描述符与 sys_open / sys_read / sys_write / sys_close

### fd_table 与 FileHandle

每个进程维护 `fd_table: Vec<Option<Mutex<FileHandle>>>`。`FileHandle` 对应一个已打开的文件（或 Inode 的封装），支持 `read` / `write` 等。`open` 时在 `fd_table` 中找空闲 slot，放入 `FileHandle` 并返回 fd；`close` 时清空对应 slot。

### sys_open

- 从用户空间解析 path、flags。
- 调用 `FS.open(path, flags)` 获取 `FileHandle`；若为 `CREATE` 且文件不存在则创建。
- 将 `FileHandle` 放入当前进程 `fd_table`，返回 fd。

### sys_read / sys_write

- 若 fd 为 0（Stdin），则从控制台读；若为 1/2（Stdout/Stderr），则向控制台写。
- 否则在 `fd_table[fd]` 中取 `FileHandle`，调用其 `read` / `write`。内核负责把用户 `buf` 翻译为安全的内核缓冲区（如 `UserBuffer`），再传入 `FileHandle`。

### sys_close

- 校验 fd 合法且已打开，将 `fd_table[fd]` 置为 `None`，即关闭文件。

## 通过 sys_exec 从文件系统加载应用

`sys_exec` 不再从链接进内核的应用列表取 ELF，而是 **按 path 从文件系统打开并读取**：

```rust
// ch6 impls 中（示意）

fn exec(&self, _caller: Caller, path: usize, count: usize) -> isize {
    let path_str = /* 从用户空间解析 path */;
    FS.open(path_str, OpenFlags::RDONLY)
        .map(|fd| {
            let elf_data = read_all(fd);
            current.exec(ElfFile::new(&elf_data).unwrap());
            0
        })
        .unwrap_or(-1)
}
```

即：用 `FS.open` 打开 path 对应文件，`read_all` 读入完整 ELF，再调用 `Process::exec` 替换地址空间并执行。找不到文件或打开失败时返回 -1。

## 小结

- **块设备**：`VirtIOBlock` 实现 `BlockDevice`，MMIO 映射 `0x10001000`，读写 `fs.img`。
- **文件系统**：`FS` 在启动时 `EasyFileSystem::open(BLOCK_DEVICE)`，对外提供 `open` / `find` / `readdir` 等。
- **进程与 fd**：进程带 `fd_table`，`sys_open` / `sys_close` 管理 `FileHandle`，`sys_read` / `sys_write` 统一处理标准 I/O 与普通文件。
- **加载程序**：initproc 与 `exec` 均从 easy-fs 读取 ELF，实现从文件系统加载并执行应用。
