.. _term-print-userminienv:

构建用户态执行环境
=================================



用户态最小化执行环境
----------------------------

在用户态（Linux / QEMU User-Mode）下，程序的“执行环境”由操作系统提供：加载 ELF、建立用户栈、初始化寄存器等。
但 **no_std** 程序仍然需要自己补齐一些最基本的东西：

- **入口**：从哪里开始执行（``_start``）
- **退出**：怎么正常结束（``exit`` 系统调用）
- **输出**：怎么把字符写到终端（``write`` 系统调用）

入口：``_start`` 做了什么
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

用户态程序不能依赖 ``std`` 提供的启动流程，因此课程代码为用户程序提供了一个统一入口 ``_start``：

.. code-block:: rust

  // rCore-Tutorial-in-single-workspace/user/src/lib.rs（节选）
  #[no_mangle]
  #[link_section = ".text.entry"]
  pub extern "C" fn _start() -> ! {
      // 初始化输出与日志
      rcore_console::init_console(&Console);
      rcore_console::set_log_level(option_env!("LOG"));
      // 初始化堆（供 alloc 使用）
      heap::init();

      // 约定：用户程序提供 main()，返回 i32 作为退出码
      extern "C" {
          fn main() -> i32;
      }
      exit(unsafe { main() });
      unreachable!()
  }

你可以把它理解为用户态“最小运行时”的骨架：把控制权从 ``_start`` 交给用户程序 ``main``，
再把 ``main`` 的返回值交给 ``exit`` 系统调用，保证程序能**正常退出**，而不是“ret 掉以后崩溃”。

系统调用：把 ``ecall`` 封装成函数
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

课程代码把 Linux 风格的系统调用封装在 ``syscall`` 模块中。以 ``write`` 与 ``exit`` 为例：

.. code-block:: rust

  // rCore-Tutorial-in-single-workspace/syscall/src/user.rs（节选）
  pub fn write(fd: usize, buffer: &[u8]) -> isize {
      unsafe { syscall3(SyscallId::WRITE, fd, buffer.as_ptr() as _, buffer.len()) }
  }

  pub fn exit(exit_code: i32) -> isize {
      unsafe { syscall1(SyscallId::EXIT, exit_code as _) }
  }

进一步往下，你会看到最底层的 ``native::syscallN`` 用内联汇编发出 ``ecall``：

.. code-block:: rust

  // rCore-Tutorial-in-single-workspace/syscall/src/user.rs（节选）
  pub mod native {
      use crate::SyscallId;
      use core::arch::asm;

      pub unsafe fn syscall1(id: SyscallId, a0: usize) -> isize {
          let ret: isize;
          asm!("ecall",
              inlateout("a0") a0 => ret,
              in("a7") id.0,
          );
          ret
      }
  }

输出：把 Console 绑定到 ``write``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

用户库通过实现 ``rcore_console::Console`` trait，把“输出一个字符/字符串”的抽象落到 ``write`` 系统调用上：

.. code-block:: rust

  // rCore-Tutorial-in-single-workspace/user/src/lib.rs（节选）
  struct Console;

  impl rcore_console::Console for Console {
      fn put_char(&self, c: u8) {
          syscall::write(STDOUT, &[c]);
      }

      fn put_str(&self, s: &str) {
          syscall::write(STDOUT, s.as_bytes());
      }
  }

这样，用户程序就可以使用用户库导出的 ``print/println``（见 ``user/src/lib.rs`` 顶部的 ``pub use``），
而无需自己反复处理 ``write`` 的细节。

你也可以直接在用户库里看到它把 ``println`` 暴露给用户程序：

.. code-block:: rust

  // rCore-Tutorial-in-single-workspace/user/src/lib.rs（节选）
  pub use rcore_console::{print, println};
  pub use syscall::*;

动手运行：用 QEMU User-Mode 执行用户程序
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

代码框架在 ``user/src/bin`` 下提供了一组示例用户程序，例如 ``00hello_world.rs``。
你可以把它编译成 RISC-V 的可执行文件，并用 ``qemu-riscv64`` 在当前主机上运行：

.. code-block:: console

  $ cd rCore-Tutorial-in-single-workspace
  $ cargo build -p user_lib --bin 00hello_world --target riscv64gc-unknown-none-elf
  $ qemu-riscv64 target/riscv64gc-unknown-none-elf/debug/00hello_world; echo $?
  Hello, world!
  0

其中：

- ``-p user_lib``：选择要编译的包（课程里用户库包名叫 ``user_lib``）；
- ``--bin 00hello_world``：选择要编译的用户程序；
- ``--target ...``：编译成 RISC-V 的用户态可执行文件。

.. tip::

  读到这里，你应该能回答两个问题：

  - 用户程序是如何从 ``_start`` 开始执行并最终 ``exit`` 的？
  - ``println!`` 最后为什么会落到 ``write`` 这个系统调用上？

  接下来我们把这个思路迁移到裸机内核：此时 ``ecall`` 不再进入 Linux，而是进入 SBI。
