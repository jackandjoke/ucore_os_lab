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