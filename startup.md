
# Memcached 源码分析 - 启动初始化分析

##概览
Memcached的启动过程代码位于memcached.c下的main函数中，整个main函数有800行代码之多，可见在启动过程中，Memcached完成了很多的工作，理解启动代码以及这些工作背后的意义将非常有助于我们理解整个Memcahced的体系结构和主要逻辑，另外，对于并不是很庞大的开源代码来说，从启动过程入手来分析源码是一个很好的途径。

我们可以将整个启动过程大致分为五个部分，分别是：设置运行时环境、初始化数据结构、初始化线程、配置网络、开始主循环。下面对这五个过程一一进行分析。

##第一部分——设置运行时环境
1、设置SIGINT信号的处理函数，其实该处理函数只是加了一条打印输出，然后程序退出。 
 
2、配置默认参数，Memcached的所有可配置参数都保存在全局`settings`结构体里，这里调用`settings_init()`为这些参数设置默认值。  

3、分析命令行参数，使用这些指定的参数来覆盖默认值。  

4、如果通过'-l'参数指定了多个监听地址，则为每个UDP分配一个worker thread，否则所有的worker thread都在一个端口上监听。  

5、增大core文件上限，core文件是用来调试的，当程序崩溃时，内核会将程序当前内存映射到core文件中。  

6、如果当前用户是root，则调用setuid和setgid切换到其他用户，这是出于安全的考虑，除非使用特定的'-u'参数，否则不能以root身份运行Memcached。  

7、如果要以daemon运行，则首先设置忽略SIGHUP挂起信号，然后调用`daemonize()`来将当前进程变为守护进程。`daemonize()`的原理就是首先fork出一个子进程，然后父进程退出，子进程继续运行，并且将子进程的标准输入输出重定向，此时子进程就成为了一个守护进行，它的父进程变为了pid = 1的进程。  

8、如果设置了‘-k’参数，则调用`mlockall(MCL_CURRENT | MCL_FUTURE)`将进程地址空间锁定在物理内存中，使之不会被换出，这里的`MCL_FUTURE`参数保证以后malloc的内存同样会被锁定。  

##第二部分——初始化数据结构
1、初始化libevent的event base，这里的`main_base`为主线程监听连接请求的base，其他工作线程都有各自的event base。  

2、初始化`stats`，也就是对Memcached运行过程中需要记录的统计数据初始化。  

3、初始化HashTable，如果没有设置参数，那么默认有2^16个桶。  

4、初始化`conn`结构体指针数组，默认支持200个连接，该结构体将用来保存和传递客户端连接。  

5、初始化slab分配器，slab默认占用最多64M内存，默认的factor为1.25。若开启大页支持，则会让slab预先分配内存池，并为每个slabclass预分配一页。这里我们应该分清几个slab设置参数的意义：

名称 		   	| 参数选项 	| 含义
------------  	| ---------	| ------------
chunk_size 		| -n  		| slab chunk的最小大小，默认为48字节
item_size_max 	| -I（大写i）| 存储item的最大大小，48 Bytes ~ 1M(默认)
maxbytes 		| -m 		| slab的最大可用内存（也是Memcached的最大可用内存），默认为64M

##第三部分——初始化线程
运行时环境和数据结构初始化完毕之后，我们就可以开始初始化并启动所需的线程了：

1、main函数首先调用`thread_init()`来启动worker thread，worker thread的数量可以由参数‘-t’指定。

- 这个函数首先初始化一些锁，其中包括最重要的`item_locks`，当我们并发访问某个item时，锁住其实是该item所在的HashTable槽，所以理论上应该为每个槽分配一个锁，但是为了减少内存浪费，这里分配的锁数目根据线程数目有所不同，不过都少于默认的HashTable槽数，所以在加锁时还需要对锁数目取余。

- 接下来，需要为每个线程分配一个数据结构`struct LIBEVENT_THREAD`，该数据结构聚合了线程函数所需的所有信息和数据：

		typedef struct {
    		pthread_t thread_id;        		/* 线程ID */
    		struct event_base *base;    		/* 该线程的event base */
    		struct event notify_event;  		/* 监听消息管道的event */
    		int notify_receive_fd;      		/* 消息管道的接收端 */
    		int notify_send_fd;         		/* 消息管道的发送端 */
    		struct thread_stats stats;  		/* 该线程的统计信息 */
    		struct conn_queue *new_conn_queue;  /* 需要处理的连接队列 */
    		cache_t *suffix_cache;      		/* suffix缓冲区 */
   			uint8_t item_lock_type;     		/* 锁粒度指示 */
		} LIBEVENT_THREAD;
		
