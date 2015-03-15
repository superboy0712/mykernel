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

[myinterrupt.c](../myinterrupt.c) 
```c
void my_timer_handler(void)
{
  #if 1
    if(time_count%1000 == 0 && my_need_sched != 1)
    {
    printk(KERN_NOTICE ">>>my_timer_handler here<<<\n");
    my_need_sched = 1;
    }
    time_count ++ ;
  #endif
  return;
}
```
定时中断服务程序. 周期性的触发,用于增加 ```time_count ++``` 每一千次置位一次 标志 ```my_need_sched``` 用于 通知进程可执行(spin locked by this flag, not really suspended). 此标志用于WHILE loop阻塞轮询式锁,并非真正挂起.

从[line 54](../myinterrupt.c#L54)开始,如下
```c
if(next->state == 0)/* -1 unrunnable, 0 runnable, >0 stopped */
{
  /* switch to next process */
  asm volatile(
    "pushl %%ebp\n\t" /* save ebp */ /* 将此时的 栈底指针EBP入栈保护*/
    "movl %%esp,%0\n\t" /* save esp */ /* 将当前进程的 栈顶指针ESP存入当前 进程的 thread.sp. 目的在保留切出前最新SP*/
    "movl %2,%%esp\n\t" /* restore esp */ /* 将esp 指向 next进程的 thread.sp */
    "movl $1f,%1\n\t" /* save eip */	/* 保留 标号 1 处的地址到 当前进程的 THREAD.EIP */
    "pushl %3\n\t" /* 将next 进程的 最近一次 切换前的 EIP 入栈, next 进程的 1标号处的地址 */
    "ret\n\t" /* restore eip */ /* 跳转到next 进程 最近一次切出 处 */
    "1:\t" /* next process start here */
    "popl %%ebp\n\t" /*恢复 ebp */
    : "=m" (prev->thread.sp),"=m" (prev->thread.ip)
    : "m" (next->thread.sp),"m" (next->thread.ip)
  );
  my_current_task = next;
printk(KERN_NOTICE ">>>switch %d to %d<<<\n",prev->pid,next->pid);
}
```
从[line 72](../myinterrupt.c#L72)开始,如下
```c
else /* 当next进程非runnable 时 */
{
  next->state = 0; /* 置为 runnable */
  my_current_task = next;
  printk(KERN_NOTICE ">>>switch %d to %d<<<\n",prev->pid,next->pid);
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
```
见如上对应的简体中文评论.

##Some modification and Test
改成了被动调度,希望能从当前进程arbitrary position 切换出去. 但是只成功从 0 到 1, 然后一直 1.
###Why Passive Scheduling won't work here?
时间中断服务函数 在编译器的默认编译下会加上 reti 函数, 强迫恢复中断前的EIP而没法改. 
###Possible Answer

##QEMU + GDB Debugging environment
看图和referrence[1][4]
![](./week2_img1.jpg)
##The Dollar Label Stuff
看 Reference[2]

##Reference
[1][QEMU + GDB Debugging Environment](https://www.ece.cmu.edu/~ee349/f-2012/lab2/qemu.pdf)  
[2][The Dollar Local Label Symbols](https://sourceware.org/binutils/docs-2.18/as/Symbol-Names.html) , [Symbol Names](http://tigcc.ticalc.org/doc/gnuasm.html#SEC46)  
[3][Debugging The Linux Kernel Using Gdb](http://www.elinux.org/Debugging_The_Linux_Kernel_Using_Gdb#Debugging_a_kernel_module_.28.o_and_.ko_.29)  
[4][Kernel Debugging Tips: Using Kernel Symbols](http://elinux.org/Kernel_Debugging_Tips#Using_kernel_symbols)  
[5][GNU Extended Asm](https://gcc.gnu.org/onlinedocs/gcc/Extended-Asm.html)  
[6][gcc inlne asm](http://www.cs.dartmouth.edu/~sergey/cs108/2009/gcc-inline-asm.pdf)
