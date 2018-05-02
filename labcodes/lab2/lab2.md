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

