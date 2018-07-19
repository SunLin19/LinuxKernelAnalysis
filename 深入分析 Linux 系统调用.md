## 分析 Linux 中断处理过程

在上一周的实验课程中，我们分析了 [Linux 内核系统调用过程](http://www.jianshu.com/p/0e25bee35c66)，理解了中断的概念和中断上下文，掌握了系统调用的原理，今天，我们继续以 Linux 内核 2 号系统调用 fork 函数为例，更加深入的分析系统调用过程。

### 1.增加 Menu 内核命令行
在这里，我们把上一次实验的 `fork` 函数以命令行的形式写入内核，在这里我就不赘述操作步骤了，之前的实验有详细的操作流程。  

![](http://upload-images.jianshu.io/upload_images/1627862-00d9a712d8b4ab0c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这就是我们将 `fork` 函数写入 `Menu` 系统内核后的效果，通过命令行，实现了操作系统调用过程。

### 2.GDB 追踪内核调用 sys_fork
通过查询操作系统内核调用函数 API，我们知道 `fork` 函数的系统调用是 sys_fork，下面我们就通过 `GDB` 来追踪 `sys_fork` 的调用过程。同样，这里我也不再赘述操作过程了，之前的实验都有涉及。  

![](http://upload-images.jianshu.io/upload_images/1627862-3b4351a25a059d49.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)  

由上图可知，`sys_fork` 在底层调用的是 `do_fork` 函数。这里需要注意一下，`GDB` 支持在 `system_call` 处设置断点，但却无法让程序停留在这个地方，因此，我们只能暂停在这里了，转向分析源码。

### 3.分析内核调用汇编源码

```
ENTRY(system_call)
	RING0_INT_FRAME		# can't unwind into user space anyway
	ASM_CLAC
	pushl_cfi %eax			# save orig_eax
	SAVE_ALL      #保存现场
	GET_THREAD_INFO(%ebp)	# system call tracing in operation / emulation
	testl $_TIF_WORK_SYSCALL_ENTRY,TI_flags(%ebp)
	jnz syscall_trace_entry
	cmpl $(NR_syscalls), %eax
	jae syscall_badsys

syscall_call:
	call *sys_call_table(,%eax,4)    #系统调用

syscall_after_call:
	movl %eax,PT_EAX(%esp)		# store the return value

syscall_exit:
	LOCKDEP_SYS_EXIT
	DISABLE_INTERRUPTS(CLBR_ANY)	# make sure we don't miss an interrupt
									# setting need_resched or sigpending
									# between sampling and the iret
	TRACE_IRQS_OFF
	movl TI_flags(%ebp), %ecx
	testl $_TIF_ALLWORK_MASK, %ecx	# current->work
	jne syscall_exit_work

restore_all:
	TRACE_IRQS_IRET    #恢复现场

irq_return:
	INTERRUPT_RETURN    #中断返回
```
为了方便阅读，我把上述系统调用源码整理了一下，关键的地方被我打上了注释。总的来说，系统调用过程并不复杂。值得注意的是，在退出系统调用之前，会检查当前进程是否需要执行 `syscall_exit_work`，如果需要处理，就会查看是否有进程信号或进程调度，若有进程信号或进程调度就再进行相应操作，这不是本次实验的重点，不再展开。据此，我们可以画出从系统调用到中断返回的函数流程图：  

![](http://upload-images.jianshu.io/upload_images/1627862-049ddde62865145a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 4.总结
本实验通过分析 Linux 内核系统调用源码，深入剖析了系统调用的全过程。通过 `int 0x80` 中断跳转到系统调用处理程序 `system_call` 函数处，执行相应的例程。

但是，由于是代表的是用户进程，所以这个执行过程并不属于中断上下文，而是进程上下文。因此，系统调用执行过程中，可以访问用户进程的许多信息，也可以被更高优先级的进程抢占。

当系统调用完成后，把控制权交回到发起调用的用户进程前，内核又会有一次调度。如果发现有优先级更高的进程或当前进程的时间片用完，那么会选择优先级更高的进程或重新选择进程执行。
