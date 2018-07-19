## 分析 Linux 内核的启动过程

### 1.计算机的启动过程
我们日常使用的计算机是怎么启动的，嗯，在我 9 岁那年第一次摸电脑的时候就有过这样的疑问。幸运的是，我现在找到了答案。今天就带着大家一起来分析分析计算机的启动和操作系统初始化的过程。

对于常见的 `x86` 计算机来说，通电后的第一个步骤就是启动一组固化到主板 `ROM` 芯片上的基本输入输出程序 `BIOS`，`BIOS` 在加电自检确认硬件完好无损后，就会开始寻找操作系统引导程序 `BootLoader`，这个引导一般位于硬盘的第一个扇区，`BootLoader` 就是负责操作系统初始化的程序了，先是哔哔哔运行一段汇编代码，然后，再跳转到 C 语言编写的 `start_kernel`，最后，开启死循环，运行操作系统。

### 2.启动内核
本次实验基于一个小型的操作系统内核 `MenuOS`，现在，让我们先来启动它看看：  

![](http://upload-images.jianshu.io/upload_images/1627862-b7b5544f1989872f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 3.GDB 调试内核
现在，我们再来使用断点调试工具，跟踪分析内核启动的过程：  

![](http://upload-images.jianshu.io/upload_images/1627862-4a1454c0c647b041.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们再 `start_kernel` 函数设置一个断点，通过 `list` 命令我们可以看到，`start_kernel`函数中有着非常多的函数调用，下面我们选择几个重要的有代表性的来分析分析。

![](http://upload-images.jianshu.io/upload_images/1627862-f7bd478e60f80435.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![](http://upload-images.jianshu.io/upload_images/1627862-013009435f4a895f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 4.重要函数分析
首先，我们来看看 `trap_init` 函数：  

![](http://upload-images.jianshu.io/upload_images/1627862-0634f94166056a60.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

中断向量表的初始化函数，可以看到它针对 `x86` 芯片，设置非常多的中断门 `Interrupt Gate`。

然后，我们来看看 `sched_init` 函数：  
进程调度初始化函数，在这个函数里面有一个很重要的步骤，即对 0 号进程的初始化。

![](http://upload-images.jianshu.io/upload_images/1627862-ede732a91104ebbf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

最后，我们来看看 `rest_init` 函数：
```
static noinline void __init_refok rest_init(void)
{
    int pid;

    rcu_scheduler_starting();
    /*
     * We need to spawn init first so that it obtains pid 1, however
     * the init task will end up wanting to create kthreads, which, if
     * we schedule it before we create kthreadd, will OOPS.
     */
    kernel_thread(kernel_init, NULL, CLONE_FS);
    numa_default_policy();
    pid = kernel_thread(kthreadd, NULL, CLONE_FS | CLONE_FILES);
    rcu_read_lock();
    kthreadd_task = find_task_by_pid_ns(pid, &init_pid_ns);
    rcu_read_unlock();
    complete(&kthreadd_done);

    /*
     * The boot idle thread must execute schedule()
     * at least once to get things moving:
     */
    init_idle_bootup_task(current);
    schedule_preempt_disabled();
    /* Call into cpu_idle with preempt disabled */
    cpu_startup_entry(CPUHP_ONLINE);
}
```
其他初始化函数，这个函数调用系统函数 `kernel_thread` 创建 1 号进程，即 init 进程，是用户态所有进程的祖先，然后，新建 `kthreadd` 进程，是内核态所有进程的祖先。最后，通过 `cpu_startup_entry` 函数启动 0 号进程。
