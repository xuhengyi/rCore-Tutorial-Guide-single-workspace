# 与进程有关的重要系统调用

## 重要系统调用

### fork 系统调用

```rust
/// 功能：由当前进程 fork 出一个子进程。
/// 返回值：对于子进程返回 0，对于当前进程则返回子进程的 PID。
/// syscall ID：220
pub fn sys_fork() -> isize;
```

`fork` 系统调用会创建一个与当前进程几乎完全相同的子进程。父子进程的区别在于：
- 父进程的返回值是子进程的 PID
- 子进程的返回值是 0

### exec 系统调用

```rust
/// 功能：将当前进程的地址空间清空并加载一个特定的可执行文件，返回用户态后开始它的执行。
/// 参数：字符串 path 给出了要加载的可执行文件的名字；
/// 返回值：如果出错的话（如找不到名字相符的可执行文件）则返回 -1，否则不应该返回。
/// 注意：path 必须以 "\0" 结尾，否则内核将无法确定其长度
/// syscall ID：221
pub fn sys_exec(path: &str) -> isize;
```

`exec` 系统调用会用新的程序替换当前进程的地址空间。调用成功后不会返回，因为程序已经被替换了。

利用 `fork` 和 `exec` 的组合，我们能让创建一个子进程，并令其执行特定的可执行文件。

### waitpid 系统调用

```rust
/// 功能：当前进程等待一个子进程变为僵尸进程，回收其全部资源并收集其返回值。
/// 参数：pid 表示要等待的子进程的进程 ID，如果为 -1 的话表示等待任意一个子进程；
///      exit_code 表示保存子进程返回值的地址，如果这个地址为 0 的话表示不必保存。
/// 返回值：如果要等待的子进程不存在则返回 -1；否则如果要等待的子进程均未结束则返回 -2；
///        否则返回结束的子进程的进程 ID。
/// syscall ID：260
pub fn sys_waitpid(pid: isize, exit_code: *mut i32) -> isize;
```

`sys_waitpid` 在用户库中被封装成两个不同的 API，`wait(exit_code: &mut i32)` 和 `waitpid(pid: usize, exit_code: &mut i32)`，前者用于等待任意一个子进程，后者用于等待特定子进程。它们实现的策略是如果子进程还未结束，就以 yield 让出时间片：

```rust
// user/src/lib.rs

pub fn wait(exit_code: &mut i32) -> isize {
    loop {
        match sys_waitpid(-1, exit_code as *mut _) {
            -2 => { sys_yield(); }
            n => { return n; }
        }
    }
}
```

### read 系统调用

```rust
/// 功能：从文件中读取一段内容到缓冲区。
/// 参数：fd 是待读取文件的文件描述符，切片 buffer 则给出缓冲区。
/// 返回值：如果出现了错误则返回 -1，否则返回实际读到的字节数。
/// syscall ID：63
pub fn sys_read(fd: usize, buffer: &mut [u8]) -> isize;
```

`read` 系统调用用于从标准输入读取用户键盘输入，这对于实现 Shell 程序至关重要。

## 应用程序示例

借助这三个重要系统调用，我们可以开发功能更强大的应用。下面是两个案例：**用户初始程序-initproc** 和 **shell程序-user_shell**。

### 用户初始程序-initproc

在内核初始化完毕后创建的第一个进程，是 **用户初始进程** (Initial Process)，它将通过 `fork+exec` 创建 `user_shell` 子进程，并将被用于回收僵尸进程。

```rust
// user/src/bin/ch5b_initproc.rs

#[no_mangle]
fn main() -> i32 {
    if fork() == 0 {
        exec("ch5b_user_shell\0");
    } else {
        loop {
            let mut exit_code: i32 = 0;
            let pid = wait(&mut exit_code);
            if pid == -1 {
                yield_();
                continue;
            }
            println!(
                "[initproc] Released a zombie process, pid={}, exit_code={}",
                pid,
                exit_code,
            );
        }
    }
    0
}
```

- `fork` 出的子进程分支，通过 `exec` 启动 shell 程序 `user_shell`
- 父进程分支不断循环调用 `wait` 来等待并回收系统中的僵尸进程占据的资源

### shell程序-user_shell

user_shell 需要捕获用户输入并进行解析处理。shell 程序的主要逻辑：

```rust
// user/src/bin/ch5b_user_shell.rs

#[no_mangle]
pub fn main() -> i32 {
    println!("Rust user shell");
    let mut line: String = String::new();
    print!(">> ");
    loop {
        let c = getchar();
        match c {
            LF | CR => {
                // 用户输入回车键
                if !line.is_empty() {
                    line.push('\0');
                    let pid = fork();
                    if pid == 0 {
                        // 子进程执行命令
                        if exec(line.as_str()) == -1 {
                            println!("Error when executing!");
                            return -4;
                        }
                        unreachable!();
                    } else {
                        // 父进程等待子进程结束
                        let mut exit_code: i32 = 0;
                        let exit_pid = waitpid(pid as usize, &mut exit_code);
                        println!(
                            "Shell: Process {} exited with code {}",
                            pid, exit_code
                        );
                    }
                    line.clear();
                }
                print!(">> ");
            }
            BS | DL => {
                // 用户输入退格键
                // 处理退格逻辑
            }
            _ => {
                // 其他字符
                print!("{}", c as char);
                line.push(c as char);
            }
        }
    }
}
```

shell 程序的工作流程：
1. 读取用户输入的字符
2. 如果输入回车，fork 一个子进程执行命令
3. 等待子进程结束并显示结果
4. 继续读取下一个命令

## 关键概念总结

1. **fork**：创建子进程，父子进程几乎完全相同
2. **exec**：加载新程序，替换当前进程的地址空间
3. **wait/waitpid**：等待子进程结束并回收资源
4. **read**：从标准输入读取用户输入
5. **initproc**：系统初始进程，负责启动 shell 和回收僵尸进程
6. **user_shell**：用户终端，提供命令行交互界面

这些系统调用是进程管理的基础，在下一节中我们会介绍如何在内核中实现它们。
