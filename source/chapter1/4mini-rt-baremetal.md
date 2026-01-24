# 构建裸机执行环境

本节中，我们将把 `Hello world!` 应用程序从用户态搬到内核态，构建在裸机上的最小执行环境。

## 裸机启动过程

用 QEMU 软件 `qemu-system-riscv64` 来模拟 RISC-V 64 计算机。加载内核程序的命令如下：

```bash
qemu-system-riscv64 \
    -machine virt \
    -nographic \
    -bios $(BOOTLOADER) \
    -device loader,file=$(KERNEL_BIN),addr=$(KERNEL_ENTRY_PA)
```

- `-bios $(BOOTLOADER)` 意味着硬件加载了一个 BootLoader 程序，即 RustSBI
- `-device loader,file=$(KERNEL_BIN),addr=$(KERNEL_ENTRY_PA)` 表示硬件内存中的特定位置 `$(KERNEL_ENTRY_PA)` 放置了操作系统的二进制代码 `$(KERNEL_BIN)`。`$(KERNEL_ENTRY_PA)` 的值是 `0x80200000`。

当我们执行包含上述启动参数的 `qemu-system-riscv64` 软件，就意味给这台虚拟的 RISC-V64 计算机加电了。此时，CPU 的其它通用寄存器清零，而 PC 会指向 `0x1000` 的位置，这里有固化在硬件中的一小段引导代码，它会很快跳转到 `0x80000000` 的 RustSBI 处。RustSBI 完成硬件初始化后，会跳转到 `$(KERNEL_BIN)` 所在内存位置 `0x80200000` 处，执行操作系统的第一条指令。

```
┌─────────────────────────────────────┐
│  0x1000: 硬件引导代码                │
│         ↓                           │
│  0x80000000: RustSBI                │
│         ↓                           │
│  0x80200000: 操作系统内核入口        │
└─────────────────────────────────────┘
```

> **RustSBI 是什么？**
> 
> SBI 是 RISC-V 的一种底层规范，RustSBI 是它的一种实现。操作系统内核与 RustSBI 的关系有点像应用与操作系统内核的关系，后者向前者提供一定的服务。只是 SBI 提供的服务很少，比如关机、显示字符串等。

## 实现关机功能

首先，我们需要使用 SBI 调用实现关机功能。在本项目的代码框架中，SBI 调用已经被封装在 `sbi` 库中。让我们看看如何使用它：

```rust
// src/main.rs
#![no_std]
#![no_main]

use sbi;

#[panic_handler]
fn panic(_: &core::panic::PanicInfo) -> ! {
    sbi::shutdown(true)
}
```

`shutdown` 函数接受一个布尔参数，`true` 表示异常关机，`false` 表示正常关机。

应用程序访问操作系统提供的系统调用的指令是 `ecall`，操作系统访问 RustSBI 提供的 SBI 调用的指令也是 `ecall`，虽然指令一样，但它们所在的特权级是不一样的。简单地说，应用程序位于最弱的用户特权级（User Mode），操作系统位于内核特权级（Supervisor Mode），RustSBI 位于机器特权级（Machine Mode）。下一章会进一步阐释具体细节。

## 设置正确的程序内存布局

可以通过 **链接脚本** (Linker Script) 调整链接器的行为，使得最终生成的可执行文件的内存布局符合我们的预期。

在本项目的代码框架中，链接脚本被放在 `build.rs` 文件中。让我们看看 `ch1/build.rs`：

```rust
const LINKER: &[u8] = b"
OUTPUT_ARCH(riscv)
SECTIONS {
    .text 0x80200000 : {
        *(.text.entry)
        *(.text .text.*)
    }
    .rodata : {
        *(.rodata .rodata.*)
        *(.srodata .srodata.*)
    }
    .data : {
        *(.data .data.*)
        *(.sdata .sdata.*)
    }
    .bss : {
        *(.bss.uninit)
        *(.bss .bss.*)
        *(.sbss .sbss.*)
    }
}";
```

第 1 行我们设置了目标平台为 riscv；第 2 行我们设置了代码段的起始地址为 `0x80200000`，这是 RustSBI 期望的 OS 起始地址。

