##分析 Linux 操作系统如何创建一个进程

###1.进程概述
在很久很久以前，我刚开始学习操作系统的时候，总是分不清进程和线程二者的区别。教科书上总是很抽象的定义进程是资源分配的最小单元，然后就是对代码段、数据段、堆栈以及进程状态的描述，从来没有讲过在操作系统中进程是如何被创建和销毁的。而所有的现代操作系统，都是以进程来作为任务的载体，因此，掌握进程的创建过程对分析操作系统内核十分重要。今天，我们就来深入的分析 Linux 中进程的创建过程。

进程控制块 PCB
作为存放进程的管理和控制信息的数据结构，每一个进程均有一个 `PCB`，在创建时建立，并伴随进程运行的全过程，直到进程撤消而撤消。

###2.进程创建过程
我们知道，Linux 系统中 `fork` 、`vfork` 、`clone`等函数都可以用来创建一个新的进程，那么，我们来看看对应的系统调用函数。  

```
SYSCALL_DEFINE0(fork) 
{ 
  return do_fork(SIGCHLD, 0, 0, NULL, NULL);
}

SYSCALL_DEFINE0(vfork)
{ 
  return do_fork(CLONE_VFORK | CLONE_VM | SIGCHLD, 0, 0, NULL, NULL);
}

SYSCALL_DEFINE5(clone, unsigned long, clone_flags, unsigned long, newsp, 
                int __user *, parent_tidptr, int __user *, child_tidptr, int, tls_val)
{
  return do_fork(clone_flags, newsp, 0, parent_tidptr, child_tidptr);
}
```
我把上述代码进行了部分裁剪，可以看出 `fork` 、`vfork` 、`clone` 等函数对应的系统调用，都是调用了 `do_fork` 函数实现的。现在，我们来分析 `do_fork` 函数：  

```
long do_fork(unsigned long clone_flags, unsigned long stack_start,
        unsigned long stack_size, int __user *parent_tidptr,
        int __user *child_tidptr)
{
  struct task_struct *p;
  int trace = 0;
  long nr;
  
  //复制进程描述符，返回创建的 task_struct 的指针
  p = copy_process(clone_flags, stack_start, stack_size,
       child_tidptr, NULL, trace);

  if (!IS_ERR(p)) {
    struct completion vfork;
    struct pid *pid;

    trace_sched_process_fork(current, p);

    //取出 task 结构体内的 pid
    pid = get_task_pid(p, PIDTYPE_PID);
    nr = pid_vnr(pid);

    if (clone_flags & CLONE_PARENT_SETTID)
      put_user(nr, parent_tidptr);

    //如果使用的是 vfork，那么必须采用某种完成机制，确保父进程后运行
    if (clone_flags & CLONE_VFORK) {
      p->vfork_done = &vfork;
      init_completion(&vfork);
      get_task_struct(p);
    }

    //将子进程添加到调度器的队列，使得子进程有机会获得 CPU
    wake_up_new_task(p);

    //如果设置了 CLONE_VFORK 则将父进程插入等待队列，并挂起父进程直到子进程释放自己的内存空间
    //保证子进程优先于父进程运行
    if (clone_flags & CLONE_VFORK) {
      if (!wait_for_vfork_done(p, &vfork))
        ptrace_event_pid(PTRACE_EVENT_VFORK_DONE, pid);
    }

    put_pid(pid);
  } else {
    nr = PTR_ERR(p);
  }
  return nr;
}
```
从上面可以看出，撇开对 `vfork` 函数实现处理不谈，`do_fork` 函数主要调用了 `copy_process` 函数来创建进程，下面我们继续来跟踪 `copy_process` 函数：  

