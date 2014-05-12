# Memcached 源码分析 - 内部数据结构之Hash Table

##概览
Memcached 使用了Hash Table对存储的对象进行索引，其实现简单平凡，使用了最常见的策略，包括链式冲突解决、扩容操作等。

##结构
Memcached中的Hash Table包括了2^`hashpower`个槽（hashpower默认为16），槽中存储的每个对象都是指向`item`结构的指针（`item`是memcached中实际存储数据的结构）。

	/* how many powers of 2's worth of buckets we use */                           
	unsigned int hashpower = HASHPOWER_DEFAULT;

Memcached实际定义了两个指向hash table指针：`primary_hashtable`和`old_hashtable`。正常情况下，`primary_hashtable`指向的是我们正在使用的main hash table，所有的查找、插入、删除等操作都其上进行；当扩容操作发生时，将`old_hashtable`指向扩容前的`primary_hashtable`，然后为`primary_hashtable`分配新空间，由扩容线程将`old_hashtable`中的所有项拷贝到新`primary_hashtable`中。在扩容拷贝进行的过程中，如果发生hash table的操作，将通过判断需引用的对象是否已经拷贝到新table中而分别在`primary_hashtable`或者`old_hashtable`上进行操作，这样就可以做到在扩容的过程中对于hash table的正常操作依然可以不受影响地进行。

	/* Main hash table. This is where we look except during expansion. */
	static item** primary_hashtable = 0;

	/*
 	* Previous hash table. During expansion, we look here for keys that haven't
 	* been moved over to the primary yet.
 	*/
	static item** old_hashtable = 0;
	
##初始化
`assoc_init`用于初始化操作，非常简单，按照参数或者默认值为`primary_hashtable`分配空间，更新`stats`信息后结束。注意，hash table仅占用`sizeof(item *) * hashsize(hashpower)`的空间，也就是每个槽的链表的头结点空间，而链表中的其他链指针则保存在每个`item`对象中，由`item`对象负责空间的分配和释放。

##查找
`assoc_find`函数负责hash table查找，它返回指向`item`的指针（若查找失败则返回NULL）。需要注意的是，在选择hash value对应的槽的时候，通过判断`expanding`这个bool值来确定现在是否正处于扩容操作中，如果正在扩容并且待查找的槽还没有被拷贝到新hash table中，则需要在`old_hashtable`中查找，否则应该在`primary_hashtable`中查找。

##插入
`assoc_insert`执行插入操作，注意该插入操作不包括更新的语义，即插入前hash table中不应包含具有同样key的对象。插入的过程也很简单，在`old_hashtable`或者`primary_hashtable`中找到对应的槽后，将新对象插入到槽链表头部。  
插入成功后，需要判断是否要进行扩容操作，当装填因子大于1.5的时候，则调用`assoc_start_expand()`唤醒扩容维护线程。_扩容过程分析见下文_。

##删除
删除操作通过`assoc_delete`完成，该函数首先调用内部函数`_hashitem_before()`获得指向待删除item的前一个元素的指针，然后执行删除操作。在内部函数`_hashitem_before()`中有一个非常经典的二级指针的应用，使得不需要维护prev指针即可找到before元素，这个用法类似于Linus大神曾经为了说明内核开发者应该具有的代码能力列举的一个例子，具体内容和分析可见CoolShell的一篇文章 <http://coolshell.cn/articles/8990.html>

##扩容
Memcached初始化时会启动一个`assoc_maintance_thread`线程，负责适时唤醒对hash table进行扩容操作。其唤醒通过`maintance_cond`条件变量控制，进行插入操作后，装填因子大于1.5时，激活该条件变量，扩容线程被唤醒。线程唤醒后扩容操作步骤如下：  

1. 暂停slab分配器的重平衡线程，以免发生错乱
2. 将对item访问所需的锁类型切换为`ITEM_LOCK_GLOBAL`，即全局锁，因为在扩容时，如果要进行hash table的动态操作，需要锁住整个hash table
3. 用`old_hashtable`指向原`primary_hashtable`，为新`primary_hashtable`分配空间，初始化`expand_bucket`为0，该变量表示已经完成扩容的最大槽编号
4. 批量移动`hash_bulk_move`个`old_hashtable`的槽到新的`primary_hashtable`，移动前需要获得item全局锁，一次移动后释放锁，这样不会因为扩容线程持续占有锁而导致Memcached长时间无法响应请求。当所有的槽移动完成后，释放`old_hashtable`的空间
5. 扩容操作完成，调用`slabs_rebalancer_resume()`重新启动slab重平衡线程，hash table扩容线程自身则进入休眠等待下一次唤醒