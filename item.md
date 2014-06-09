##概览
Item模块相对于HashTable和Slab的抽象层次更高，它使用HashTable和Slab提供的数据结构接口为Memcached提供存储实际数据对象的基础设施。Item的内部结构相对并不复杂，除了定义了一个用于封装数据的结构体`struct item`外，还维护了一个LRU队列，用于支持对象的淘汰机制，其实该LRU队列并不是只由一条链表组成，而是每个HashTable的槽对应于一个链表，所有这些链表的头、尾节点都保存在`heads`和`tails`两个数组中。另外，对于每个LRU链表，都为其分配了一个用于保存统计信息的结构体`struct itemstats_t`，这些结构体保存在数组`itemstats`中。

##struct item 结构

	/**
	 * Structure for storing items within memcached.
	 */
	typedef struct _stritem {
	    struct _stritem *next;      /* LRU队列链指针 */
	    struct _stritem *prev;
	    struct _stritem *h_next;    /* Hash槽中的下一个对象 */
	    rel_time_t      time;       /* 最近访问时间 */
	    rel_time_t      exptime;    /* 超时时间 */
	    int             nbytes;     /* 数据域长度 */
	    unsigned short  refcount;
	    uint8_t         nsuffix;    /* 后缀长度，包括flag和长度字段 */
	    uint8_t         it_flags;   /* ITEM_* above */
	    uint8_t         slabs_clsid;/* 所处的slabclass编号 */
	    uint8_t         nkey;       /* key长度 */
	    /* this odd type prevents type-punning issues when we do
	     * the little shuffle to save space when not using CAS. */
	    union {
	        uint64_t cas;
	        char end;
	    } data[];
	    /* if it_flags & ITEM_CAS we have 8 bytes CAS */
	    /* then null-terminated key */
	    /* then " flags length\r\n" (no terminating null) */
	    /* then data with terminating \r\n (no terminating null; it's binary!) */
	} item;
	
需要注意的是，item结构体中使用了被称为“灵活数组成员”（flexible array member）的C99特性，它被用来定义一个可变长度的结构体，用法是在结构体末尾加入一个0长度的数组成员，如这里的`data[]`，这样就可以在为该结构体分配足够的空间后，通过data域来访问后续可变长度的内容了。在这里的item结构体中，这个可变长度的data域用来存储CAS序号、后缀、数据载荷以及终结符。

一个存储了实际对象的item结构如下图：

