# Memcached 源码分析 - 内部数据结构之Slab Allocator

##概览
slab分配器常见于各种系统级软件的内部实现中，包括Linux内核也使用了slab来作为分配常用数据结构（如task_struct）的方法。slab分配器的核心概念就是接管内存块的分配，避免内存块的频繁分配和释放，同时可以避免因频繁分配和释放引起的外部碎片；另外，slab分配器被划分为大小各不相同的类，在满足不同内存块大小请求的情况下，尽可能减少内部碎片。Linux内核中的slab分配器较为复杂，考虑到了缓存着色等问题，同时还需要与伙伴系统合作。  

Memcached的slab分配器实现比较朴素，可以看做是更高级“空闲链表”的实现，当然，较新版本的Memcached也在slab中加入了“slab重平衡”这种动态调整的特性。

总体来看，Memcached的slab分配器有如下几个特点：

1. slab类最大块大小默认为1M，也就是说Memcached中所能存储的最大对象大小为1M；当然，这个最大值可以在Memcached启动时使用"-I"参数来设置，允许范围是1024 bytes ~ 128 mb
2. slab类最小块为`item`结构加一对很小的key、value，默认为`sizeof(item) + 48`；也可以通过"-n"参数进行设置

##结构
slab分配器以chunk(块)作为内存分配的单元，按照预先指定的块大小slab被划分成不同的类，以满足不同的内存大小请求。类编号从`POWER_SMALLEST = 1`到`power_largest`，最小的类（编号为1）默认块大小为`sizeof(item) + 48`，然后编号为2的类块大小为`(sizeof(item) + 48) * factor`，编号3的类块大小为`(sizeof(item) + 48) * factor^2`，以此类推，直到最大的块大小默认为1MB。Memcached以`slabclass_t`来表示slab类，该结构如下：

    typedef struct {
    	unsigned int size;     	/* 该slab类的chunk大小 */
    	unsigned int perslab;  	/* 每一个slab页中有多少个item */

    	void *slots;          	/* 空闲item块的链表 */
    	unsigned int sl_curr; 	/* 空闲item块链表长度 */

    	unsigned int slabs;    	/* 我们已经为该slab类分配了多少页 */

    	void **slab_list;      	/* slab页链表 */
    	unsigned int list_size;	/* slab页链表长度 */

    	unsigned int killing;  	/* index+1 of dying slab, or zero if none */
    	size_t requested; 		/* The number of requested bytes */  
    } slabclass_t;
    
    // 代码中预先分配了MAX_NUMBER_OF_SLAB_CLASS = POWER_LAGREST + 1个slab类结构
    // 但是我们只会用到其中编号为[1, power_largest]范围内的结构
    static slabclass_t slabclass[MAX_NUMBER_OF_SLAB_CLASSES];  
    
可以看到，slab分配器引入了slab页的概念，slab在向系统请求内存空间时，总是请求至少一页（这里的页大小未必与系统虚拟内存页面大小一致），然后将这一页划分成指定大小的块链入空闲链表`slots`中，这样做也是为了减少内存分配系统调用的次数，减少内存碎片，同时尽量保证存储的内容在物理空间上连续，以提高系统缓存的命中率。