从 `0x80200000` 开始，代码段 `.text`、只读数据段 `.rodata`、数据段 `.data`、bss 段 `.bss` 由低到高依次放置。

> **注意**：linker 脚本的语法不做要求，感兴趣的同学可以自行查阅相关资料。

## 正确配置栈空间和入口点

在本项目的代码框架中，我们使用 Rust 的 `naked` 函数特性来实现入口点。让我们看看 `ch1/src/main.rs` 的完整代码：

```rust
#![no_std]
#![no_main]
#![deny(warnings)]

use sbi;

/// Supervisor 汇编入口。
///
/// 设置栈并跳转到 Rust。
#[unsafe(naked)]
#[no_mangle]
#[link_section = ".text.entry"]
unsafe extern "C" fn _start() -> ! {
    const STACK_SIZE: usize = 4096;

    #[link_section = ".bss.uninit"]
    static mut STACK: [u8; STACK_SIZE] = [0u8; STACK_SIZE];

    core::arch::naked_asm!(
        "la sp, {stack} + {stack_size}",
        "j  {main}",
        stack_size = const STACK_SIZE,
        stack      =   sym STACK,
        main       =   sym rust_main,
    )
}

/// 非常简单的 Supervisor 裸机程序。
///
/// 打印 `Hello, World!`，然后关机。
extern "C" fn rust_main() -> ! {
    for c in b"Hello, world!\n" {
        sbi::console_putchar(*c);
    }
    sbi::shutdown(false)
}

/// Rust 异常处理函数，以异常方式关机。
#[panic_handler]
fn panic(_: &core::panic::PanicInfo) -> ! {
    sbi::shutdown(true)
}
```

让我们逐步分析这段代码：

### 入口点 `_start`

`_start` 函数是一个 **naked 函数**（`#[unsafe(naked)]`），这意味着编译器不会为它添加任何序言和尾声，因此可以在没有栈的情况下执行。

- `#[link_section = ".text.entry"]` 确保这个函数被放在链接脚本中定义的 `.text.entry` 段，也就是代码段的开始位置（`0x80200000`）
- 在函数内部，我们定义了一个静态的栈空间 `STACK`，大小为 4096 字节（4KB）
- `#[link_section = ".bss.uninit"]` 确保栈空间被放在 `.bss.uninit` 段
- 使用 `naked_asm!` 宏内联汇编代码：
  - `la sp, {stack} + {stack_size}`：将栈指针 `sp` 设置为栈空间的顶部
  - `j {main}`：跳转到 `rust_main` 函数

### 主函数 `rust_main`

`rust_main` 函数是实际执行逻辑的地方：

- 遍历 `"Hello, world!\n"` 字节数组中的每个字符
- 调用 `sbi::console_putchar` 打印每个字符
- 调用 `sbi::shutdown(false)` 正常关机

### Panic 处理

`panic` 函数在程序发生 panic 时被调用，它调用 `sbi::shutdown(true)` 进行异常关机。

## 编译和运行

编译生成可执行文件：

```bash
$ cd ch1
$ cargo build --release
```

运行程序（需要先准备好 RustSBI）：

```bash
$ qemu-system-riscv64 \
    -machine virt \
    -nographic \
    -bios ../rustsbi-qemu.bin \
    -device loader,file=target/riscv64gc-unknown-none-elf/release/ch1,addr=0x80200000
```

或者使用项目提供的运行脚本（如果有的话）。

预期输出：

```
Hello, world!
```

然后 QEMU 会优雅地退出。

## 代码结构总结

本章的代码结构非常简洁：

1. **入口点设置**：使用 naked 函数 `_start` 设置栈并跳转到 Rust 代码
2. **栈空间**：在 `.bss.uninit` 段中预留 4KB 栈空间
3. **内存布局**：通过 `build.rs` 中的链接脚本将代码放在 `0x80200000`
4. **SBI 调用**：使用 `sbi` 库提供的 `console_putchar` 和 `shutdown` 函数
5. **异常处理**：实现 `panic_handler` 在发生错误时关机

至此，我们完成了第一章的实验内容，成功在裸机上运行了一个能够打印 `Hello, world!` 的简单程序。
