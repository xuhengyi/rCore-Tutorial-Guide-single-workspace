引言
=====================

本章导读
--------------------------

大多数程序员的职业生涯都从 ``Hello, world!`` 开始。

.. code-block::

   printf("Hello world!\n");
   cout << "Hello world!\n";
   print("Hello world!")
   System.out.println("Hello world!");
   echo "Hello world!"
   println!("Hello world!");

然而，要用几行代码向世界问好，并不像表面上那么简单。
``Hello, world!`` 程序能够编译运行，靠的是以 **编译器** 为主的开发环境和以 **操作系统** 为主的执行环境。

在本章中，我们将抽丝剥茧，一步步让 ``Hello, world!`` 程序脱离其依赖的执行环境，
编写一个能打印 ``Hello, world!`` 的 OS。这趟旅途将让我们对应用程序及其执行环境有更深入的理解。

.. attention::
   实验指导书存在的目的是帮助读者理解框架代码。

   为便于测试,完成编程实验时,请以框架代码为基础,不必跟着文档从零开始编写内核。

实践体验
-------------------------------------

本章一步步实现了支持打印字符串的简单特权态裸机应用程序。

在 ``rCore-Tutorial-in-single-workspace`` 项目中运行本章代码:

.. code-block:: console

   $ cargo qemu --ch 1

运行后,将看到以下输出:

.. code-block::

   [rustsbi] RustSBI version 0.3.1, adapting to RISC-V SBI v1.0.0
   .______       __    __      _______.___________.  _______..______   __
   |   _  \     |  |  |  |    /       |           | /       ||   _  \ |  |
   |  |_)  |    |  |  |  |   |   (----`---|  |----`|   (----`|  |_)  ||  |
   |      /     |  |  |  |    \   \       |  |      \   \    |   _  < |  |
   |  |\  \----.|  `--'  |.----)   |      |  |  .----)   |   |  |_)  ||  |
   | _| `._____| \______/ |_______/       |__|  |_______/    |______/ |__|
   [rustsbi] Implementation     : RustSBI-QEMU Version 0.2.0-alpha.2
   [rustsbi] Platform Name      : riscv-virtio,qemu
   [rustsbi] Platform SMP       : 1
   [rustsbi] Platform Memory    : 0x80000000..0x84000000
   [rustsbi] Boot HART          : 0
   [rustsbi] Device Tree Region : 0x83e00000..0x83e010f6
   [rustsbi] Firmware Address   : 0x80000000
   [rustsbi] Supervisor Address : 0x80200000
   [rustsbi] pmp01: 0x00000000..0x80000000 (-wr)
   [rustsbi] pmp02: 0x80000000..0x80200000 (---)
   [rustsbi] pmp03: 0x80200000..0x84000000 (xwr)
   [rustsbi] pmp04: 0x84000000..0x00000000 (-wr)

   Hello, world!

然后程序会正常退出。

本章代码树
---------------------------------------------

.. code-block::

   ├── ch1(内核实现)
   │   ├── Cargo.toml(配置文件,定义 nobios feature)
   │   ├── build.rs(构建脚本,根据 feature 生成不同的链接脚本)
   │   └── src(内核源代码)
   │       ├── main.rs(内核主函数,包含汇编入口和 rust_main)
   │       ├── sbi.rs(SBI 服务调用封装)
   │       ├── msbi.rs(M-Mode SBI 实现,nobios 模式)
   │       ├── m_entry_rv64.asm(M-Mode 入口汇编,nobios RV64)
   │       └── m_entry_rv32.asm(M-Mode 入口汇编,nobios RV32)

   cloc ch1
   -------------------------------------------------------------------------------
   Language                     files          blank        comment           code
   -------------------------------------------------------------------------------
   Rust                             4             45             41            240
   Assembly                         2             32             32            140
   Markdown                         1             23              0             80
   TOML                             1              2              0              9
   -------------------------------------------------------------------------------
   SUM:                             8            102             73            469
   -------------------------------------------------------------------------------