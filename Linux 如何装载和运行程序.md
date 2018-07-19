##分析 Linux 操作系统如何装载链接并执行程序

###1.理论基础
我们在本科刚开始学习 `C` 语言的时候，老师们都会讲 `C` 语言程序的执行必须经历预处理、编译、汇编、链接、执行程序等过程。由于那时候教学大多使用 `VC6` 集成开发环境，所以对于上述过程并没有很清晰的概念。今天，我们就通过 `GDB` 来跟踪分析一个 `execve` 系统调用内核处理函数 `sys_execve`，深入理解 `Linux` 操作系统装载链接和运行可执行程序的过程。

我们先以 `hello_world.c` 程序为例，搞清楚可执行程序是如何生成的：
```
#include <stdio.h>
int main(void)
{
    printf("hello, world!\n");
    return 0;
}
```
1.预处理，处理代码中的宏定义和 `include` 文件，并做语法检查
> gcc -E hello_world.c -o hello_world.i

2.编译，生成汇编代码
>gcc -S hello_world.i -o hello_world.s

3.汇编，生成 `ELF` 格式的目标代码
>gcc -c hello_world.s -o hello_world.o

4.链接，生成可执行代码
>gcc hello_world.o -o hello_world

5.执行程序
>./hello_world
>hello, world!

我们可以对这个过程做一个总结，如下图：

