## 操作系统镜像文件ucore.img是如何一步一步生成的？(需要比较详细地解释Makefile中每一条相关命令和命令参数的含义，以及说明命令导致的结果)

### 直接查看lab1/Makefile：
* 得到ucore.img保存的位置为：bin/ucore.img
```makefile
UCOREIMG := $(call totarget, ucore.img)
```

* 首先，将bin/ucore.img 生成为 512*10000个字节大小的空文件，  
* 然后将bootloader所在的block(占用512个字节)，覆盖它的前512个字节，  
* 最后，在这个512字节后，写入kernel
```makefile
$(UCOREIMG) : $(kernel) $(bootblock)
    $(V)dd if=/dev/zero of=$@ count=10000
    $(V)dd if=$(bootblock) of=$@ conv=notrunc
    $(V)dd if=$(kernel) of=$@ seek=1 conv=notrunc
```

* create_target调用了do_create_target  

```makefile
$(call create_target,ucore.img)
```

* kernel保存的文件名/bin/kernel  
* 编译kernel依赖kernel.ld这个link的script  

```makefile
kernel = $(call totarget,kernel)
$(kernel): tools/kernel.ld
$(kernel): $(KOBJS)
    @echo + ld $@ 
    
```

* $(OKBJS) 为kernel要用的libs， 有将obj/kern下的文件夹里的各种.o文件link起来  
```makefile
KOBJS = $(call read_packet,kernel libs)
read_packet = $(foreach p,$(call packetname,$(1)),$($p)))
packetname = $(if $(1), $(addprefix $(OBJPREFIX),$(1)),$(OBJPREFIX))
```
### 在shell里使用 Make V= ：
* kernel和bootblock都是link obj/kern 以及boot/下的.o文件得到的 
```bash
ld -m    elf_i386 -nostdlib -N -e start -Ttext 0x7C00 
obj/boot/bootasm.o obj/boot/bootmain.o -o obj/bootblock.o

ld -m    elf_i386 -nostdlib -T tools/kernel.ld -o 
bin/kernel  obj/kern/init/init.o obj/kern/libs/stdio.o
 obj/kern/libs/readline.o obj/kern/debug/panic.o 
obj/kern/debug/kdebug.o obj/kern/debug/kmonitor.o 
obj/kern/driver/clock.o obj/kern/driver/console.o 
obj/kern/driver/picirq.o obj/kern/driver/intr.o 
obj/kern/trap/trap.o obj/kern/trap/vectors.o 
obj/kern/trap/trapentry.o obj/kern/mm/pmm.o 
 obj/libs/string.o obj/libs/printfmt.o
+ cc boot/bootasm.S
```

* 他们用到的.o文件，都是先用gcc将相应的.c文件编译成.o的可重定向文件,比如  
```bash
gcc -Ikern/debug/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  
-fno-stack-protector -Ilibs/ -Ikern/debug/ -Ikern/driver/
 -Ikern/trap/ -Ikern/mm/ -c kern/debug/kmonitor.c -o obj/kern/debug/kmonitor.o
```

* 而link bootblock, 用到了boot/下的.o文件

* 除此之外，还编译出了sign的可执行文件
```bash
gcc -Itools/ -g -Wall -O2 -c tools/sign.c -o obj/sign/tools/sign.o
gcc -g -Wall -O2 obj/sign/tools/sign.o -o bin/sign
```
## 一个被系统认为是符合规范的硬盘主引导扇区的特征是什么？  
### 查看tools/sign.c  
* __512字节__ 
* 首先检测文件是否超过510字节  
```C
if (st.st_size > 510) {
     fprintf(stderr, "%lld >> 510!!\n", (long long)st.st_size);
     return -1;
}
```
* 建立一个512字节的buf,将文件里的内容全读进去  
```C
char buf[512];
int size = fread(buf, 1, st.st_size, ifp);
```
* 将 __最后两个字节设置成为0x55和0xAA__
```C
buf[510] = 0x55;
buf[511] = 0xAA;
```
* 写到ouput文件，此处为bin/bootblock,
```C
size = fwrite(buf, 1, 512, ofp);
```
* 输入输出可以在make V= 里看到是obj/bootblock.out以及bin/bootblock
```bash
'obj/bootblock.out' size: 488 bytes
build 512 bytes boot sector: 'bin/bootblock' success! 
```


