# 文件系统接口

## 简易文件与目录抽象

与完整 UNIX 文件系统相比，本章实现做了大量简化：

- **扁平化**：仅存在根目录，所有文件均放在根目录下，通过文件名直接索引。
- **无用户/组、时间戳**：不区分用户、用户组，不维护访问/修改时间，也不实现软硬链接。
- **仅实现最基础的文件系统相关系统调用**。

## 打开与读写文件的系统调用

### 打开文件

通过 `sys_openat`（或兼容的 `open` 封装）打开文件：

- **path**：要打开的文件名（根目录下的名字）。
- **flags**：打开方式，如只读、只写、读写、创建、截断等。

常用标志含义：

- `RDONLY`(0)：只读
- `WRONLY`(0x001)：只写
- `RDWR`(0x002)：读写
- `CREATE`(0x200)：若不存在则创建；若已存在则可配合 `TRUNC` 使用
- `TRUNC`(0x400)：打开时清空文件内容

成功时返回分配到的 **文件描述符**（即 `fd_table` 中的下标）；失败返回 -1。

### 顺序读写与关闭

打开后，用 `sys_read` / `sys_write` 按 **顺序** 读写文件内容，用 `sys_close` 关闭文件。本教程只考虑顺序读写，不涉及 `seek` 等随机读写。

## 测例：`filetest_simple`

以 `ch6_file0`（或 `filetest_simple`）为例，说明上述接口的用法：

```rust
// 以 只写 + 创建 打开 "filea"
let fd = open("filea\0", OpenFlags::CREATE | OpenFlags::WRONLY);
write(fd, b"Hello, world!");
close(fd);

// 以只读重新打开，读回并校验
let fd = open("filea\0", OpenFlags::RDONLY);
let mut buffer = [0u8; 100];
let n = read(fd, &mut buffer);
close(fd);
assert_eq!(core::str::from_utf8(&buffer[..n]).unwrap(), "Hello, world!");
```

- 先 `CREATE | WRONLY` 创建并写入 `filea`，然后 `close`。
- 再 `RDONLY` 打开同一文件，`read` 到缓冲区，校验后 `close`。

通常，`read` 可能需循环调用直至返回 0，才表示文件读完；这里因内容很短，一次 `read` 即可。

## 小结

- 打开：`open` / `sys_openat`，通过 path、flags 在根目录下创建或查找文件，返回 fd。
- 读写：`read` / `write` 通过 fd 在 `fd_table` 中找到对应 `FileHandle`，做顺序读写。
- 关闭：`close` 释放 fd，并将 `fd_table` 中该项置为空闲。

以上构成本章所用的简易文件系统接口，便于在 shell 与测例中进行文件操作。
