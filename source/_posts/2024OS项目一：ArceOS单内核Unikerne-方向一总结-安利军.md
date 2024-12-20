---
title: '2024OS项目一：ArceOS单内核Unikerne,方向一总结-安利军'
date: 2024-12-19 12:58:55
categories:
    - 2024OS项目一
tags:
    - author:<Ansen88>
---

# 了解了`libos` 的概念
以前学习的都是传统的内核态和用户态这种方式的方式的系统，即使在嵌入式系统中也是使用了 `syscall` 来进行用户态和内核态的交互。
`libos` 颠覆了这种传统的操作系统的概念，使得在思想上跳出传统操作系统的束缚，这使得在一些应用场景中能够更好的发挥硬件的性能。
对 `libos` 进行了一些调研：[《libos 介绍》](https://blog.csdn.net/m0_37749564/article/details/144233693)

# 分析了 `arceos` 的启动过程

要在 `arceos` 上作更改，则需要深入了解该系统，所以对该系统的启动过程进行了简单的分析。

相关文档：[arceos分析](https://blog.csdn.net/m0_37749564/article/details/144233925)

# 熟悉了 `libos` 实现用函数调用替代系统调用的方案
系统调用从硬件的层面给用户调用内核功能提供了一个标准的接口；而由于软件代码的地址是编译时产生的，所以要想通过软件提供给用户代码一个标准接口就是一个挑战。以前只是考虑使用链接文件来固定这个接口；但这次训练营提供了另外一个思路来提供这么一个标准接口 —— 通过寄存器/栈来给用户程序建立一个 `ABI` 的环境变量，以便库函数可以获取并使用该标准接口。
为此，对 `musl` 库进行了一些分析：[C 语言运行分析](https://blog.csdn.net/m0_37749564/article/details/144446969)

# 对 `arceos` 支持多进程提出了几点想法
大体思路就是通过内核代码和用户代码的关系来组成不同的方案。
朱懿老师对我的想法给出了几点意见。
相关文档：[Arceos 多进程支持思路](https://blog.csdn.net/m0_37749564/article/details/144516395) 

# 进展

目前通过修改 `musl` 库的 `crt1.c` 文件：

```c
#include "crt_arch.h"

int main();
weak void _init();
weak void _fini();
int __libc_start_main(int (*)(), int, char **,
	void (*)(), void(*)(), void(*)());

unsigned long volatile abi_table = 0;

hidden void _start_c(long *p)
{
	abi_table = *p;
	p++;
	int argc = p[0];
	char **argv = (void *)(p+1);
	// __libc_start_main(main, argc, argv, _init, _fini, 0);
	main(argc, argv, 0);
}
```

将修改后的 `crt1.o` 文件替代 `riscv64-linux-musl-gcc` 中的 `crt1.o` 后，重新编译 `main.c` 可以正常运行。

`main.c`

```c
#include "puts.h"

int main(int argc, char**argv)
{
//	hello();
	return 0;
}

```

运行结果：

```shell
... ...
Load payload ok!
Execute app ...
QEMU: Terminated
```

需要更改 `bin` 文件的运行地址。
