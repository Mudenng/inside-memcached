# Memcached 源码分析 - 处理流程之响应新客户端连接

在启动过程代码中，已经为每个用来监听客户端连接的TCP套接字或者UNIX域套接字创建了一个`conn`结构体，并且把这些套接字都加入到了主线程`main_base`的event loop中，这样，发生在这些监听套接字上的事件都会触发`main_base`上的一个event，该event的处理函数为`event_handler()`，函数参数是与套接字对应的`conn`结构体。所以，一个新的客户端连接到来后，主线程会调用`event_handler()`处理函数，这也是我们分析的入口点。

##处理流程
###主线程
`event_handler()`是`drive_machine()`的一个封装，它只做了少量检查工作后就调用`drive_machine()`，而从名字就可以看出，接下来的工作是“驱动一台机器运转”，这台“机器”其实就是一个显式自动机。Memcached将所有命令的处理过程都表达成了一个统一的自动机，所以只要理清了这个自动机的状态转换过程，也就理清了Memcached的处理流程；而自动机的状态保存在`conn`结构体的`state`域中，`drive_machine()`将根据每个连接的state决定将要进行的工作。

由于在启动过程中，已经将每个监听套接字的conn->state设置成了`connect_listen`，因此`drive_machine()`将进入第一个case分支：

1. 首先调用`accept()`接受客户端连接，然后调用`dispatch_conn_new()`将这个新连接分配给一个worker thread
2. 分配时首先创建一个新的`CQ_ITEM`结构体，该结构体是一个用来传递连接的临时结构体，用来组成每个工作线程的新连接队列，结构体中有一个`init_state`成员用来表明该连接所要进入的初始状态，在这里初始状态为`conn_new_cmd`
3. 选择一个工作线程，这里的选择方法是round-robin，使得每个线程都能够被轮流选择
4. 将`CQ_ITEM`结构体送入选择线程的连接新队列中
5. 最后，向线程的通知管道中写入一个字符'c'，来触发对应线程的event，表明有新连接到达

###工作线程
前面已经了解，在工作线程初始化时，已经将线程自己的通知管道描述符加入到了各自的event base中，并且绑定其处理函数为`thread_libevent_process()`。所以，当主线程分配新连接并向某个工作线程的通知管道里写入字符'c'后，工作线程被唤醒，进入`thread_libevent_process()`处理：

1. 由于通知管道中被写入了一个字符'c'，所以进入第一个case分支；首先从线程的新连接队列中将主线程分配的新连接临时结构体弹出
2. 调用`conn_new()`来为新连接创建一个重要的`conn`结构体，该结构体将用来维护该客户端连接的所有状态和数据。`conn_new()`会初始化结构体中的很多成员，包括为其中的buffer分配空间，并且会把`c->state`设置为`init_state`，在这里为`conn_new_cmd`
3. 最后，将这个新的客户端连接描述符加入到线程的event base中，并且设置其event处理函数为`event_handler()`，所传递的参数是`conn`结构体。这样，这个新的连接就创建完成了，被选中的这个线程以后将全权负责与新客户端的通信，为其提供Memcached服务

##总结
总结一下整个处理流程：新客户端连接将唤醒主线程，主线程调用event处理函数`event_handler()`，接受该连接，然后主线程选择一个工作线程，并把连接推入到选中线程的新连接队列中，在选中线程的通知管道中写入字符'c'。之后工作线程由于通知管道的写入事件被唤醒，进入`thread_libevetn_process()`处理，创建一个`conn`结构体，初始化之。最后把这个新客户端连接的描述符添加到自己的event base中，设置event处理函数为`event_handler()`。以上工作完成后，被选中的工作线程将对新连接的客户端进行响应，提供服务。