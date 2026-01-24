# 进程管理机制的设计实现

## 概述

本节将介绍如何基于上一节设计的内核数据结构来实现进程管理：

- 初始进程 `initproc` 的创建
- 进程调度机制：当进程主动调用 `sys_yield` 交出 CPU 使用权，或者内核本轮分配的时间片用尽之后如何切换到下一个进程
- 进程生成机制：介绍进程相关的两个重要系统调用 `sys_fork/sys_exec` 的实现
- 字符输入机制：介绍 `sys_read` 系统调用的实现
- 进程资源回收机制：当进程调用 `sys_exit` 正常退出或者出错被内核终止后，如何保存其退出码，其父进程又是如何通过 `sys_waitpid` 收集该进程的信息并回收其资源

## 初始进程的创建

内核初始化完毕之后，即会创建初始进程 `initproc`：

```rust
// ch5/src/main.rs

extern "C" fn rust_main() -> ! {
    // ... 初始化工作
    
    // 加载初始进程
    let initproc_data = APPS.get("initproc").unwrap();
    if let Some(process) = Process::from_elf(ElfFile::new(initproc_data).unwrap()) {
        PROCESSOR.get_mut().set_manager(ProcManager::new());
        PROCESSOR
            .get_mut()
            .add(process.pid, process, ProcId::from_usize(usize::MAX));
    }
    // ...
}
```

我们调用 `Process::from_elf()` 来创建一个进程，它需要传入 ELF 可执行文件的数据切片作为参数。然后将其加入到进程管理器中。

### Process::from_elf

`Process::from_elf()` 方法的实现：

```rust
// ch5/src/process.rs

impl Process {
    pub fn from_elf(elf: ElfFile) -> Option<Self> {
        // 解析 ELF 入口点
        let entry = // ...
        
        // 创建地址空间并映射 ELF 段
        let mut address_space = AddressSpace::new();
        for program in elf.program_iter() {
            if !matches!(program.get_type(), Ok(program::Type::Load)) {
                continue;
            }
            // 映射程序段
            address_space.map(/* ... */);
        }
        
        // 映射用户栈
        // ...
        
        // 映射异界传送门
        map_portal(&address_space);
        
        // 创建上下文
        let mut context = LocalContext::user(entry);
        let satp = (8 << 60) | address_space.root_ppn().val();
        *context.sp_mut() = 1 << 38;
        
        Some(Self {
            pid: ProcId::new(),
            context: ForeignContext { context, satp },
            address_space,
        })
    }
}
```

这个方法会：
1. 解析 ELF 文件，获取入口点
2. 创建地址空间并映射 ELF 的各个段
3. 映射用户栈
4. 映射传送门
5. 创建进程上下文
6. 返回 `Process` 对象

## 进程调度机制

在 `ch5/src/main.rs` 的 `rust_main` 函数中，我们实现了进程调度循环：

```rust
// ch5/src/main.rs

loop {
    let processor: *mut PManager<Process, ProcManager> = PROCESSOR.get_mut() as *mut _;
    if let Some(task) = unsafe { (*processor).find_next() } {
        unsafe { task.context.execute(portal, ()) };
        match scause::read().cause() {
            scause::Trap::Exception(scause::Exception::UserEnvCall) => {
                // 处理系统调用
                // ...
            }
            e => {
                // 处理异常
                unsafe { (*processor).make_current_exited(-3) };
            }
        }
    } else {
        println!("no task");
        break;
    }
}
```

这个循环会：
1. 从进程管理器中取出下一个可运行的进程
2. 执行该进程（`context.execute()`）
3. 当发生 Trap 时，根据 Trap 类型进行处理
4. 如果是系统调用，处理系统调用并根据结果决定是否切换
5. 如果是异常，标记进程退出

### 进程切换

当进程主动让出 CPU 或时间片用尽时，会调用 `make_current_suspend()`：

```rust
// ch5/src/main.rs (impls 模块)

impl Process for SyscallContext {
    fn sched_yield(&self, _caller: Caller) -> isize {
        0
    }
}
```

在系统调用处理中：

```rust
// ch5/src/main.rs

match syscall::handle(Caller { entity: 0, flow: 0 }, id, args) {
    Ret::Done(ret) => match id {
        Id::EXIT => unsafe { (*processor).make_current_exited(ret) },
        _ => {
            let ctx = &mut task.context.context;
            *ctx.a_mut(0) = ret as _;
            unsafe { (*processor).make_current_suspend() };
        }
    },
    // ...
}
```

`make_current_suspend()` 会：
1. 将当前进程重新加入调度队列
2. 清除当前进程标记
3. 下次循环时会选择下一个进程执行

## 进程生成机制

### fork 系统调用的实现

实现 fork 时，最为关键且困难一点的是为子进程创建一个和父进程几乎完全相同的地址空间。

在 `ch5/src/process.rs` 中，我们实现了 `Process::fork()`：

```rust
// ch5/src/process.rs

impl Process {
    pub fn fork(&mut self) -> Option<Process> {
        // 子进程 pid
        let pid = ProcId::new();
        // 复制父进程地址空间
        let parent_addr_space = &self.address_space;
        let mut address_space: AddressSpace<Sv39, Sv39Manager> = AddressSpace::new();
        parent_addr_space.cloneself(&mut address_space);
        map_portal(&address_space);
        // 复制父进程上下文
        let context = self.context.context.clone();
        let satp = (8 << 60) | address_space.root_ppn().val();
        let foreign_ctx = ForeignContext { context, satp };
        Some(Self {
            pid,
            context: foreign_ctx,
            address_space,
        })
    }
}
```

`fork()` 方法会：
1. 为子进程分配新的 PID
2. 复制父进程的地址空间（使用 `cloneself()` 方法）
3. 映射传送门
4. 复制父进程的上下文
5. 创建新的 `Process` 对象

