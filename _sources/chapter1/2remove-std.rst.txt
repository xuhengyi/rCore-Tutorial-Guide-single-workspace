移除标准库依赖
==========================

本节的目标是：解释为什么课程代码中的 ``os`` 不能依赖 Rust 标准库 ``std``，以及当我们拿掉 ``std`` 之后，
需要在代码中补齐哪些“最小执行环境”要素。

.. note::

   本章文档中我们把要构建的内核工程统称为 ``os``。在课程提供的代码中，本章对应目录是
   ``rCore-Tutorial-in-single-workspace/ch1``。

交叉编译目标
----------------------------------

本章的内核运行在 RISC-V 裸机环境，因此编译目标是 ``riscv64gc-unknown-none-elf``。

如果你第一次使用该目标，可能会遇到：

.. code-block:: console

   error: target `riscv64gc-unknown-none-elf` not found

需要先安装 target：

.. code-block:: console

   $ rustup target add riscv64gc-unknown-none-elf

移除 println! 宏
----------------------------------

``println!`` 是由 ``std`` 提供的宏；而 ``std`` 依赖操作系统提供的系统调用（例如 ``write``）才能把字符打印到终端。
当我们把程序移到裸机上时，这些系统调用并不存在，所以内核不能使用 ``std``，也就不能直接使用 ``println!``。

在课程代码中，本章内核采用更原始但更清晰的方式输出字符：把字符串逐字节交给 SBI 输出接口：

.. code-block:: rust

   // rCore-Tutorial-in-single-workspace/ch1/src/main.rs（节选）
   #![no_std]
   #![no_main]

   mod sbi;

   extern "C" fn rust_main() -> ! {
       for c in b"Hello, world!\\n" {
           sbi::console_putchar(*c);
       }
       sbi::shutdown(false)
   }

.. note::

   这里的 ``b"Hello, world!\\n"`` 是一个**字节串**（类型是 ``&[u8]``）。
   因为我们此时不依赖 ``std``，也不急着引入复杂的格式化输出；逐字节输出能把依赖最小化，也最容易理解。

提供语义项 panic_handler
----------------------------------------------------

当我们禁用 ``std`` 之后，编译器会要求你提供 panic 的处理逻辑（``#[panic_handler]``）。
``std`` 在“有操作系统的环境”里会帮你打印错误信息并退出；但内核态/裸机程序没有这些默认能力，只能由我们自己决定“panic 时做什么”。

课程代码里，本章内核的策略很简单：直接通过 SBI 关机（把 panic 当成失败退出）：

.. code-block:: rust

   // rCore-Tutorial-in-single-workspace/ch1/src/main.rs（节选）
   #[panic_handler]
   fn panic(_: &core::panic::PanicInfo) -> ! {
       sbi::shutdown(true)
   }

移除 main 函数
-----------------------------

在没有 ``std`` 的情况下，Rust 仍然会尝试按“正常应用程序”的方式去寻找 ``main`` 并做一系列启动初始化（``start`` lang item）。
但内核/裸机程序不走这套路径，因此需要：

- 用 ``#![no_main]`` 告诉编译器“我不使用标准入口 main”；
- 自己提供入口函数 ``_start``，并在其中完成最小启动工作（例如设栈）。

课程代码中，本章内核的入口 ``_start`` 是一个 *naked* 函数：编译器不会为它生成函数序言/尾声，
因此它可以在**没有栈**的情况下执行，并由我们手动设置 ``sp``：

.. code-block:: rust

   // rCore-Tutorial-in-single-workspace/ch1/src/main.rs（节选）
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

把这段代码拆开解释：

- **``#[no_mangle]``**：告诉编译器不要改函数名；否则链接器/固件可能找不到 ``_start``；
- **``#[link_section = ".text.entry"]``**：把 ``_start`` 放进一个叫 ``.text.entry`` 的段里，方便链接脚本把它放到最前面；
- **``static mut STACK``**：在静态区预留一块内存当栈（本章先不讨论栈溢出等问题）；
- **``la sp, ...``**：把栈指针 ``sp`` 指到这块栈的“栈顶”（高地址）；
- **``j {main}``**：跳到 Rust 写的 ``rust_main`` 执行真正逻辑。

小结：我们“补齐了什么”
----------------------------------

本节我们并没有让你从零写出 ``os``，而是明确了课程代码里为了解决 “no_std 之后缺了什么” 做了哪些补齐：

- **入口**：提供 ``_start``，并在其中设栈；
- **panic**：提供 ``#[panic_handler]``；
- **输出与退出**：不依赖 ``std``，改为使用更底层的机制（例如 SBI）。

现在我们已经知道：没有 ``std`` 以后，程序需要自己提供入口、输出与退出能力。
接下来我们马上换到“用户态运行”的视角，看看课程代码如何组织 ``_start/exit/write`` 这条链路，
再把同样的思路迁移到裸机内核里。
