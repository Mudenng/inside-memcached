#Memcached 源码分析 - 处理流程之GET请求

在Memcached对客户端的请求进行预处理之后，如果发现是`get`、`bget`或`gets`请求，则会在`process_command()`中调用`process_get_command()`开始真正的处理。

##process_get_command
在该函数中，先对key的长度等进行合法性检查，然后调用`item_get()`去查找该key所对应的item，如果该item未过期，则返回它。

接下来需要构造响应消息，消息的格式为：

    "VALUE "
    key
    " " + flags + " " + data length + "\r\n" + data (with \r\n)
    
如果是`gets`命令，也就是说需要返回CAS，则需要将CAS标记添加到`data`之前。

对于响应消息的每一部分，都调用`add_iov()`把数据添加到conn的输出缓存链表中，这里并没有发生真实的数据拷贝，只是将数据指针添加到`msghdr`中的`msg_iov`链表中，在发送数据时直接使用它们。

在`add_iov()`中，很大一部分工作都用来处理数据包大小的问题了，如果使用的是UDP协议，则我们需要将每个`msghdr`所包含的数据包大小限制在UDP的允许范围之内(UDP_MAX_PAYLOAD_SIZE)，如果超过了这个大小就需要分割数据，将其放到不同的`msghdr`中。

响应消息构造完成之后，就需要进行一些收尾工作了。由于我们刚刚访问了一个item，所以需要调用`item_update()`来更新它，主要是更新对象的最近访问时间，并将其移动到LRU队尾；另外，需要将这个对象添加到conn的ilist链表中，该链表的成员是这个conn上准备write out的item对象指针。

由于一条get命令可以同时请求多个对象，所以以上的工作在一个while循环中，对于get命令请求的每个key都进行这种操作后，循环退出。

最后，如果一切正常，则切换状态到`conn_mwrite`，准备进行网络层面的传输工作。

##conn_mwrite
以上处理过程返回后，自动机将进入下一次循环，进入到`conn_mwrite`分支，在该分支下将调用`transmit()`一次传送一块msglist上的数据，根据返回的状态，如果还有待传送的数据（TRANSMIT_INCOMPLETE），则继续下一次自动机循环并进入相同的分支，否则（TRANSMIT_COMPLETE）进行收尾工作后切换状态。

在`transmit()`中，每次都调用`sendmsg()`以conn的msglist链表项为单位进行传送。处理的过程主要是一些指针和计数的移动，就不再赘述了。如果一次传送完成后，msglist中还有待传送数据，就返回TRANSMIT_INCOMPLETE，否则返回TRANSMIT_COMPLETE，当然也有可能传送过程出现了错误，则返回相应的错误状态。

回到`conn_mwrite`分支中，每次`transmit()`调用后都会判断其返回状态，如果是INCOMPLETE，则保持conn_mwrite状态开始下一次循环，继续传送；如果是COMPLETE，则进行传输后的收尾工作，包括：

- 减小本次get请求获得的所有对象的引用计数（它们的引用计数在`item_get()`的时候被增加了），
- 释放用来存储CAS标记的缓存

最后，将conn切换回`conn_new_cmd`状态，准备处理下一条请求；如果当前客户端连接上没有更多的请求或者已经处理了该客户端足够多的请求，则线程退出自动机循环，处理其他客户端触发的event事件或进入睡眠等待event。