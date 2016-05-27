
#Lab4 实验报告

## 理解和填写代码

首先需要阅读的是众多之前没有用到的头文件。

#### `include/unistd.h`

```c
#define __SYSCALL_BASE 9527
#define SYS_putchar ((__SYSCALL_BASE ) + (0 ) )
```

可以看到这个文件中定义了众多`SYS_xxx`的数值，初步的猜想这应该是某些地址或者某些函数的参数。

#### `user/syscall_wrap.S`

这里定义了`msyscall`函数，但是可以看到调用时它会接受 6 个参数，根据注释，这些参数前 4 个保存在 a0 ~ a3 寄存器中，后面的 2 个保存在栈中。但是

关于`syscall`指令，查阅 *See MIPS Run* 可以看到：

> For the MIPS architecture that’s done witha software-triggered exception from the syscall instruction, which arrives inthe kernel exception handler with a distinctive code (8 or “Sys”) in the CPU’s Cause(ExcCode) register field.
>
> - The syscall number is put in v0.
> - Arguments are passed as required by the “o32” ABI.

那么这个 ABI 是什么呢，根据书中的指示，查阅 11.2.1 可以看到：


> All arguments are allocated space in a data structure on the stack, but the contents thatbelong to the first few stack locations are in fact passed in CPU registers—thecorresponding memory locations are left undefined.
>

也就是说，此时`msyscall`是在向`syscall`指令传递参数。那么`syscall`指令到底会做什么事情呢？

> the syscall instruction, which arrives inthe kernel exception handler with a distinctive code (8 or “Sys”) in the CPU’s Cause(ExcCode) register field. The lowest-level exception handler will con-sult that field to decide what to do next and will switch into the kernel’s systemcall handler.

可见，`syscall`指令会产生一个中断，并把中断号设置为 8，此后的工作交由中断处理程序来完成。那么我们的中断处理程序是什么呢？

#### `lib/traps.c`

在这个文件中可以看到 8 号中断对应的是`handle_sys()`函数，而这个函数在哪里定义呢？

#### `lib/syscall.S`

```c
la t1, sys_call_table
addu t1, t0
lw t2, (t1)
//...
jalr t2
```

可以看到，`handle_sys()`函数计算了对应系统调用的处理函数的地址，并跳转到了该地址。而根据文件最后定义的宏和变量名，这些地址应该就是`syscall_all.c`中定义的函数了。

#### `lib/syscall_all.c`

在这里可以看到真正的系统调用服务函数了，需要填写的也是这些函数。

## 研究思考题

#### Thinking 4.1 不同的进程代码执行

子进程按照父进程的代码来执行，说明子进程的代码段指向了父进程的代码段。没有执行`fork()`之前的代码，说明操作系统调整了子进程的 PC 值。

#### Thinking 4.2 `fork`的返回结果

注意到定义在`trap.h`中的“异常帧”，也就是进程开始运行时加载入 CPU 的上下文。其中`regs[32]`应该保存的是 32 个通用寄存器的值，那么哪一个是返回值呢？再次查阅 *See MIPS Run*：

> An integer or pointer return value will be in register v0 ($2).

那么我们只需将`regs[2]`设置为 0，即可实现双返回值。

#### Thinking 4.3 用户空间的保护

我认为只需要保护可写的页，也就是权限为中有 `PTE_R` 的页。

#### Thinking 4.4 `vpt`宏的使用

这两个宏是为了在用户空间访问当前的页目录和页表。在实验二中，可以直接使用 `boot_pgdir` 等数组。


### 难度评估

这次实验我觉得比较难，主要是有前几次实验的遗留问题，一直到实验课上才解决。总共用了大约 15 小时，主要时间都用在填写代码和调试上。