在系统调用处理中：

```rust
// ch5/src/main.rs (impls 模块)

impl Process for SyscallContext {
    fn fork(&self, _caller: Caller) -> isize {
        let processor: *mut PManager<ProcStruct, ProcManager> = PROCESSOR.get_mut() as *mut _;
        let current = unsafe { (*processor).current().unwrap() };
        let mut child_proc = current.fork().unwrap();
        let pid = child_proc.pid;
        let context = &mut child_proc.context.context;
        *context.a_mut(0) = 0 as _;  // 子进程返回 0
        unsafe { (*processor).add(pid, child_proc, current.pid) };
        pid.get_usize() as isize  // 父进程返回子进程 PID
    }
}
```

注意：
- 子进程的返回值（`a0` 寄存器）被设置为 0
- 父进程的返回值是子进程的 PID
- 子进程被加入到进程管理器中

### exec 系统调用的实现

`exec` 系统调用使得一个进程能够加载一个新的 ELF 可执行文件替换原有的应用地址空间并开始执行。

在 `ch5/src/process.rs` 中，我们实现了 `Process::exec()`：

```rust
// ch5/src/process.rs

impl Process {
    pub fn exec(&mut self, elf: ElfFile) {
        let proc = Process::from_elf(elf).unwrap();
        self.address_space = proc.address_space;
        self.context = proc.context;
    }
}
```

`exec()` 方法会：
1. 解析新的 ELF 文件，创建新的地址空间和上下文
2. 替换当前进程的地址空间和上下文
3. 原有的地址空间会被自动回收（RAII）

在系统调用处理中：

```rust
// ch5/src/main.rs (impls 模块)

impl Process for SyscallContext {
    fn exec(&self, _caller: Caller, path: usize, count: usize) -> isize {
        // 从用户地址空间读取应用名
        // 查找应用 ELF 数据
        // 调用 exec 替换地址空间
        // ...
    }
}
```

## sys_read 获取输入

我们需要实现 `sys_read` 系统调用，使应用能够取得用户的键盘输入。

```rust
// ch5/src/main.rs (impls 模块)

impl IO for SyscallContext {
    fn read(&self, _caller: Caller, fd: usize, buf: usize, count: usize) -> isize {
        if fd == STDIN {
            const WRITEABLE: VmFlags<Sv39> = VmFlags::build_from_str("W_V");
            if let Some(mut ptr) = PROCESSOR
                .get_mut()
                .current()
                .unwrap()
                .address_space
                .translate(VAddr::new(buf), WRITEABLE)
            {
                let mut ptr = unsafe { ptr.as_mut() } as *mut u8;
                for _ in 0..count {
                    let c = sbi::console_getchar() as u8;
                    unsafe {
                        *ptr = c;
                        ptr = ptr.add(1);
                    }
                }
                count as _
            } else {
                -1
            }
        } else {
            -1
        }
    }
}
```

`sys_read` 的实现：
1. 检查文件描述符是否为标准输入（STDIN）
2. 手动查页表找到用户缓冲区在内核地址空间中的位置
3. 从 SBI 获取字符并写入缓冲区
4. 返回读取的字节数

## 进程资源回收机制

### 进程的退出

当应用调用 `sys_exit` 系统调用主动退出，或者出错由内核终止之后，会在内核中调用 `make_current_exited()`：

```rust
// ch5/src/main.rs

match syscall::handle(Caller { entity: 0, flow: 0 }, id, args) {
    Ret::Done(ret) => match id {
        Id::EXIT => unsafe { (*processor).make_current_exited(ret) },
        // ...
    },
    // ...
}
```

`make_current_exited()` 会：
1. 从进程管理器中删除当前进程
2. 处理进程的父子关系（将子进程转移给 initproc）
3. 清除当前进程标记
4. 进程的资源会被自动回收（RAII）

### 父进程回收子进程资源

`wait` 系统调用的实现：

```rust
// ch5/src/main.rs (impls 模块)

impl Process for SyscallContext {
    fn wait(&self, _caller: Caller, pid: isize, exit_code_ptr: usize) -> isize {
        let processor: *mut PManager<ProcStruct, ProcManager> = PROCESSOR.get_mut() as *mut _;
        let current = unsafe { (*processor).current().unwrap() };
        const WRITABLE: VmFlags<Sv39> = VmFlags::build_from_str("W_V");
        if let Some((dead_pid, exit_code)) =
            unsafe { (*processor).wait(ProcId::from_usize(pid as usize)) }
        {
            if let Some(mut ptr) = current
                .address_space
                .translate::<i32>(VAddr::new(exit_code_ptr), WRITABLE)
            {
                unsafe { *ptr.as_mut() = exit_code as i32 };
            }
            return dead_pid.get_usize() as isize;
        } else {
            // 等待的子进程不存在
            return -1;
        }
    }
}
```

`wait` 系统调用的处理流程：
1. 调用 `PManager::wait()` 查找已结束的子进程
2. 如果找到，获取子进程的退出码
3. 将退出码写入用户地址空间
4. 返回子进程的 PID
5. 子进程的资源会被自动回收（当引用计数为 0 时）

## 关键概念总结

1. **进程创建**：通过 `Process::from_elf()` 创建新进程
2. **进程复制**：通过 `Process::fork()` 创建子进程
3. **程序加载**：通过 `Process::exec()` 加载新程序
4. **进程调度**：通过 `PManager::find_next()` 选择下一个进程
5. **进程退出**：通过 `PManager::make_current_exited()` 处理进程退出
6. **资源回收**：通过 `PManager::wait()` 等待子进程并回收资源

这些机制共同构成了完整的进程管理系统，使得操作系统能够支持多进程并发执行和进程间的协作。