![slabclass](https://github.com/Mudenng/inside-memcached/raw/master/images/item-structure.png)

##CAS
Memcached于1.2.4版本加入了CAS语义的支持，CAS即"Check And Set"，它被用来处理多个线程对同一个item并发写而可能造成的问题。CAS的实现很简单，可以理解为版本号，每一个item的每一次写入时都会同时分配一个唯一的版本号，所以，不论是插入新的item，还是对原有item进行更新，都会使得item获得一个新的版本号，这在Memcached中的就是通过一个不断累加的64位静态变量实现cas_id实现的。

在使用CAS之前，两个并发线程的写入过程可能这样：  

1. 线程1获得对象a
2. 线程2获得对象a
3. 线程2修改对象a为对象a2，存入Memcached
4. 线程1修改对象a为对象a1，存入Memcached（覆盖了对象b）
5. 线程2再次读取刚刚存入的对象a2，但此时发现对象a2已经被修改成a1了，发生了冲突

使用CAS之后，该过程不会产生类似的冲突：

1. 线程1获得对象a，以及其cas_id = 1
2. 线程2获得对象a，以及其cas_id = 1
3. 线程2修改对象a为对象a2，写入Memcached前，检查Memcached中对象a的cas_id是否与线程2所持有的对象a拷贝的cas_id相同，此时二者都为1，所以写入成功；Memcached为更新后的对象a分配一个新的cas_id = 2
4. 线程1欲修改对象a为对象a1，写入前检查cas_id，但此时Memcached中对象a的cas_id已经更新为2，而线程1持有的对象a拷贝的cas_id还是1，发生了cas_id不一致，说明在该线程读取-写入这段时间中，有其他线程并发地修改了同一个对象，为了防止产生不一致的现象，线程2的这次写入被拒绝，可以重新读取对象后重试。

##分配item对象
`do_item_alloc(key, nkey, flags, exptime, nbytes, cur_hv)`用来分配一个合适大小的item对象，向其中填入键、标志、过期时间等属性，并为存储data保留足够的空间。

这个函数首先调用`item_make_header()`来为要存储的对象生成“头部”，这个头部包括了标志、data长度等信息，并且返回存储这个对象所需的总空间大小（如果使用CAS，还需要再加上一个64位整形的大小）。接下来使用这个返回的大小调用`slabs_clsid()`来得到能够存储该对象的slabclass编号，接下来我们需要根据编号获得一块相应的slab chunk以存储该对象。

Memcached为每个slabclass都维护了一个LRU队列，用来执行块的淘汰替换机制，这些队列的头部由heads指向，尾部由tails指向。所以这里我们从对应LRU队列尾部开始，查找5次，尝试寻找一个已经过期的对象来替换，或者向slab分配器申请一个新chunk，如果都不行，才淘汰一个LRU队列尾部的对象。

进入对LRU队列遍历的循环体后，首先尝试锁住正在检查的对象，如果不能锁住，说明该对象正在被修改；然后将该对象引用计数增加1后判断是否为2，如果不为2，说明该对象正在被其他线程操作引用。当找到一个没有正在被修改和引用的对象后，判断它是否已经过期，这里过期有两种形式，一种是我们为每个对象单独设置的过期时间exptime，如果当前时间超过了exptime，则该对象可以视作过期；另一种是我们为整个Memcached系统设置的一个对象最久存活时间，如果超过了这个时间也视作过期。如果找到了这样一个过期对象后，就可以用它来存储新的对象，首先需要调用`slabs_adjust_mem_requested()`来更新每个slabclass中已分配空间的统计值，然后调用`do_item_unlink_nolock()`将该对象从HashTable和LRU队列中移除，该对象成了一个自由对象。

当然，找到的这个没有正在被修改和引用的对象有可能尚未超期，此时我们不能覆盖替换它，所以这时候调用`slabs_alloc()`函数向slab分配器请求分配一个空闲的chunk，如果分配成功，我们就可以使用这个新的空闲chunk来存储对象了；如果分配失败，就需要淘汰一个对象了，这里我们淘汰的对象就是最接近LRU尾部的没有正在被修改和引用的对象，更新一些统计数据后，依然是调用`slabs_adjust_mem_requested()`和`do_item_unlink()`。另外，如果开启了slab分配器的"angry bird"模式（slab_automove == 2），在淘汰发生后，立马开始执行一次slab重平衡操作，理由是既然发生了LRU淘汰，说明正在操作的这个slabclass空间已经不足，最好能够从其他较空闲的slabclass移动一些空间来支援该slabclass。

循环结束后，如果我们根本就没有在LRU队尾找到合适的对象，我们就需要调用`slabs_alloc()`来请求空闲块，这里的请求空闲块与上面循环体内的请求并不重复，因为循环体内部的请求空闲块只有在LRU队尾找到一个合适的对象，但那个对象还没有超期的情况下才会发生。

如果这时候我们依然没有得到一个可用的块，说明内存真的非常紧张了，即没有可用替换的超期块，也没用空闲的slab空间可用分配，而且由于我们可能并没有开启LRU淘汰机制，所以这时候Memcached无法满足分配item对象存储空间的要求，只能返回空指针交给上层处理。

最后，不论我们用什么方式获得了一个可用的item对象块，都需要对它做一些初始化和赋值操作，包括重置指针、填入slabclass编号、填入nkey和nbytes、拷贝key到块中、拷贝suffix（flag和size）等操作，最后返回这个item对象的指针，对于data的拷贝会发生在其他地方。

##释放item对象
item对象的释放相对容易，由`item_free()`函数完成，该函数其实是slab分配器中`slabs_free()`的一个封装。整个释放过程如下：检查对象的状态，包括标志位、引用计数等，然后调用`slabs_free()`，这个函数首先为slab分配器加锁，然后将要释放item对象所占用的slot归还到对应slabclass的空闲链表中，更新空闲链表计数以及已分配的空间统计值。

##LRU队列的插入和删除
前面说到Memcached为每个slabclass都维护了各自的一个LRU队列，该队列用来在淘汰替换对象时进行LRU决策，所以，当一个新的对象存储到Memcached中时，就需要将它插入到对应的LRU队列中，这个工作由`item_link_q()`完成；当要从Memcahced中删除一个对象的时候，也需要把它从LRU队列中删除，由`item_unlink_q()`完成。

`item_link_q()`会把新来的对象插入到LRU队列的头部，每个LRU队列的头部指针都保存在`item *heads`数组中，所以只要将新item的指针保存到heads中，然后修改item结构体内部的next和prev两个LRU队列指针即可。当然，由于同时在`sizes`数组中维护了每个LRU队列的长度，所以需要增加对应的长度项。  
`item_unlink_q()`是上述插入的逆过程，将对象从`heads`头部取出，修改next和prev指针，减少长度计数即可。

##插入并生效item对象
当客户端使用set等命令设置一对新的key-value后，Memcached会进行空间分配、key和data拷贝、使新key-value生效等一系列过程。这里我们看看如何使一对新的key-value在完成前面的所有工作后正式生效，也就是能够被客户端读写引用。

由于item是存储key-value的具体数据结构，所以使一对key-value生效也就是使新加入的item生效。这个过程由`do_item_link()`完成，注意不要与上面的LRU队列插入函数`item_link_q()`混淆，`do_item_link()`会调用`item_link_q()`，但是这并不是它的全部工作，除了插入LRU队列，还需要进行一些其他重要的工作，才能使新的item真正生效。

执行的过程首先是设置item的最近访问时间为当前时间，然后增加一些Memcached的统计计数，之后依次完成以下工作：

1. 如果设置了CAS，则为该item生成一个新的CAS序号
2. 把该item插入到HashTable中，这样这个item就可以被get命令找到了
3. 把该item插入到对应slabclass的LRU队列里
4. 增加该item的引用计数

删除并失效一个item对象恰好也是上述过程的逆过程，包括从HashTable中删除，从LRU队列中删除，最后还需要调用`do_item_remove()`，这个函数用来减小item的引用计数，如果引用计数变为0，则调用`item_free()`释放该item占用的slab块。

##更新item访问时间
`do_item_update()`函数用来更新一个item对象的最近访问时间，这个操作的主要作用是访问时间的更新会同时影响对象在LRU队列中的位置，使得对象不会被优先淘汰。Memcached不允许我们频繁地更新一个对象的访问时间，最小更新间隔为60秒，更新的过程比较简单，首先从LRU队列中将对象移除，然后用当前时间来更新对象的最近访问时间，最后把对象重新插入到LRU队列的头部。

##do_item_cachedump
`do_item_cachedump(slabs_clsid, limit, *bytes)`函数的作用是将id为slbas_clsid的slabclass中的有效对象以LRU队列的顺序dump到buffer中，dump的内容包括key、data的长度、超时时间，不包括data数据本身。可以指定limit参数来限制dump对象的数目。函数的实现较简单，就不展开解释了。

##do_item_get
该函数是对HashTable中`assoc_find()`的封装，它用来使用指定的key去查找一个对象，除了实现对象的查找功能，这个函数的另一个重要用途是实现了Memcached中对象的惰性淘汰机制。

在Memcached中，我们可以为每个对象指定一个过期时间，也可以为所有对象指定统一的存活时间，当超时后需要将对象淘汰出缓存，Memcached实现了惰性淘汰的机制，意思是我们并不主动去逐一检查每个对象是否超时（除非在分配新对象时空间不足，需要选择一个对象进行淘汰），这样可能消耗大量的CPU资源，我们仅在请求查找获取对象时才检查要获取的对象是否超时，如果超时，就将它清除并返回NULL。

在很多不同的应用场景下，我们都可以看到某种“惰性”机制，比如内核中的请求调页、copy-on-write，甚至是函数式语言中常见的惰性求值，这些巧妙的惰性机制在正确实现逻辑的同时节约了大量的计算资源或者拥有其它优点，可以说“惰性”是计算机科学中一个非常经典的优化方法。

##do_item_flush_expired
除了以上的惰性淘汰机制，Memcached其实也提供了手动调用去淘汰所有超期对象的接口，这个函数会在每个slabclass的LRU队列上执行循环，删除所有超过全局存活时间的对象。由于LRU队列本身就是按照访问时间排序的，所以只需从头到尾遍历LRU队列即可，O(n)的复杂度。