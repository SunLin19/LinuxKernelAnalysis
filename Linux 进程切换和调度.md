##分析 Linux 操作系统进程上下切换过程

###1.基本概念
我们知道，在操作系统运行过程中，有一种很常见的状态，就是从一个进程 `A` 切换到另一个进程 `B`。在很多操作系统原理的教科书中，花费了大量的篇幅介绍了进程调度算法，其实，这些算法仅仅是采用不同的策略从进程队列中选择一个新的进程而已。因此，学习操作系统的运行机制，掌握进程的调度时机与进程上下文切换机制更为关键。

###2.进程调度和上下文
一般来说，进程调度分为三种类型：
 - 中断处理过程（包括时钟中断、`I/O` 中断、系统调用和异常）中，直接调用`schedule`，或者返回用户态时根据 `need_resched` 标记调用 `schedule`；

 - 内核线程可以直接调用 `schedule` 进行进程切换，也可以在中断处理过程中进行调度，也就是说内核线程作为一类的特殊的进程可以**主动调度**，也可以**被动调度**；

 - 用户态进程无法实现主动调度，仅能通过陷入内核态后的某个时机点进行调度，即在中断处理过程中进行调度。

为了控制进程的执行，内核必须有能力挂起正在 `CPU` 上执行的进程，并恢复以前挂起的某个进程的执行的过程，叫做进程切换、任务切换、上下文切换。挂起正在 `CPU` 上执行的进程，与中断时保存现场是有区别的，中断前后是在同一个进程上下文中，只是由用户态转向内核态执行。

进程上下文信息：
 - 用户地址空间：包括程序代码，数据，用户堆栈等
 - 控制信息：进程描述符，内核堆栈等
 - 硬件上下文（注意中断也要保存硬件上下文只是保存的方法不同）
 - `schedule` 函数选择一个新的进程来运行，并调用 `context_switch` 宏进行上下文的切换，这个宏又调用 `switch_to` 宏来进行关键上下文切换
 - `switch_to` 宏中定义了 `prev` 和 `next` 两个参数：`prev` 指向当前进程，`next` 指向被调度的进程

###3.分析代码
```
static void __sched __schedule(void)
{
  struct task_struct *prev, *next;
  unsigned long *switch_count;
  struct rq *rq;
  int cpu;

need_resched:
  preempt_disable();
  cpu = smp_processor_id();
  rq = cpu_rq(cpu);
  rcu_note_context_switch(cpu);
  prev = rq->curr;

  schedule_debug(prev);

  if (sched_feat(HRTICK))
    hrtick_clear(rq);

  /*
   * Make sure that signal_pending_state()->signal_pending() below
   * can't be reordered with __set_current_state(TASK_INTERRUPTIBLE)
   * done by the caller to avoid the race with signal_wake_up().
   */
  smp_mb__before_spinlock();
  raw_spin_lock_irq(&rq->lock);

  switch_count = &prev->nivcsw;
  if (prev->state && !(preempt_count() & PREEMPT_ACTIVE)) {
    if (unlikely(signal_pending_state(prev->state, prev))) {
      prev->state = TASK_RUNNING;
    } else {
      deactivate_task(rq, prev, DEQUEUE_SLEEP);
      prev->on_rq = 0;

      /*
       * If a worker went to sleep, notify and ask workqueue
       * whether it wants to wake up a task to maintain
       * concurrency.
       */
      if (prev->flags & PF_WQ_WORKER) {
        struct task_struct *to_wakeup;

        to_wakeup = wq_worker_sleeping(prev, cpu);
        if (to_wakeup)
          try_to_wake_up_local(to_wakeup);
      }
    }
    switch_count = &prev->nvcsw;
  }

  if (task_on_rq_queued(prev) || rq->skip_clock_update < 0)
    update_rq_clock(rq);

  next = pick_next_task(rq, prev);
  clear_tsk_need_resched(prev);
  clear_preempt_need_resched();
  rq->skip_clock_update = 0;

  if (likely(prev != next)) {
    rq->nr_switches++;
    rq->curr = next;
    ++*switch_count;

    context_switch(rq, prev, next); /* unlocks the rq */
    /*
     * The context switch have flipped the stack from under us
     * and restored the local variables which were saved when
     * this task called schedule() in the past. prev == current
     * is still correct, but it can be moved to another cpu/rq.
     */
    cpu = smp_processor_id();
    rq = cpu_rq(cpu);
  } else
    raw_spin_unlock_irq(&rq->lock);

  post_schedule(rq);

  sched_preempt_enable_no_resched();
  if (need_resched())
    goto need_resched;
}
```