![](http://upload-images.jianshu.io/upload_images/1627862-5b61d2cb1f492d50.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

###2.ELF 文件格式
`ELF` 格式：[可执行和可链接格式](http://en.wikipedia.org/wiki/Executable_and_Linkable_Format) (`Executable and Linkable Format`) 是一种用于二进制文件、可执行文件、目标代码、共享库和核心转储的标准文件格式。
>可重定位文件，如：.o 文件，包含代码和数据，可以被链接成可执行文件或共享目标文件，静态链接库属于这一类。  
>可执行文件，如：/bin/bash 文件，包含可直接执行的程序，没有扩展名。  
>共享目标文件，如：.so 文件，包含代码和数据，可以跟其他可重定位文件和共享目标文件链接产生新的目标文件，也可以跟可执行文件结合作为进程映像的一部分。  

`ELF` 文件由 `ELF header` 和文件数据组成，文件数据包括:
>Program header table, 程序头：描述段信息  
>.text, 代码段：保存编译后得到的指令数据  
>.data, 数据段：保存已经初始化的全局静态变量和局部静态变量  
>Section header table, 节头表：链接与重定位需要的数据  

![](http://upload-images.jianshu.io/upload_images/1627862-3b2f80455bd8a193.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

###3.静态&动态链接
链接，是收集和组织程序所需的不同代码和数据的过程，以便程序能被装入内存并被执行。一般分为两步：1.空间与地址分配，2.符号解析与重定位。一般有两种类型，一是静态链接，二是动态链接。

使用静态链接的好处是，依赖的动态链接库较少（这句话有点绕），对动态链接库的版本更新不会很敏感，具有较好的兼容性；不好地方主要是生成的程序比较大，占用资源多。使用动态链接的好处是生成的程序小，占用资源少。动态链接分为可执行程序装载时动态链接和运行时动态链接。

当用户启动一个应用程序时，它们就会调用一个可执行和链接格式映像。`Linux` 中 `ELF` 支持两种类型的库：静态库包含在编译时静态绑定到一个程序的函数。动态库则是在加载应用程序时被加载的，而且它与应用程序是在运行时绑定的。

![](http://upload-images.jianshu.io/upload_images/1627862-4d8e3d6069c189bd.gif?imageMogr2/auto-orient/strip)

###4.装载程序
我们在 `sys_execve` 处设置断点开始追踪分析，如下图：

![](http://upload-images.jianshu.io/upload_images/1627862-2eb10c9fbce76279.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![](http://upload-images.jianshu.io/upload_images/1627862-60e8ce526d6fdac9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

do_execve：  

![](http://upload-images.jianshu.io/upload_images/1627862-6251e658a1529f86.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

do_execve_common：

![](http://upload-images.jianshu.io/upload_images/1627862-e4da4410eacc7c6d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

exec_binprm：

![](http://upload-images.jianshu.io/upload_images/1627862-ed38a9b065d5e1a6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们发现 `execve` 是比较特殊的系统调用，`exec_binprm` 在保存了 `bprm` 后调用该函数来进一步操作，`execve` 加载的可执行文件会把当前的进程覆盖掉，返回之后就不是原来的程序而是新的可执行程序起点。这个函数除了保存 `pid` 以外，还执行了 `search_binary_handler` 来查询能够处理相应可执行文件格式的处理器，并调用相应的`load_binary` 方法以启动新进程。

search_binary_handler：

```
int search_binary_handler(struct linux_binprm *bprm)
{
    ...
    retry:
        read_lock(&binfmt_lock);
        //循环查找 linux_binfmt
        list_for_each_entry(fmt, &formats, lh) {
            if (!try_module_get(fmt->module))
                continue;
            read_unlock(&binfmt_lock);
            bprm->recursion_depth++;
            
            //对于 elf 文件，实际上执行的就是 load_elf_binary
            retval = fmt->load_binary(bprm);
            read_lock(&binfmt_lock);
            ...
}
```

load_elf_binary：

```
static int load_elf_binary(struct linux_binprm *bprm)
{
    ...

    //获取头
    loc->elf_ex = *((struct elfhdr *)bprm->buf);
    //读取头信息
    if (loc->elf_ex.e_phentsize != sizeof(struct elf_phdr))
        goto out;
    if (loc->elf_ex.e_phnum < 1 ||
        loc->elf_ex.e_phnum > 65536U / sizeof(struct elf_phdr))
        goto out;
    size = loc->elf_ex.e_phnum * sizeof(struct elf_phdr);
    retval = -ENOMEM;
    elf_phdata = kmalloc(size, GFP_KERNEL);
    if (!elf_phdata)
        goto out;
        ...
    //读取可执行文件的解析器
    for (i = 0; i < loc->elf_ex.e_phnum; i++) {
        if (elf_ppnt->p_type == PT_INTERP) {
            ...
    }
    ...
    //如果需要装入解释器，并且解释器的映像是ELF格式的，就通过load_elf_interp()装入其映像，并把将来进入用户空间时的入口地址设置成load_elf_interp()的返回值，那显然是解释器的程序入口。而若不装入解释器，那么这个地址就是目标映像本身的程序入口。
    if (elf_interpreter) {
        unsigned long interp_map_addr = 0;
        elf_entry = load_elf_interp(&loc->interp_elf_ex,
                        interpreter,
                        &interp_map_addr,
                        load_bias);
        if (!IS_ERR((void *)elf_entry)) {

            interp_load_addr = elf_entry;
            elf_entry += loc->interp_elf_ex.e_entry;
        }
        if (BAD_ADDR(elf_entry)) {
            retval = IS_ERR((void *)elf_entry) ?
                    (int)elf_entry : -EINVAL;
            goto out_free_dentry;
        }
        reloc_func_desc = interp_load_addr;
        allow_write_access(interpreter);
        fput(interpreter);
        kfree(elf_interpreter);
    } else {
        elf_entry = loc->elf_ex.e_entry;
        if (BAD_ADDR(elf_entry)) {
            retval = -EINVAL;
            goto out_free_dentry;
        }
    }
}
```
`load_elf_binary` 调用 `start_thread` 函数。修改 `int 0x80` 压入内核堆栈的 `EIP`，当 `load_elf_binary` 执行完毕，返回至 `do_execve` 再返回至 `sys_execve` 时，系统调用的返回地址，即 `EIP` 寄存器，已经被改写成了被装载的 `ELF` 程序的入口地址了。

###5.总结
`Linux` 系统通过用户态 `execve` 函数调用内核态 `sys_execve` 系统调用，负责将新的程序代码和数据替换到新的进程中，打开可执行文件，载入依赖的库文件，申请新的内存空间，最后执行 `start_thread` 函数设置 `new_ip` 和 `new_sp`，完成新进程的代码和数据替换，然后返回并执行新的进程代码。