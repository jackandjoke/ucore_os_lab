## LAB6

###练习1: 使用 Round Robin 调度算法（不需要编码）

* 将前几个lab的代码merge时，需要修改两个地方。  
* * kern/process/proc.c里对struct proc进行初始化时，要最lab6新增的域进行初始化，如run_link, lab6_run_pool等  
* * kern/trap/trap.c里，时钟中断处，每次要记得调用sched_class_proc_tick()，调用下一个程序。
* sched_init里初始化调度器为Round robin, sched_class里的函数指针的意义：
* * init: 初始化工作,对RR，就是就绪链表初始化，程序个数清零。
* * enqueue, 将某个进程加入就绪链表，对RR，还要设置时间片数，增加程序个数
* * dequeue, 将某个进程从就绪链表里删除，对RR，还要再初始化该进程的run_link, 并且减少程序个数  
* * pick_next, 从就绪链表里跳出下一个执行的进程  
* * proc_tick, RR里是指让进程运行完整个时间片，再启动新的调度。  
* 