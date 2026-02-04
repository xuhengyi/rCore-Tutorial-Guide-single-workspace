.. _term-print-kernelminienv:

构建裸机执行环境
=================================

有了上一节实现的用户态的最小执行环境，稍加改造，就可以完成裸机上的最小执行环境了。
本节中，我们将把 ``Hello world!`` 应用程序从用户态搬到内核态。


裸机启动过程
----------------------------

用 QEMU 的系统态模拟器 ``qemu-system-riscv64`` 来模拟一台 RISC-V 64 计算机。

在课程代码中，推荐直接用：

.. code-block:: console

   $ cd rCore-Tutorial-in-single-workspace
   $ cargo qemu --ch 1

这条命令会先编译 ``ch1``，再启动 ``qemu-system-riscv64``。为了不让你一开始就被一长串参数吓到，
我们先给出**运行结果**（能看到 ``Hello, world!``），再回过头解释它背后发生了什么。

从 ``cargo qemu`` 到 QEMU 命令
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

课程代码在 ``.cargo/config.toml`` 中把 ``cargo qemu`` 配置成一个别名：

.. code-block:: toml

   # rCore-Tutorial-in-single-workspace/.cargo/config.toml（节选）
   [alias]
   qemu = "xtask qemu"

也就是说，``cargo qemu --ch 1`` 实际会运行 ``xtask`` 包里的 ``qemu`` 子命令。
在 ``xtask/src/main.rs`` 中可以看到它如何拼出 QEMU 启动参数（节选）：

.. code-block:: rust

   // rCore-Tutorial-in-single-workspace/xtask/src/main.rs（节选）
   let mut qemu = Qemu::system(self.build.arch.qemu_system());
   qemu.args(&["-machine", "virt"])
       .arg("-nographic");

   if self.build.nobios {
       qemu.args(&["-bios", "none"]);
   } else {
       qemu.arg("-bios").arg(PROJECT.join("rustsbi-qemu.bin"));
   }

   qemu.arg("-kernel")
       .arg(objcopy(elf, true))
       .args(&["-smp", &self.smp.unwrap_or(1).to_string()])
       .args(&["-m", "64M"])
       .args(&["-serial", "mon:stdio"]);

因此，对本章默认的 SBI 模式来说，它等价于运行一条形如下面的 QEMU 命令（这里把路径简化成变量）：

.. code-block:: bash

    qemu-system-riscv64 \
		-machine virt \
		-nographic \
		-bios $(RUSTSBI_BIN) \
		-kernel $(KERNEL_BIN) \
		-smp 1 -m 64M \
		-serial mon:stdio


-  ``-bios $(RUSTSBI_BIN)``：加载 RustSBI（M 态固件）
-  ``-kernel $(KERNEL_BIN)``：加载内核二进制；在 ``virt`` 机器上默认会放到 ``0x8020_0000`` 附近（与本章链接脚本一致）

当我们执行包含上述启动参数的 qemu-system-riscv64 软件，就意味给这台虚拟的 RISC-V64 计算机加电了。
此时，CPU 的其它通用寄存器清零，而 PC 会指向 ``0x1000`` 的位置，这里有固化在硬件中的一小段引导代码，
它会很快跳转到 ``0x80000000`` 的 RustSBI 处。
RustSBI完成硬件初始化后，会跳转到 ``$(KERNEL_BIN)`` 所在内存位置 ``0x80200000`` 处，
执行操作系统的第一条指令。

.. figure:: chap1-intro.png
   :align: center

.. note::

  **RustSBI 是什么？**

  SBI 是 RISC-V 的一种底层规范，RustSBI 是它的一种实现。
  操作系统内核与 RustSBI 的关系有点像应用与操作系统内核的关系，后者向前者提供一定的服务。只是SBI提供的服务很少，
  比如关机，显示字符串等。

实现裸机版 Hello, world!
------------------------------------------------

先跑起来：你会看到什么输出
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

执行 ``cargo qemu --ch 1`` 后，你会看到两部分输出：

- RustSBI 的启动信息（版本/平台/内存布局/内核入口地址等）
- ``Hello, world!``（来自 ``ch1`` 内核）

可以先抓住两条关键信息：

- RustSBI 打印的 ``Supervisor Address : 0x80200000``：这就是它将要跳转执行内核入口的位置；
- 我们的链接脚本也把 ``.text`` 放到了 ``0x8020_0000``：二者必须一致，否则就会“跳错地方”。

ch1 的关键文件
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

