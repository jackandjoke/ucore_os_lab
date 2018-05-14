## LAB 4  
### 练习1：分配并初始化一个进程控制块（需要编码）  
* alloc_proc函数是分配一个PCB的空间，然后不做实际的资源分配    
* 这里的初始化是用个默认值占位，而不是初始的设置正确的值，因为后续会进行相应的正确的设置，所以对于数值型的，设置为0/-1，对于指针，设置为NULL，对于数组，设置为0  
* 但是cr3 默认要设置boot_cr3  
* struct context context 的作用是保存在内核态切换进程/线程时候的当前执行现场的寄存器，eip,esp,ebx,ecx,edx,esi,edi,ebp ，当调度器选择该线程，则需要从context恢复它的执行现场
* struct trapframe *ft 保存中断帧，作用是用于用户态到内核态的切换时候，将被打断程序的现场保存在内核栈的某空间里，当要从内核态切换回用户态时，可以从那里恢复用户态的现场  
*  

### 练习2：为新创建的内核线程分配资源（需要编码）  
* do_fork函数用来创建内核线程  
* 首先分配PCB,
* 然后为PCB的kstack设置  
* * idle_proc直接使用bootstack, 对其他的proc,用alloc_pages(KSTACKPAGE) 分配一段空间用作kstack  
* 设置该线程的parent  
* 将mm（内存管理信息）也复制
* * copy_mm根据clone_flag来决定是共享mm还是copy  
* 将tf复制，同时也保存用户栈的esp  
* * 为tf分配空间，然后将临时的tf拷贝进去，设置context上下文，一个细节是要设置tf_eflags要使能中断，表示内核线程执行时可以被中断  
* 在进行获取pid,hash proc, 将PCB添加到链表里时，要暂时disable中断  (为啥呢)
* * hash_proc将proc加入hash表里，这个表是用来根据pid查找proc时更快
* * 但是proc本身也要也要加入proc_list里，proc_list有所有的proc
* wake_up该线程
* 返回该线程的pid  

* kernel_thread()可以认为是do_fork的wrapper, 主要用临时变量设置好kernel的tf中断帧，然后调用do_fork  
* kernel_thread_entry是内核线程开始执行的位置  
* * 做的事情是，将线程主体函数所需的次参数压栈，然后call主体函数，最后将结果保存在eax,call do_exit  


* schedule()  用来查找第一个符合RUNNABLE的proc
* 通过遍历proc_list  
* 在查找的时候要屏蔽中断  

* proc_run ,切换执行proc   
* 恢复现场时要屏蔽中断  
* * 设置它为当前运行程序  
* * 设置ts特权级为0的kstack为它的kstack,因为之后可能会有用户态/内核态的切换, 从用户态到内核态转换时，该栈用来压栈原用户态的现场，从内核态到用户态时，则是从该栈压栈保存内核态的现场，最后iret返回时，从该栈出栈恢复现场  
* * 读取它的页表  
* * 切换上下文  

本实验有2个内核线程  
local_intr_save(intr_flag)， local_intr_restore是用来屏蔽和使能中断的

