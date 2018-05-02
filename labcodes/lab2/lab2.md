## 练习0：填写已有实验   
* 使用meld, 手动合并  
## 练习1：实现 first-fit 连续物理内存分配算法（需要编程）
* first-fit 指使用第一个找到满足要求的空闲块来分配  
* 需要实现 default_init, default_init_memmap, default_alloc_pages, default_free_pages  
* 空闲块由一个个page组成，struct Page 则是物理帧的描述项，包括
* * 该page的ref (被多少个逻辑页所引用)
* * flags (第0位表示是否reserved给kernel使用，第1位表示是否为一个块的head page )
* *  property(表示page为head page时，后面有多少个连续空闲的page, 即该空闲块有多少个page)  
* * page_link ( 是list_entry的链表项， 用来加入到free_list链表中，可以指向前一个空闲块，以及后一个空闲块)  

* default_init的实现：
* * 初始化free_area.free_list , 以及初始化nr_free为0 （number of rest free list） 

* default_init_memmap (struct Page* base, size_t n )的实现：
* * 该函数将 base 开始的 连续n个物理页帧初始化，并且加入到free_list中  
* * 这n个页帧要将他们的ref设置为0，
* * 要将他们的property设置为0，除了base的property设置为n  
* * 要将他们的flags设置为0，除了base的flags中表示是否为head page的位设置为 1   
* * nr_free增加n个  
* * 将base的page_link加入到free_list中  
* * 答案中将page_link插入到free_list的前面，这样做的化就是查找空闲块的时候是按照空闲块插入的先后顺序查找的  

* default_alloc_pages (size_t n):
* * 该函数是要按照first-fit找到第一个满足大于等于n个page的空闲块，然后分配出去  
* * 首先判断空闲帧的总数是不是大于n  
* * 然后在free_list中，从前往后查找，  
* * 要将list_entry转成page, 然后看page->property是否》=n 
* * 找到之后，除了分配n个page外, 要将后续多余的page，重新连接进入free_list中,放在分配的page对应的list_entry后面,然后删除要分配的page的list_entry  

* default_free_pages(struct Page *base, size_t n)  
* * 该函数用来释放以base为head page,连续n个page的块，回到free_list中  
* * 首先将这些page都清除下，flag设置为0，ref设置为0,property 不用设置，因为在alloc时，head page的property被设置为0了，而不是head page的本身property就是0。
* * 对base 所代表的page, property 要设置回n  
* * 然后在free_list中寻找要插入的位置  
* * 应该插入在某个list_entry所对应的page之前，该page的地址比base+n要大于等于  
* * 而且在插入时，还应该顺带判断base是否能和它之前的空闲块合并，以及它是否能和它后面的空闲块合并  
* * 合并完之后，要再从free_list中找到合适的位置插入  *   


## 发现了lab1 的kern/debug/kdebug.c 里的print_stackframe有一个bug
### 要先更新eip, 然后更新ebp, 因为eip需要用旧的ebp的值  


## 练习2  练习2：实现寻找虚拟地址对应的页表项（需要编程）  

### 主要是实现kern/mm/pmm.c 中的get_pte函数  
* get_pte函数输入为一个指向页目录表的指针，虚拟地址（逻辑地址），以及一个create参数（表示若页目录中的某一项表示的页表还没有构建的时候，是否需要分配一个page来构建该页表）
* 首先在页目录中查询，index是PDX（la）
* 若当前页目录项不存在或者present位为0， 这都表示要去查的二级页表还没有建立好
* * 如果create为0，表示不用建立，则直接返回NULL
* * create为1，表示要建立该页表
* * * 首先分配一个page, 
* * * 然后将该page清空，因为新建页表就是什么都没有存放  
* * * 该page的ref加1，表示被引用了（用途是存放页表）
* * 建立完新的页表后，要将其地址填入页目录表项目中
* * 页目录中的每一项都是高20位有用的，低12位全为0，其实这代表某个页表的基地址，低12位是用来表示叶内偏移量的。
* *  每个页目录项的最低3位，分别表示present, writeable, user accessible, 也要设置  
* 最后返回页表（不是页目录）的对应于la的（用PDX（la）作为索引）的那一项的地址。  

#### 请描述页目录项（Page Directory Entry）和页表项（Page Table Entry）中每个组成部分的含义以及对ucore而言的潜在用处  
* 页目录项的值有两部分含义：
* * 首先高20位组合上低12位的0而得到的32位地址，是用来表示页表的基地址，
* * 低3位代表了present, writeble, user accessible  
* 页表项目的值也有两部分含义：
* * 高20位表示线形地址对应的物理地址的高20位，
* * 低3位和页目录项一样，代表p,w,u   
 
#### 如果ucore执行过程中访问内存，出现了页访问异常，请问硬件要做哪些事情？    
* 主要做的事情有：
* 查看是否有空闲的物理页能被映射到该逻辑页，
* 没有的话，需要用某个替换方法，将某个物理页替换成该逻辑页
* 该物理页原来的内容可能需要写回内存，根据dirty位判断  
* 更新页表,tlb表
* 重新执行产生页访问异常的指令  


## 练习3：释放某虚地址所在的页并取消对应二级页表项的映射（需要编程）
* 首先检查要取消的二级页表项是否存在  
* 存在的话，将二级页表项对应的page的ref减少1，
* 若ref==0, 则free该page  
* 然后将二级页表项内容清零，
* 以及tlb相关内容设置为无效了  

#### 数据结构Page的全局变量（其实是一个数组）的每一项与页表中的页目录项和页表项有无对应关系？如果有，其对应关系是啥？  
* page中是为每一个物理页都建立了一个page结构，页表目录项的值对应一个page,该page被用来建立一个页表，页表项的值对应一个物理frame,该物理frame对应于逻辑地址所在的page