![slabclass](https://github.com/Mudenng/inside-memcached/raw/master/images/slabclass.png)


下面我们来看一下slab分配器重要操作的实现细节。

##slab类查找/匹配
`slabs_clsid(size)`是一个简单的函数，用来查找最适合存储size大小块的slab类编号。其原理就是从编号为1的slab类开始找到第一个slab块大小大于等于size的类，返回其类编号；如果没有满足要求的类，则返回0。

##初始化
slab分配器的初始化由`slabs_init(limit, factor, prealloc)`完成，该函数有3个参数，`limit`参数指定了slab分配器能够使用的最大内存大小，这个内存大小也就是Memcached能够用来存储对象的最大内存大小了；`factor`是slab块大小的增长乘数，其值越大，各slab类中间的区分度越大，在一些情况下可能更容易满足不同的请求大小，但是同样也可能增加内部碎片，该值最初默认为2，在后续版本的Memcached中进行了调整，目前默认为1.25；`prealloc`用来指定是否需要提前分配空间，提前分配有利于缓存系统，也可以略微提高运行中的性能。  
若指定了提前分配，该函数首先调用malloc分配`limit`大小的内存，由`mem_base`指向，slab今后需要分配新的slab页时均从该块内存中划分得到。之后，函数为每个slabclass初始化，填入其对应的块大小和每一页的块数目，页大小被指定为`item_size_max`，同时也是最大item块大小，默认为1MB。需要注意的是，编号最大的slab类（power_largest）块大小总是等于`item_size_max`，并且该类中每一页都只包含一块。  
简单的slabclass初始化完成后，调用`slabs_prealloc()`为每一个slab类预先分配一个slab页面，slab页面的分配工作由`do_slabs_newslab()`完成。

##分配新slab页面
分配一个新slab页的工作由`do_slabs_newslab(id)`完成，它首先调用`grow_slab_list(id)`对编号为id的slab类进行检查，如果它的页面链表已满，则需要对链表扩容，将`slab_list`的容量加倍；之后，为新页分配一块空闲空间，调用`split_slab_page_into_freelist()`将新页划分为一个个chunk，并链到空闲链表`slots`中；最后，将新分配好的这一页面指针插入到页面链表`slab_list`的头部。至此，一个新的slab页面已经分配和初始化完成，slab类的容量得到了扩充。

##分配块
`slabs_alloc(size, id)`函数用来在编号为id的slabclass上分配一块空间，其大小至少是size。在调用该函数之前，应该已经通过`slabs_clsid(size)`获得了能够满足要求的slabclass id。该函数首先获得`slabs_lock`锁，然后调用`do_slabs_alloc(size, id)`执行真正的分配过程，它尝试从空闲链表`slots`上获得一块，如果空闲链表为空，则调用`do_slabs_newslab(id)`分配一个新页面后再从空闲链表分配。

##释放块
`slabs_free()`该函数是上述`slabs_alloc()`的逆过程，用来释放一块先前分配的chunk。该函数过程与分配函数基本一致，同样先获得锁，然后调用`do_slabs_free()`进行实质操作，其过程很简单，只要将要释放的块放回空闲链表`slots`即可。

##维护过程
slab的维护和重平衡是Memcached新加入的特性，它在Memcached执行过程中动态维护slab分配器，提升空间的利用率，其本质思想就是将较空闲的slab类的页面移动到较繁忙的slab类中，达到空间的平衡和高效利用，因为slab页面一旦分配后不会再释放，这种重平衡机制可以减小很多情况下的内存占用。

维护过程由两个线程共同完成，一个是`slab_maintenance_thread`，另一个是`slab_rebalance_thread`，在Memcached初始化时这两个线程被启动。

`slab_maintenance_thread`每隔一段时间唤醒一次，唤醒后检查`auto_move`标志以及重平衡的源和目标是否已经准备好。选择重平衡的源和目标是非常重要的工作，这个决策由`slab_automove_decision(&src, &dest)`来完成，决策做出后它会将源和目标slabclass id分别填入`src`和`dst`中。

`slab_automove_decision()`的决策过程不是一次调用函数就完成的，它需要被调用多次，直到选择出了合适的源和目标。它选择源slabclass的策略是找到一个在该函数被调用三次以上后，某个slabclass从来没有发生过块替换，也就是说该slabclass空间比较充足，没有发生淘汰旧块的行为；如果有满足上述条件的slabclass，就将它的id作为候选源。而选择目标slabclass的策略也与淘汰行为有关，它会选择连续三次以上调用期间，块淘汰行为发生次数都为最高的那个slabclass，将id作为候选目标，也就是说被选择的目标slabclass一定是在过去的一段时间内空间已满，并且依然在不断插入新块而不得不淘汰旧块。**需要注意**的是，该函数当且仅当在一次调用过程中同时找到了候选源和候选目标时才会将源id和目标id存入`src`和`dst`中，并返回1，这保证了我们仅需要在既有slabclass空间非常紧张，又同时有另一个slabclass空间宽裕的情况下才真正进行重平衡调整。

一旦调整决策真正做出，`slab_maintenance_thread`会调用`slabs_ressign(src, dst)`开始实施重平衡过程。

接下来，会将`src`和`dst`都存入`slab_rebal`结构中，该结构用来维护重平衡过程中的状态，然后发送`slab_rebalance_signal`信号，该信号会唤醒另一个维护线程——`slab_rebalance_thread`。该线程被唤醒后，开始以一个显示自动机的方式运行，按照start——>remove——>finish的过程逐步进行。

首先是`slab_rebalance_start()`，该函数主要工作是继续初始化`slab_rebal`结构体，选择slab页链表上第一页（最后新建的页）作为要移动的对象，然后初始化移动的起始地址和终止地址等。接着，流程转入到`slab_rebalance_move()`，该函数每次尝试释放`slab_bulk_check`数目的slab块，释放的过程是将一块从slabclass的空闲链表上移除，对于正在被引用的slab块，会在函数下次执行的时候重试，直到这一页上所有的块都被释放完毕。页面所有的块都释放完毕后，该页面成为了空闲的自由页面，这时它就可以被移动到目标slabclass了，自动机转入`slab_rebalance_finish()`过程，该函数首先将空闲页面从`src`中删除，移动到`dst`中，然后对`dst`中的这个空闲页面调用`split_slab_page_into_freelist()`，把它划分为chunk并链入到空闲链表中；这时，重平衡的主要过程已经完成，只剩下一些收尾工作，包括清理`slab_rebal`结构，撤销线程的条件变量等。重平衡完成后，相对空闲的slabclass类中的一页被移动到了相对繁忙且空间不足的slabclass中，使得空间分布变得更加均匀。