- ``rCore-Tutorial-in-single-workspace/ch1/src/main.rs``：S 态入口 ``_start``、设栈、``rust_main``；
- ``rCore-Tutorial-in-single-workspace/ch1/src/sbi.rs``：SBI 调用封装（通过 ``ecall``）；
- ``rCore-Tutorial-in-single-workspace/ch1/build.rs``：**生成链接脚本** 并传给链接器；
- （可选）``--nobios``：会额外引入 ``m_entry_rv64.asm`` / ``msbi.rs`` 来完成 M→S 切换与最小 SBI。

SBI 调用：输出与关机
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

在用户态时，我们用 ``ecall`` 调 Linux 系统调用；到了裸机 S 态，我们仍然用 ``ecall``，
但这次它会进入 RustSBI（M 态）并触发 **SBI 调用**。

课程代码在 ``ch1/src/sbi.rs`` 中实现了两个最小能力：

- ``console_putchar``：输出一个字符；
- ``shutdown``：关机（优先尝试 SRST 扩展，失败则回退到 legacy shutdown）。


.. code-block:: rust

   // rCore-Tutorial-in-single-workspace/ch1/src/sbi.rs（节选）
   asm!(
       "ecall",
       inlateout("a0") arg0 => error,
       inlateout("a1") arg1 => value,
       in("a2") arg2,
       in("a6") fid,
       in("a7") eid,
   );

它做的事可以概括成一句话：**把参数放进寄存器，然后执行 ``ecall`` 请求 M 态服务**。


入口与栈：为什么必须先设 sp
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

在裸机上，启动时**没有任何人替你准备栈**。因此第一件事就是把栈指针 ``sp`` 指到一段你预留的内存。

.. note::

   栈是向低地址增长的。我们把 ``sp`` 设为 ``stack + stack_size``（栈顶，高地址），
   后续函数调用/入栈会让 ``sp`` 逐步减小。

课程代码选择在 ``ch1/src/main.rs`` 里用一个 *naked* 入口来完成这件事（省去单独的 ``entry.asm`` 文件）：

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

链接脚本：由 build.rs 生成并注入
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

让内核“从正确地址开始执行”，离不开链接脚本。课程代码把链接脚本写在 ``build.rs`` 里并在编译时生成，
从而保证：

- SBI 模式下，内核代码与入口放在 ``0x8020_0000`` 附近；
- nobios 模式下，M 态入口从 ``0x8000_0000`` 开始，而 S 态内核从 ``0x8020_0000`` 开始。

 ``build.rs`` 中 SBI 模式的链接脚本片段：

.. code-block:: rust

   // rCore-Tutorial-in-single-workspace/ch1/build.rs（节选）
   // SBI mode: kernel at 0x80200000
   r#"
   OUTPUT_ARCH(riscv)
   SECTIONS {
       .text 0x80200000 : {
           *(.text.entry)
           *(.text .text.*)
       }
       .rodata : { *(.rodata .rodata.*) *(.srodata .srodata.*) }
       .data   : { *(.data   .data.*)   *(.sdata   .sdata.*)   }
       .bss    : { *(.bss.uninit) *(.bss .bss.*) *(.sbss .sbss.*) }
   }
   "#

你可以先把它理解为：把 ``.text`` 放到 ``0x8020_0000``，并确保 ``.text.entry``（也就是 ``_start``）在最前面。

把用户态逻辑“迁移”到裸机
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

``rust_main`` 里做的事情和上一节很像：输出字符串，然后退出。
区别在于：

- **输出**：不再是 ``write`` 系统调用，而是对 ``console_putchar`` 的循环调用；
- **退出**：不再是 ``exit`` 系统调用，而是 ``sbi::shutdown``。

.. code-block:: rust

   // rCore-Tutorial-in-single-workspace/ch1/src/main.rs（节选）
   extern "C" fn rust_main() -> ! {
       for c in b"Hello, world!\\n" {
           sbi::console_putchar(*c);
       }
       sbi::shutdown(false)
   }

可选：nobios 模式
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

如果使用：

.. code-block:: console

   $ cargo qemu --ch 1 --nobios

则 QEMU 不加载 RustSBI（``-bios none``），而是从 ``0x8000_0000`` 直接启动，
由课程代码提供的 M 态入口与最小 SBI 实现完成 M→S 的切换与服务。

至此，我们完成了第一章的实验内容，


.. note::

    背景知识：`理解应用程序和执行环境 <https://rcore-os.github.io/rCore-Tutorial-Book-v3/chapter1/4understand-prog.html>`_
