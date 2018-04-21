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


