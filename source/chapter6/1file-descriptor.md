# 文件与文件描述符

## 文件简介

文件可代表很多种不同类型的 I/O 资源，但是在进程看来，所有文件的访问都可以通过一个简洁统一的抽象接口进行。

在本项目的代码框架中，easy-fs 通过 `FileHandle`、`FSManager` 等抽象与内核衔接。用户态通过 **文件描述符** (fd) 来访问已打开的文件，内核在进程的 **文件描述符表** (`fd_table`) 中维护 fd 到具体文件（如 `FileHandle`）的映射。

`read` 指从文件（或 I/O 资源）中读取数据放入缓冲区，最多填满缓冲区长度，并返回实际读取的字节数；`write` 指将缓冲区数据写入文件，最多写尽缓冲区，并返回实际写入的字节数。

## 标准输入和标准输出

我们沿用常见约定：**标准输入** 对应 fd 0（Stdin），**标准输出** 对应 fd 1（Stdout），**标准错误** 对应 fd 2（Stderr）。  
`sys_read` / `sys_write` 根据 fd 决定是操作标准输入/输出，还是操作 `fd_table` 中的普通文件。

## 文件描述符与文件描述符表

每个进程有一个 **文件描述符表** `fd_table`，记录当前进程打开的文件。**文件描述符** 是一个非负整数，表示该表中的下标。  
通过 `open` / `openat` 打开文件时，内核在 `fd_table` 中分配一个空闲位置，返回对应的 fd；通过 `close` 关闭文件时，则释放该位置。

在本章实现中，`Process` 包含 `fd_table`：

```rust
// ch6/src/process.rs

pub struct Process {
    pub pid: ProcId,
    pub context: ForeignContext,
    pub address_space: AddressSpace<Sv39, Sv39Manager>,
    /// 文件描述符表
    pub fd_table: Vec<Option<Mutex<FileHandle>>>,
}
```

- `Vec` 实现动态大小的 fd 表
- `Option` 区分该 slot 是否已被占用（`None` 表示空闲）
- `Mutex<FileHandle>` 表示一个已打开的文件，支持并发访问控制

新建进程时，会为 Stdin、Stdout、Stderr 等预留 fd 并填入对应的 `FileHandle`（或类似抽象）。`fork` 时子进程会继承父进程的 `fd_table`（或其拷贝），从而共享已打开的文件。

## 文件 I/O 与系统调用

`sys_read` / `sys_write` 通过 fd 在当前进程的 `fd_table` 中找到对应的 `FileHandle`，然后调用其 `read` / `write` 方法。若 fd 超出范围或该 slot 为 `None`，则返回错误。

在 `ch6` 的 `impls::IO` 中，大致逻辑为：

- `sys_write(fd, buf, count)`：若 `fd` 为 Stdout/Stderr，则向控制台输出；否则在 `fd_table[fd]` 中取 `FileHandle`，写入 `buf` 的 `count` 字节。
- `sys_read(fd, buf, count)`：若 `fd` 为 Stdin，则从控制台读入；否则从 `fd_table[fd]` 对应的文件中读取到 `buf`。

内核会通过地址空间翻译把用户提供的 `buf` 转为可安全访问的内核缓冲区（例如 `UserBuffer` 或 `translated_byte_buffer`），再交给具体文件实现进行读写。

## 小结

- 文件是对 I/O 资源的统一抽象；用户通过 fd 访问。
- 每个进程有 `fd_table`，fd 为表内索引。
- 约定 fd 0/1/2 为标准输入/输出/错误；其余 fd 对应 `open` 打开的文件。
- `read` / `write` 根据 fd 查找 `fd_table`，再调用具体文件的读写接口，从而统一处理标准 I/O 与普通文件。