```
/*
  创建进程描述符以及子进程所需要所有数据结构
*/
static struct task_struct *copy_process(unsigned long clone_flags,
          unsigned long stack_start,
          unsigned long stack_size,
          int __user *child_tidptr,
          struct pid *pid,
          int trace)
{
  int retval;
  struct task_struct *p;

  //分配一个新的 task_struct，此时的p与当前进程的 task，仅仅是 stack 地址不同
  p = dup_task_struct(current);

  //检查该用户的进程数是否超过限制
  if (atomic_read(&p->real_cred->user->processes) >=
      task_rlimit(p, RLIMIT_NPROC)) {
    //检查该用户是否具有相关权限，不一定是 root
    if (p->real_cred->user != INIT_USER &&
        !capable(CAP_SYS_RESOURCE) && !capable(CAP_SYS_ADMIN))
      goto bad_fork_free;
  }

  retval = -EAGAIN;
  //检查进程数量是否超过 max_threads，后者取决于内存的大小
  if (nr_threads >= max_threads)
    goto bad_fork_cleanup_count;

  //完成对新进程调度程序数据结构的初始化，并把新进程的状态设置为TASK_RUNNING
  retval = sched_fork(clone_flags, p);

  //初始化子进程的内核栈
  retval = copy_thread(clone_flags, stack_start, stack_size, p);
  if (retval)
    goto bad_fork_cleanup_io;

  if (pid != &init_struct_pid) {
    retval = -ENOMEM;
    //这里为子进程分配了新的 pid 号
    pid = alloc_pid(p->nsproxy->pid_ns_for_children);
    if (!pid)
      goto bad_fork_cleanup_io;
  }

  //设置子进程的 pid
  p->pid = pid_nr(pid);
  //如果是创建线程
  if (clone_flags & CLONE_THREAD) {
    p->exit_signal = -1;
    //线程组的 leader 设置为当前线程的 leader
    p->group_leader = current->group_leader;
    //tgid 是当前线程组的 id，也就是 main 进程的 pid
    p->tgid = current->tgid;
  } else {
    if (clone_flags & CLONE_PARENT)
      p->exit_signal = current->group_leader->exit_signal;
    else
      p->exit_signal = (clone_flags & CSIGNAL);
    //创建的是进程，自己是一个单独的线程组
    p->group_leader = p;
    //tgid 和 pid 相同
    p->tgid = p->pid;
  }

  if (clone_flags & (CLONE_PARENT|CLONE_THREAD)) {
    //如果是创建线程，那么同一线程组内的所有线程、进程共享 parent
    p->real_parent = current->real_parent;
    p->parent_exec_id = current->parent_exec_id;
  } else {
    //如果是创建进程，当前进程就是子进程的 parent
    p->real_parent = current;
    p->parent_exec_id = current->self_exec_id;
  }

  //将 pid 加入 PIDTYPE_PID 这个散列表
  attach_pid(p, PIDTYPE_PID);
  //递增 nr_threads 的值
  nr_threads++;

  //返回被创建的 task 结构体指针
  return p;
}
```

`copy_process` 函数的代码非常复杂，在这里我裁剪并注释了一部分代码，做个简单的总结，`copy_process` 函数为了创建子进程，主要做了这么几件事：  

 - 调用 dup_task_struct 复制当前的 task_struct；
 - 调用 sched_fork 初始化进程数据结构，并把进程状态设置为 TASK_RUNNING；
 - 复制父进程的所有信息；
 - 调用 copy_thread 初始化子进程内核栈；
 - 为新进程分配并设置新的 pid；

最后，我们来看看 `copy_thread` 是怎样为子进程创建好上下文堆栈的：

```
// 初始化子进程的内核栈
int copy_thread(unsigned long clone_flags, unsigned long sp,
  unsigned long arg, struct task_struct *p)
{

  // 获取寄存器信息
  struct pt_regs *childregs = task_pt_regs(p);
  struct task_struct *tsk;
  int err;

  // 栈顶 空栈
  p->thread.sp = (unsigned long) childregs;
  p->thread.sp0 = (unsigned long) (childregs+1);
  memset(p->thread.ptrace_bps, 0, sizeof(p->thread.ptrace_bps));

  // 如果是创建的内核线程
  if (unlikely(p->flags & PF_KTHREAD)) {
    /* kernel thread */
    memset(childregs, 0, sizeof(struct pt_regs));
    // 内核线程开始执行的位置
    p->thread.ip = (unsigned long) ret_from_kernel_thread;
    task_user_gs(p) = __KERNEL_STACK_CANARY;
    childregs->ds = __USER_DS;
    childregs->es = __USER_DS;
    childregs->fs = __KERNEL_PERCPU;
    childregs->bx = sp; /* function */
    childregs->bp = arg;
    childregs->orig_ax = -1;
    childregs->cs = __KERNEL_CS | get_kernel_rpl();
    childregs->flags = X86_EFLAGS_IF | X86_EFLAGS_FIXED;
    p->thread.io_bitmap_ptr = NULL;
    return 0;
  }

  // 将当前进程的寄存器信息复制给子进程
  *childregs = *current_pt_regs();
  // 子进程的eax置为0，所以fork的子进程返回值为0
  childregs->ax = 0;
  if (sp)
    childregs->sp = sp;

  // 子进程从ret_from_fork开始执行
  p->thread.ip = (unsigned long) ret_from_fork;
  task_user_gs(p) = get_user_gs(current_pt_regs());

  return err;
}
```

`copy_thread` 解释了两个非常重要的问题：
> 1、为什么 fork 在子进程中返回 0，原因是 childregs->ax = 0 这句，把子进程的 eax 赋值为 0  
> 2、p->thread.ip = (unsigned long) ret_from_fork， 将子进程的 ip 设置为 ret_form_fork 的首地址，因此子进程是从 ret_from_fork 开始执行的  

###3.总结
新进层的创建过程，可以被分为如下图断点所示的几个重要阶段：  

![](http://upload-images.jianshu.io/upload_images/1627862-499677289c192929.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

 - do_fork 系统内核调用；
 - copy_process 复制父进程的所有信息给子进程；
 - dup_task_struct 中为子进程分配了新的堆栈；
 - 调用 sched_fork，并把新进程设置为 TASK_RUNNING；
 - copy_thread 中把父进程的寄存器上下文复制给子进程；
 - 设置 ret_from_fork 的地址为 eip 寄存器的值。
