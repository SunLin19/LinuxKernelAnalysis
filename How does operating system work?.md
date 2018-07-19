##1.函数调用堆栈和中断
在上一节实验中我们已经明白了，存储程序计算机的运行原理，就是通过不断的取指执行存储在堆栈上 `CPU` 指令，而这些二进制指令一一对应于汇编函数指令。因此，对于操作系统，我们引入相应的函数调用堆栈机制。  

我们都知道 `CPU` 的运算速度是极其快的，很多时候例如输入输出过程，如果没有系统中断，`CPU` 就只能等待外设输入输出完成后再工作，这样会造成了极大的资源浪费。因此，现代操作系统都引入了中断机制，`CPU` 会根据当前进程的执行情况，交替执行多个程序，提高运行效率。  

在本次实验基于一个小小的时间片轮转多道程序 [mykernel](https://github.com/mengning/mykernel)，分析操作系统的函数调用堆栈和中断的实现过程。  


##2.内核代码分析
`mykernel` 程序有三个源文件，分别是 `mypcb.h`，`mymain.c` 和 `myinterrupt.c`，`mypcb.h` 头文件，定义了一些结构和函数。`mymain.c` 定义了多个进程启动和运行的函数。`myinerrupt.c` 定义了时间片和中断处理的函数。

####2.1>我们先来看看 `mypcb.h` 头文件:
```
#define MAX_TASK_NUM 10 // max num of task in system
#define KERNEL_STACK_SIZE 1024*8
#define PRIORITY_MAX 30 //priority range from 0 to 30
```
首先，定义了最大任务数量、内核栈大小和任务的优先级序列。
```
/* CPU-specific state of this task */
struct Thread {
    unsigned long	ip;//point to cpu run address
    unsigned long	sp;//point to the thread stack's top address
    
    //todo add other attrubte of system thread
};
```
然后，定义 `Thread` 结构体，`ip` 和 `sp` 分别表示执行指针和栈指针，用于执行和保护现场。 
```
//PCB Struct
typedef struct PCB{
    int pid; // pcb id 
    volatile long state;	/* -1 unrunnable, 0 runnable, >0 stopped */
    char stack[KERNEL_STACK_SIZE];// each pcb stack size is 1024*8
    
    /* CPU-specific state of this task */
    struct Thread thread;
    unsigned long	task_entry;//the task execute entry memory address
    struct PCB *next;//pcb is a circular linked list
    unsigned long priority;// task priority ////////
    
    //todo add other attrubte of process control block
}tPCB;
```
再然后，定义了进程管理结构体，包括：1、`pid` 进程号；2、`state` 进程状态；3、栈空间；4、`thread` 变量；5、任务入口地址；6、下一个 `PCB` 指针地址；7、任务的优先级。
```
//void my_schedule(int pid);
void my_schedule(void);
```
最后，定义了 `my_schedule` 任务调度函数。

####2.2>再来分析 `mymain.c` 文件：
```
tPCB task[MAX_TASK_NUM];
tPCB * my_current_task = NULL;
volatile int my_need_sched = 0;
```
首先，定义三个全局变量，一个是 `PCB` 数组，一个是指向当前任务的指针，`my_need_sched` 被来判断是否调用中断处理函数。
```
void __init my_start_kernel(void)
{
    int pid = 0;
   
    /* Initialize process 0*/
    task[pid].pid = pid;
    task[pid].state = 0;/* -1 unrunnable, 0 runnable, >0 stopped */
    
    // set task 0 execute entry address to my_process
    task[pid].task_entry = task[pid].thread.ip = (unsigned long)my_process;
    task[pid].thread.sp = (unsigned long)&task[pid].stack[KERNEL_STACK_SIZE-1];
    task[pid].next = &task[pid];
    
    /*fork more process */
    for(pid=1;pid<MAX_TASK_NUM;pid++)
    {
        memcpy(&task[pid],&task[0],sizeof(tPCB));
        task[pid].pid = pid;
        task[pid].state = -1;
        task[pid].thread.sp = (unsigned long)&task[pid].stack[KERNEL_STACK_SIZE-1];
   task[pid].priority=get_rand(PRIORITY_MAX);//each time all tasks get a random priority
    }
task[MAX_TASK_NUM-1].next=&task[0];
    printk(KERN_NOTICE "\n\n\n\n\n\n system begin :>>>process 0 running!!!<<<\n\n");
    
    /* start process 0 by task[0] */
    pid = 0;
    my_current_task = &task[pid];
    asm volatile(
         "movl %1,%%esp\n\t" /* set task[pid].thread.sp to esp */
         "pushl %1\n\t" /* push ebp */
         "pushl %0\n\t" /* push task[pid].thread.ip */
         "ret\n\t" /* pop task[pid].thread.ip to eip */
         "popl %%ebp\n\t"
         :
         : "c" (task[pid].thread.ip),"d" (task[pid].thread.sp) /* input c or d mean %ecx/%edx*/
    );
}
```
`my_start_kernel` 函数非常重要，首先初始化任务 0，设置为可运行状态，设置任务运行的 `ip`，设置任务运行 `sp` 空间，设置下一个 `PCB` 指向自己。然后，设置余下的任务并随机分配优先权(我们设置最大任务数为 10)。最后，通过内联汇编把 `sp` 赋给 `esp`，`ebp` 进栈，把 `ip` 保存到栈中，这样如果任务切换的话就可以保证下一个进程结束后回到现场继续执行。

```
void my_process(void)
{
    int i = 0;
    while(1)
    {
        i++;
        if(i%10000000 == 0)
        {
       if(my_need_sched == 1)
            {
                my_need_sched = 0;
       sand_priority();
         my_schedule();
       }
        }
    }
}//end of my_process
```
`my_process` 函数很简单，就是建立一个循环不断运行进程，输出进程信息，同时，如果 `my_need_sched = 1`，就开始中断并切换进程。

####2.3>最后分析 myinterrupt.c 文件：
```
extern tPCB task[MAX_TASK_NUM];
extern tPCB * my_current_task;
extern volatile int my_need_sched;
volatile int time_count = 0;

/*
* Called by timer interrupt.
* it runs in the name of current running process,
* so it use kernel stack of current running process
*/
void my_timer_handler(void)
{
#if 1
    // make sure need schedule after system circle 2000 times.
    if(time_count%2000 == 0 && my_need_sched != 1)
    {
        my_need_sched = 1;
   //time_count=0;
    }
    time_count++ ;
#endif
    return;
}
```
`my_timer_handler` 函数也非常简单，通过判断 `time_count` 时间片变量的阈值，更改 `my_need_sched` 值实现了中断调用。

```
void my_schedule(void)
{
    tPCB * next;
    tPCB * prev;
    // if there no task running or only a task ,it shouldn't need schedule
    if(my_current_task == NULL || my_current_task->next == NULL)
    {
printk(KERN_NOTICE " time out!!!,but no more than 2 task,need not schedule\n");
     return;
    }
    /* schedule */

    next = get_next();
    prev = my_current_task;
    printk(KERN_NOTICE "                the next task is %d priority is %u\n",next->pid,next->priority);
    if(next->state == 0)/* -1 unrunnable, 0 runnable, >0 stopped */
    {//save current scene
     /* switch to next process */
     asm volatile(
         "pushl %%ebp\n\t" /* save ebp */
         "movl %%esp,%0\n\t" /* save esp */
         "movl %2,%%esp\n\t" /* restore esp */
         "movl $1f,%1\n\t" /* save eip */
         "pushl %3\n\t"
         "ret\n\t" /* restore eip */
         "1:\t" /* next process start here */
         "popl %%ebp\n\t"
         : "=m" (prev->thread.sp),"=m" (prev->thread.ip)
         : "m" (next->thread.sp),"m" (next->thread.ip)
     );
     my_current_task = next;//switch to the next task
    printk(KERN_NOTICE "switch from %d process to %d process\n >>>process %d running!!!<<<\n\n",prev->pid,next->pid,next->pid);

  }
    else
    {
        next->state = 0;
        my_current_task = next;
    printk(KERN_NOTICE "switch from %d process to %d process\n >>>process %d running!!!<<<\n\n\n",prev->pid,next->pid,next->pid);

     /* switch to new process */
     asm volatile(
         "pushl %%ebp\n\t" /* save ebp */
         "movl %%esp,%0\n\t" /* save esp */
         "movl %2,%%esp\n\t" /* restore esp */
         "movl %2,%%ebp\n\t" /* restore ebp */
         "movl $1f,%1\n\t" /* save eip */
         "pushl %3\n\t"
         "ret\n\t" /* restore eip */
         : "=m" (prev->thread.sp),"=m" (prev->thread.ip)
         : "m" (next->thread.sp),"m" (next->thread.ip)
     );
    }
    return;
}//end of my_schedule
```
`my_schedule` 函数是这个内核的重点，首先初始化 `next` 和 `prev` 两个 `PCB` 结构体，然后判断如果任务 `state` 状态是可运行时，说明这个任务正在执行，保存 `ebp` 和 `esp`，并切换到下一个任务 `ip` 执行；如果任务 `state` 状态是不可运行时，说明这个任务没执行过，保存当前任务，开始执行新任务。

###3.总结：
通过分析实验代码，我们学习了一个简单的时间片轮转多道操作系统内核，了解了操作系统的中断上下文和进程上下文切换。每个任务被分配一定的时间片执行，如果在时间片结束后任务仍在执行，CPU 将会剥夺它的执行权并分配给其他任务；如果任务在时间片结束前完成，CPU 则会立即进行切换，调度程序就是在维护一个就绪进程队列，当进程用完属于它的时间片后，在队列中重新按照优先级排序，这是最简单、最公平也最高效的一种方式。
