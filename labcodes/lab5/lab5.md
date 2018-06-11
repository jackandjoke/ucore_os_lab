## LAB5

### 练习1: 加载应用程序并执行（需要编码）  

* do_execve要检查传入的用户程序名是否在合法的地址上
* * 是否在用户地址空间[USERBASE,USERTOP]
* * 
* 然后处理当前mm, 若mm的空间没有被别的程序使用，则释放掉。 
* 之后利用load_icode()加载新的用户程序
* * 设置新的mm struct
* *  设置PDT
* *  设置新的vma
* * 分配内存空间，复制TEXT/DATA
* *  分配内存空间，复制BSS
* * 建立用户栈
* * 将当前程序设置为新的程序
* * * 新的mm 
* * * 载入新的页表
* * 设置trapframe
* 最后设置新程序的名字

### 请在实验报告中描述当创建一个用户态进程并加载了应用程序后，CPU是如何让这个应用程序最终在用户态执行起来的。即这个用户态进程被ucore选择占用CPU执行（RUNNING态）到具体执行应用程序第一条指令的整个经过。

* 调用kernel_execve(), kernel_execve()调用system call里的sys_exec, sys_exec调用do_execve()
* do_execve如1中所说，将当前进程的mm设置为null,必要时候释放mm,然后调用load_icode()设置新的用户进程,载入到当前进程中  

### 练习2： 父进程复制自己的内存空间给子进程

* do_fork()里对两个步骤进行修改
* 一是要确保当前程序wait_state 为0
* 二是增加了set_links()
* 然后alloc_proc()里要对新增加的wait_state, cptr, yptr, optr进行初始化
* copy_range里，将原来page的内容复制到新的page里  

* trap.c里也有一些修改
* 要为用户态增加syscall的trap
* 要在时钟中断里每隔一段时间设置resched = 1;
* 

### 练习3: 阅读分析源代码，理解进程执行 fork/exec/wait/exit 的实现，以及系统调用的实现（不需要编码）

* fork:
* * 分配pcb
* * 设置内核栈
* * 拷贝mm内容
* * 设置tf到内核栈,然后设置内核开始位置，以及栈顶
* * 加入hash_list,
* * 加入proc_list,以及设置程序之间的关系

* exec
* * 将当前程序mm设置为Null
* * 必要时候删除原mm的空间
* * * 清除页表内容
* * * 删除页表
* * * 删除页表所占的空间
* * 调用load_icode()加载用户程序到当前程序   

* wait:
* * 若传入pid,表示要wait某个特定的子进程，否则就是查看所有子进程
* * 如果子进程是僵尸状态，则进入found操作，从hash_list里删除子进程，删除它的内核栈，释放它的PCB
* * 之后，设置当前程序为WT_CHILD,然后重新调度

* exit: 
* * 调用内核栈的页表，然后设置mm为Null,必要时候删除原来Null的内存空间
* * 设置当前状态为僵尸进程，并且设置exit_code
* * 如果夫进程在wait,正好唤醒
* * 如果有子进程，将他们交给initproc进程托管
* * 再重新调度别的进程








遇到一些奇怪的错误

* 时钟中断时print_ticks()会导致spin等后续几个没法通过  
* 