- 然后为`dispatcher_thread`数据结构初始化，这个结构比上面worker thread的数据结构简单很多，只有线程ID和event base两项，可以看到这里将event base指向了`main_base`，也就是启动时main函数创建的base，而线程ID也填为当前唯一线程（主线程）的ID。所以这里所说的`dispatcher_thread`其实就是主线程，主线程在运行时将负责分发任务到各worker thread

- 然后给每个worker thread创建一个PIPE，这个管道是用来接收其他线程发来的消息，比如主线程将一个任务分配到某个worker thread后，可以通过这个管道通知该worker thread

- 接着调用`setup_thread()`为上述所说的每个线程的数据结构初始化。首先，每个线程都创建自己独立的一个event base；然后将消息管道的接收端挂到自己的event base上，这样当有其他线程通过这个管道发消息给自己时，产生一个notify_event，自己将被唤醒并处理；最后，初始化连接队列、锁、cache等。

- 完成上述初始化后，我们就可以启动各线程了，这里调用`create_worker()`来启动各个worker thread，该函数其实就是POSIX线程的`pthread_create()`的封装，可以看到，各worker thread的线程函数为`worker_libevent()`，传递的参数就是每个线程的`struct LIBEVENT_THREAD`结构。

- 这时主线程会阻塞等待各个工作线程完成最后的初始化工作，这些工作可以在各线程的处理函数`worker_libevent()`中进行，完成后线程需要调用`register_thread_initialized()`来注册，其实就是通知主线程自己已经初始化好了；最后，线程开始进行event loop，等待事件发生。

- 当主线程发现所有的worker thread都完成了初始化并开始运转后，`thread_init()`函数就可以返回了，到这里，worker thread已经全部创建成功并进入event循环。

2、worker thread创建完成后，还需要创建两个特殊线程，一个是HashTable的扩容维护线程，一个是slab的重平衡线程。这两个线程已经在之前的数据结构解析中分析过了，这里就不再复述。

##第四部分——配置网络
初始化的最后一步是配置网络，创建套接字并监听。

1、如果指定了‘-s’参数，则使用UNIX域套接字，该协议用于本地客户端/服务器通信，是一种IPC的方式。这种协议的优点是既可以让我们使用和因特网套接字一样的接口，又省去了协议处理的部分，包括添加报头、校验和等，提高了效率。它的初始化过程与普通因特网套接字类似，首先创建一个socket fd，然后把它bind到一个本地套接字文件上，该文件是用来通知客户端该套接字名称的，最后调用listen监听该套接字即可。上面的创建过程完成后，Memcached会新建一个`conn`结构体与刚创建的socket绑定，这个过程由`conn_new()`完成，该函数创建并初始化一个conn结构体，然后与socket绑定，最后把socket事件加入到指定的event base中，指定event处理函数是`event_handler()`，所传递的参数是这个conn结构体本身。可以看出，conn结构体的用处就是维护每个客户端连接的状态，保存和传递整个请求处理周期中用到的数据。  
这里，初始化部分为新建的UNIX域套接字创建了一个conn结构体，并在主线程的`main_base`中监听套接字上的事件，创建的这个conn结构体由全局变量`listen_conn`指向，由名字可以看出，这个conn的工作就是监听新客户端连接的到来。

2、对于另外两种协议类型TCP和UDP，总体思路与UNIX域套接字类似，主要工作都是创建新的socket，然后分配一个conn结构体与socket绑定，最后在`main_base`上监听socket事件。不同的是，由于我们可以同时指定多个端口，所以需要为所有的端口都各自创建一个socket和conn，这些conn以链表连接起来，`listen_conn`指向链表头部。

##开始主循环
至此，Memcached所有初始化都完成了，具备了为客户端服务的能力，所以主线程调用`event_base_loop(main_loop, 0)`开始了在`main_base`上的主循环，等待事件的到来；而其余各worker线程其实也早已进入了各自的event循环，等待主线程分配任务。
