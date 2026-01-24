# 进程管理的核心数据结构

## 概述

为了更好实现进程管理，我们需要设计和调整内核中的一些数据结构，包括：

- 基于应用名的应用链接/加载器
- 进程标识符 `ProcId` 以及进程管理
- 进程控制块 `Process`
- 进程管理器 `ProcManager` 和 `PManager`
- 处理器管理结构 `Processor`

## 基于应用名的应用链接/加载器

在实现 `exec` 系统调用的时候，我们需要根据应用的名字而不仅仅是一个编号来获取应用的 ELF 格式数据。

在本项目的代码框架中，应用程序通过 `linker` 库链接进来，并且可以通过应用名来查找：

```rust
// ch5/src/main.rs

/// 加载用户进程。
static APPS: Lazy<BTreeMap<&'static str, &'static [u8]>> = Lazy::new(|| {
    extern "C" {
        static app_names: u8;
    }
    unsafe {
        linker::AppMeta::locate()
            .iter()
            .scan(&app_names as *const _ as usize, |addr, data| {
                let name = CStr::from_ptr(*addr as _).to_str().unwrap();
                *addr += name.as_bytes().len() + 1;
                Some((name, data))
            })
    }
    .collect()
});
```

这样，我们可以通过应用名来查找对应的 ELF 数据：

```rust
// ch5/src/main.rs (impls 模块)

impl Process for SyscallContext {
    fn exec(&self, _caller: Caller, path: usize, count: usize) -> isize {
        const READABLE: VmFlags<Sv39> = VmFlags::build_from_str("RV");
        let current = PROCESSOR.get_mut().current().unwrap();
        current
            .address_space
            .translate(VAddr::new(path), READABLE)
            .map(|ptr| unsafe {
                core::str::from_utf8_unchecked(core::slice::from_raw_parts(ptr.as_ptr(), count))
            })
            .and_then(|name| APPS.get(name))
            .and_then(|input| ElfFile::new(input).ok())
            .map_or_else(
                || {
                    log::error!("unknown app, select one in the list: ");
                    APPS.keys().for_each(|app| println!("{app}"));
                    -1
                },
                |data| {
                    current.exec(data);
                    0
                },
            )
    }
}
```

## 进程标识符和进程管理

在本项目的代码框架中，进程标识符由 `rcore-task-manage` 库提供：

```rust
// task-manage/src/id.rs

pub struct ProcId {
    // ...
}
```

`ProcId` 提供了进程 ID 的抽象，可以唯一标识一个进程。

## 进程控制块

在本项目的代码框架中，进程控制块由 `Process` 结构体表示：

```rust
// ch5/src/process.rs

pub struct Process {
    /// 不可变
    pub pid: ProcId,
    /// 可变
    pub context: ForeignContext,
    pub address_space: AddressSpace<Sv39, Sv39Manager>,
}
```

这个结构体包含：
- `pid`：进程标识符
- `context`：进程的执行上下文（`ForeignContext`），包含 `LocalContext` 和 `satp`
- `address_space`：进程的地址空间

## 进程管理器

在本项目的代码框架中，进程管理由 `rcore-task-manage` 库提供。主要的数据结构是 `PManager`：

```rust
// task-manage/src/proc_manage.rs

pub struct PManager<P, MP: Manage<P, ProcId> + Schedule<ProcId>> {
    rel_map: BTreeMap<ProcId, ProcRel>,  // 进程之间父子关系
    manager: Option<MP>,                  // 进程对象管理和调度
    current: Option<ProcId>,              // 当前正在运行的进程 ID
}
```

`PManager` 提供了：
- 进程管理：通过 `Manage` trait 管理进程对象
- 进程调度：通过 `Schedule` trait 进行进程调度
- 父子关系：通过 `ProcRel` 管理进程之间的父子关系

### ProcManager

在 `ch5/src/processor.rs` 中，我们实现了 `ProcManager`：

```rust
// ch5/src/processor.rs

pub struct ProcManager {
    tasks: BTreeMap<ProcId, Process>,
    ready_queue: VecDeque<ProcId>,
}
```

`ProcManager` 实现了 `Manage<Process, ProcId>` 和 `Schedule<ProcId>` trait：
- `Manage`：提供进程的插入、获取、删除操作
- `Schedule`：提供进程的调度队列操作（`add` 和 `fetch`）

## 处理器管理结构

在 `ch5/src/processor.rs` 中，我们定义了 `Processor` 结构：

```rust
// ch5/src/processor.rs

pub struct Processor {
    inner: UnsafeCell<PManager<Process, ProcManager>>,
}

pub static PROCESSOR: Processor = Processor::new();
```

`Processor` 封装了 `PManager`，提供了对进程管理器的访问接口。

### 当前进程访问

通过 `PROCESSOR` 可以访问当前正在运行的进程：

```rust
// ch5/src/main.rs

let processor: *mut PManager<Process, ProcManager> = PROCESSOR.get_mut() as *mut _;
if let Some(task) = unsafe { (*processor).find_next() } {
    // 执行任务
}
```

`PManager::find_next()` 会：
1. 从调度队列中取出一个进程 ID
2. 获取对应的进程对象
3. 设置为当前进程
4. 返回进程对象的可变引用

## 进程关系管理

`PManager` 通过 `ProcRel` 来管理进程之间的父子关系：

```rust
// task-manage/src/proc_rel.rs

pub struct ProcRel {
    parent: ProcId,
    children: Vec<ProcId>,
    // ...
}
```

`ProcRel` 记录了：
- `parent`：父进程 ID
- `children`：子进程 ID 列表

这样，我们可以：
- 查找一个进程的所有子进程
- 查找一个进程的父进程
- 维护进程之间的层次关系

## 关键概念总结

1. **进程控制块**：`Process` 结构体，包含进程的所有信息
2. **进程管理器**：`PManager` 和 `ProcManager`，管理进程和调度
3. **进程 ID**：`ProcId`，唯一标识一个进程
4. **进程关系**：`ProcRel`，管理进程之间的父子关系
5. **处理器管理**：`Processor`，封装进程管理器并提供访问接口

这些数据结构是进程管理的基础，在下一节中我们会介绍如何实现进程的创建、调度和资源回收。
