#Week2: Tiny Shared Time Slice/Round Robin scheduling kernel
# 第二周: 时间片小内核

  Author: Yulong Bai  
  Please give credit to me if you want to use it somewhere else.  
  This is a solution to the assignments in A MOOC course [Linux kernel analysis](http://mooc.study.163.com/course/USTC-1000029000)

  白宇龙  
  原创作品转载请注明出处   
[\<Linux内核分析\> MOOC课程](http://mooc.study.163.com/course/USTC-1000029000)

##Requirements
>1.题目自拟，内容围绕操作系统是如何工作的进行；  
>2.博客中需要使用实验截图;  
>3.博客内容中需要仔细分析进程的启动和进程的切换机制;  
>4.总结部分需要阐明自己对“操作系统是如何工作的”理解;  
  
##Warm up & Play Around
##原版例子解析line by line
此处为原有例子源文件[myinterrupt.c](../myinterrupt.c) 和 [mymain.c](../mymain.c)
以下将对关键的代码进行解释.  
 [mymain.c](../mymain.c)
``` c
void __init my_start_kernel(void)
```
```__init``` 的用法见 [what-does-init-mean-in-the-linux-kernel-code](http://stackoverflow.com/questions/8832114/what-does-init-mean-in-the-linux-kernel-code)
简而言之,用于 告知 编译器 和 连接器 把 所修饰的函数 置于指定的 可执行二进制文件 elf 所在的 section. 此处置于```__section__(".init.text")``` 即初始化代码段,在进入main函数入口前.

``` c
task[pid].task_entry = task[pid].thread.ip = (unsigned long)my_process;
task[pid].thread.sp = (unsigned long)&task[pid].stack[KERNEL_STACK_SIZE-1];
task[pid].next = &task[pid];
```
0进程初始化, ip 指向 函数 my_process, sp 指向 PCB 中 STACK成员的最后一个 unsigned long element. 因为stack向低地址生长,固初始化指向最大地址. PCB.NEXT 指向自己.
```c
/*fork more process */
for(i=1;i<MAX_TASK_NUM;i++)
{
  memcpy(&task[i],&task[0],sizeof(tPCB));
  task[i].pid = i;
  task[i].state = -1;
  task[i].thread.sp = (unsigned long)&task[i].stack[KERNEL_STACK_SIZE-1];
  task[i].next = task[i-1].next;
  task[i-1].next = &task[i];
}
```
初始化其他的进程,并且维护好其next指针. 其他进程初始化为unrunnable 状态

```c
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
如评论. 让ESP 指向 0进程 栈顶(此时也是栈底). 把0进程栈底ebp入栈(orz 真绕.. ), 把 当前IP 入栈,也就是myprocess 入口地址. 跳转到 myprocess. 除非 myprocess 返回, 否则 ```"popl %%ebp\n\t"``` 永远不会执行.

##Some modification and Test

###Why Passive Scheduling won't work here?

###Possible Answer

##QEMU + GDB Debugging environment

##The Dollar Label Stuff


##Reference
[1][QEMU + GDB Debugging Environment](https://www.ece.cmu.edu/~ee349/f-2012/lab2/qemu.pdf)  
[2][The Dollar Local Label Symbols](https://sourceware.org/binutils/docs-2.18/as/Symbol-Names.html) , [Symbol Names](http://tigcc.ticalc.org/doc/gnuasm.html#SEC46)  
[3][Debugging The Linux Kernel Using Gdb](http://www.elinux.org/Debugging_The_Linux_Kernel_Using_Gdb#Debugging_a_kernel_module_.28.o_and_.ko_.29)  
[4][Kernel Debugging Tips: Using Kernel Symbols](http://elinux.org/Kernel_Debugging_Tips#Using_kernel_symbols)  
[5][GNU Extended Asm](https://gcc.gnu.org/onlinedocs/gcc/Extended-Asm.html)  
[6][gcc inlne asm](http://www.cs.dartmouth.edu/~sergey/cs108/2009/gcc-inline-asm.pdf)