schedule 函数主要做了这么几件事：
>1、针对抢占的处理  
>2、raw_spin_lock_irq(&rq->lock)  
>3、检查prev的状态，并且重设state的状态  
>4、next = pick_next_task(rq, prev) //进程调度  
>5、更新就绪队列的时钟  
>6、context_switch(rq, prev, next) //进程上下文切换  

```
asm volatile("pushfl\n\t"  /* save    flags */ \
       "pushl %%ebp\n\t"  /* save    EBP   */ \
       "movl %%esp,%[prev_sp]\n\t" /* save    ESP   */ \
       "movl %[next_sp],%%esp\n\t" /* restore ESP   */ \
       "movl $1f,%[prev_ip]\n\t" /* save    EIP   */ \
       "pushl %[next_ip]\n\t" /* restore EIP   */ \
       "jmp __switch_to\n" /* regparm call  */ \
       "1:\t"      \
       "popl %%ebp\n\t"  /* restore EBP   */ \
       "popfl\n"   /* restore flags */ \
         \
       /* output parameters */                       \
       : [prev_sp] "=m" (prev->thread.sp),  \
       /* =m 表示把变量放入内存，即把 [prev_sp] 存储的变量放入内存，最后再写入prev->thread.sp */\
         [prev_ip] "=m" (prev->thread.ip),  \
         "=a" (last),                                           \
         /*=a 表示把变量 last 放入 ax, eax = last */         \
         \
         /* clobbered output registers: */  \
         "=b" (ebx), "=c" (ecx), "=d" (edx),  \
         /* b 表示放入ebx, c 表示放入 ecx，d 表示放入 edx, S表示放入 si, D 表示放入 edi */\
         "=S" (esi), "=D" (edi)    \
                \
         /* input parameters: */    \
       : [next_sp]  "m" (next->thread.sp),  \
       /* next->thread.sp 放入内存中的 [next_sp] */\
         [next_ip]  "m" (next->thread.ip),  \
                \
         /* regparm parameters for __switch_to (): */ \
         [prev]     "a" (prev),    \
         /*eax = prev  edx = next*/\
         [next]     "d" (next)    \
         \
       : /* reloaded segment registers */   \
       "memory");
```

`switch_to` 是 `A` 进程到 `B` 进程的过渡，我们可以认为在 `switch_to` 这个点上，`A` 进程被切出，`B` 进程被切入。进入 `switch_to` 的宏里面之后，首先 `pushfl` 和 `pushl` `ebp` 肯定仍然属于进程 `A`，之后把 `esp` 指向了 `B` 的堆栈，严格的说，从此时开始的指令流都属于 `B` 进程了。但是，这个时候 `B` 进程还没有完全准备好继续运行，因为 `ebp`、硬件上下文等内容还没有切换成 `B` 的，剩下的部分宏代码就是完成这些事情。也就是说，对于 `A` 进程它始终没有感觉到自己被打断过，它认为自己一直是不间断执行的。`switch_to` 这条“指令”，除了改变了 `A` 进程中的 `prev` 变量外，对 `A` 没有其它任何影响。在系统中任何进程看到的都是这个样子，所有进程都认为自己在不间断的独立运行。然而，实际上 `switch_to` 的执行并不是一瞬间完成的，`switch_to` 执行花了很长很长的时间，但是，在执行完 `switch_to` 之后，这段时间被从 `A` 的记忆中抹除，所以 `A` 并没有觉察到自己被打断过。

###4.总结
** 一般情形：**
正在运行的用户态进程 `A` 切换到运行用户态进程 `B` 的过程：
>1、正在运行的用户态进程 A  
>2、中断——save cs:eip/esp/eflags(current) to kernel stack，and load cs:eip(entry of a specific ISR) and ss:esp(point to kernel stack)  
>3、SAVE_ALL //保存现场  
>4、中断处理或中断返回前调用 schedule，其中，switch_to 做了关键的进程上下文切换  
>5、标号1之后开始运行用户态进程 B  
>6、restore_all //恢复现场  
>7、iret——pop cs:eip/ss:esp/eflags from kernel stack  
>8、继续运行用户态进程 B  

** 特殊情况：**
>1、通过中断处理过程中的调度，用户态进程与内核进程之间互相切换，与一般情形类似  
>2、内核进程程主动调用 schedule 函数，只有进程上下文的切换，没有中断上下文切换  
>3、创建子进程的系统调用在子进程中的执行起点及返回用户态，如：fork  
>4、加载一个新的可执行程序后返回到用户态的情况，如：execve  
