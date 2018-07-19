
# 1.计算机体系结构  
计算机诞生距今正好 60 年的历史，尽管如今的计算机在性能指标、运算速度、应用领域等方面与过去有了天翻地覆的差别，但是基本的体系结构却一直没有变过，也就是大家常说的[冯·诺依曼体系结构](https://en.wikipedia.org/wiki/Von_Neumann_architecture#Von_Neumann_bottleneck)。  

冯·诺依曼开创性的提出了"存储程序"的概念，把指令和数据同时存放在存储器中，奠定了现代计算机的结构理论。

##### 概括来说，主要有如下三点：  
 - 计算机主要由运算器、存储器、控制器、输入和输出设备组成，以运算器为中心，I/O 设备与存储器之间的数据流都必须经过运算器；  
 - 计算机内部使用二进制表示指令和数据，每一条指令由表示运算类型的操作码和存储器位置的地址码组成； 
 - 计算机的运行过程，是不断的把程序和数据读入主存储器，然后，按照顺序自动逐条取出指令并执行的的过程；  

# 2.反汇编分析一段 C 语言程序的执行  
```C
int g(int x)
{
    return x + 9;
}
     
int f(int x)
{
    return g(x);
}
     
int main(void)
{
    return f(8) + 6;
}
```
通过[实验楼](https://www.shiyanlou.com/)的已搭建好的 64 位 Linux 虚拟机环境，反汇编上述 C 代码：

```
gcc -S -o main.s main.c -m32
```
汇编代码：
```
.file	"main.c"
.text
.globl	g
.type	g, @function
g:
.LFB0:
.cfi_startproc
pushl	%ebp
.cfi_def_cfa_offset 8
.cfi_offset 5, -8
movl	%esp, %ebp
.cfi_def_cfa_register 5
movl	8(%ebp), %eax
addl	$9, %eax
    popl    %ebp
    .cfi_restore 5
    .cfi_def_cfa 4, 4
    ret
    .cfi_endproc
.LFE0:
    .size	g, .-g
    .globl	f
    .type	f, @function
f:
.LFB1:
    .cfi_startproc
    pushl   %ebp
    .cfi_def_cfa_offset 8
    .cfi_offset 5, -8
    movl    %esp, %ebp
    .cfi_def_cfa_register 5
    subl    $4, %esp
movl	8(%ebp), %eax
movl	%eax, (%esp)
call	g
leave
.cfi_restore 5
.cfi_def_cfa 4, 4
ret
.cfi_endproc
.LFE1:
.size	f, .-f
.globl	main
.type	main, @function
main:
.LFB2:
.cfi_startproc
pushl	%ebp
.cfi_def_cfa_offset 8
.cfi_offset 5, -8
movl	%esp, %ebp
.cfi_def_cfa_register 5
subl	$4, %esp
    movl    $8, (%esp)
call	f
addl	$6, %eax
leave
.cfi_restore 5
.cfi_def_cfa 4, 4
ret
.cfi_endproc
.LFE2:
.size	main, .-main
.ident	"GCC: (Ubuntu 4.8.2-19ubuntu1) 4.8.2"
.section	.note.GNU-stack,"",@progbits
```

### 汇编代码分析：
首先从 `main` 函数开始，前两条语句：

```
pushl   %ebp
movl    %esp, %ebp
```
出现在所有函数的开头，表示将基指针压进栈，然后把栈顶指针的值赋给基指针。

```
subl	$4, %esp
movl	$8, (%esp)
```
然后，由于栈是由高地址向低地址扩展，在 32 位操作系统系统上，每个栈元素为 4 个字节，所以栈顶指针减 4，同时，采用立即数寻址的方式，把局部变量 8 放入栈顶指针所指向的位置。`call` 指令调用函数 f 执行，在这里 `call` 首先保存当前 %eip 的值，然后读取待执行函数 f 的指令地址：
```
call f
```
函数 f 以同样的形式进栈并继续调用函数 g，把局部变量 9 保存在 `Accumulator Register` eax 中，然后 `popl` 出栈返回到先前函数 f 中 `call` 指令保存执行指令地址：
```
addl	$9, %eax
popl    %ebp
```
函数 f 的 `leave` 指令相当于：
```
movl  %ebp, %esp
popl  %ebp
```
返回到 `main` 函数中继续执行直至函数运行结束，在此不再赘述。
```
addl	$6, %eax
leave
```

# 3.总结：  
##### 冯·诺依曼体系告诉我们，程序数据和指令被计算机以二进制的形式不加区分的存储在存储器中，理解了"存储程序"，然后再去理解计算机的执行过程就变得非常容易了。  
 - 取出指令：从存储器某个地址中取出要执行的指令送到 CPU 内部的指令寄存器暂存。  
 - 分析指令：把保存在指令寄存器中的指令送到指令译码器，译出该指令对应的微操作。  
 - 执行指令：根据指令译码，向各个部件发出相应控制信号，完成指令规定的各种操作。  
 - 最后，为执行下一条指令作好准备，即取出下一条指令地址。  