# 练习2    
* gdb显示的第一条指令是0x0000fff0, 为eip的值
* info registers ,看到eip为0xfff0, cs 为0xf000, cs*16 + eip 应该为0xffff0，是第一条指令的地址
* 问题是gdb里显示0x0000fff0, 但是执行的代码却是0xfffff0的代码，所以下面的hoop-stop加上$cs的值0xf0000
* tools/gdbinit是gdb的初始化文件，加入  
```
define hook-stop
x/i $pc
end
```
* 可以使gdb显示地址，以及相应的汇编语句  
### 单步调试BIOS  
* 用x /10i 0xffff0 查看地址0xffff0附近的10条汇编，知道了要跳转到0xfe05b
```asm
   0xffff0: ljmp $0xf000, $0xe05b
```
要跳转到0xfd165
```asm
   0xfe05b:     cmpl   $0x0,%cs:0x6c48
   0xfe062:     jne    0xfd2e1
   0xfe066:     xor     %dx,%dx                 # 清零dx寄存器
   0xfe068:     mov    %dx,%ss                  # ss置0
   0xfe06a:     mov    $0x7000,%esp         # 栈顶指针设为0x7000
   0xfe070:     mov    $0xf3691,%edx
   0xfe076:     jmp    0xfd165
```
```asm
   0xfd165:     mov    %eax,%ecx
   0xfd168:     cli                                 # 设置EFLAGS的IF为0，屏蔽中断
   0xfd169:     cld                             #DF=0, Direction flag
   0xfd16a:     mov    $0x8f,%eax
   0xfd170:     out    %al,$0x70        #从端口0x70输出%al
   0xfd172:     in     $0x71,%al
   0xfd174:     in     $0x92,%al
   0xfd176:     or     $0x2,%al
   0xfd178:     out    %al,$0x92    # enable A20 地址线
   0xfd17a:     lidtw  %cs:0x6c38   # 读取idt
   0xfd180:     lgdtw  %cs:0x6bf4  # 读取gdt
   0xfd186:     mov    %cr0,%eax
   0xfd189:     or     $0x1,%eax
   0xfd18d:     mov    %eax,%cr0    #设置cr0=1, 进入保护模式
   0xfd190:     ljmpl  $0x8,$0xfd198
```

* 刚开始为实模式，BIOS读取的第一条指令所在的地址为0xffff0, 该地址处的指令为一条跳转指令
* 跳转之后，进行一些初始化的工作，比如设置esp, 屏蔽中段，设置DF，读取idt,gdt, 进入保护模式  

###  在0x7c00设置实断点  
* 0x7c00是bootloader的启始地址  
###  从0x7c00跟踪代码,比较反汇编代码和bootasm.S的区别  
* 可以不管gdbinit，直接在gdb里调试  
* Makefile里debug里加入一些参数，可以保存调试的log  
### 自己找一个bootloader或内核中的代码位置，设置断点并进行测试  


##3. 分析bootloader进入保护模式的过程   

* 禁止中断  
* 设置DF
* 设置DS，ES，SS段寄存器为0，
* 开启A20，
* 设置cr0为1，
* ljmp跳到保护模式  
```asm
    ljmp $PROT_MODE_CSEG, $protcseg  # 将CS设置为$PROT_MODE_CSEG, EIP设置为$protcseg
```
* 设置DS，ES，SS
* 设置栈， ebp, esp 
* call bootmain




### 为何开启A20， 以及如何开启A20  
* 在8086机器中，内存的大小只有1MB，也就是$2^{20}$，所以理论上需要20位的地址线，但是8086的地址线只有16位，所以它使用的是segment:offset的方式来访问内存地址。
* segment:offset的原理是，将CS寄存器保存的值乘以4,然后加上EIP寄存器的值。
* 但是ffff0h + ffffh = 10ffefh 大于1MB, 所以超过的部分就回卷  
* 在后续的80286中，有24位的地址线，内存为16MB，为了向下兼容，就设置了A20来模拟回卷。 第21个bit来表示是否开启A20，没开启的话，就只能访问1MB的空间，若要访问高地址空间，必须要打开。  
* 打开的方式是通过向8042键盘控制器的P21输出端口发送信号  
* 8042有input buffer, output buffer, status register  
* 像P21输出端口发送信号的步骤有：  
* 先等待input buffer空闲，通过读0x64的status register 看第1个bit是否为1判断
* 向它发送一个字节的命令0xd1,表示要写数据到P21的端口  
* 再次等待input buffer 空闲，
*  将0xdf 写入到0x60端口（也就是input buffer的端口）  

### 如何初始化GDT表  
* 第一个段描述符为空  
* 第二个为code segment  (读/执行,0,0xffffffff)
* 第三个为data segment   (写，0,0xffffffff)
* 每个segement 由(type，基地址，长度)或者(type,base,limit,dpl)组成，分别在asm.h和mmu.h里定义的  
* 

### 如何使能和进入保护模式  
* 通过将%cr0设置为1打开使能，
* 然后通过ljmp进入保护模式  
* 

## 练习4  
### bootloader如何读取硬盘扇区的  
* 通过CPU访问硬盘的IO地址寄存器  
* 等待硬盘空闲，可以通过0x1f7端口判断  
* 向0x1f2端口写入要读几个扇区，
* 传入要读的参数，分别是0x1f3,0x1f4,0x1f5,0x1f6这个几个端口  
* 向0x1f7端口写入要读取的命令  
* 等待硬盘空闲  
* 调用insl读取扇区, 每次读取都是2个字（一个字2个字节）  


### bootloader 是如何加载ELF格式的OS  
* 首先加载elf的header, 有8个扇区？
* 检查magic是否为elf的magic  
* 然后读取header里的program header  
* 通过program header将需要的段加载到相应的内存里 
* 跳转到os入口  


## 练习5  
### 完成kern/debug/kdebug.c中的print_stackframe函数    
* 通过read_eip()读取到当的eip值
* 通过read_ebp()读到当前 ebp的值  
* 循环打印堆栈信息：
* 传进函数的四个参数（从第一个到第四个）分别在ebp+8,ebp+12,ebp+16，ebp+20的位置  
* 下一个ebp的位置在ebp所指向的地址里，即*(ebp指针)  
* 下一个eip的位置在返回地址里，即*(ebp指针+1)  
* ebp的边界在0，当ebp等于0的时候跳出循环   
* 忽略的一个是栈不能无限的递归调用，而是有个上限，为STACKFRAME_DEPTH  
* 



