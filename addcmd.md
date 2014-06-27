# Memcached 源码分析 - 处理流程之ADD/SET/REPLACE/CAS请求
与GET请求一样，请求预处理后，对于`add`、`set`、`replace`、`cas`等写入操作的请求，会进入到`process_command()`的相应分支中调用`process_update_command()`进行处理。

##process_update_command
该函数负责所有添加/更新key-value相关的操作，函数首先从请求指令的token中解析出key、key length、flags、exptime、data length等，然后调用`item_alloc()`分配item对象，对于`item_alloc()`的详细分析见“内部数据结构之ITEM”。总之，这里如果分配成功，会将以上解析出的字段填入分配的item中，然后返回该item的指针；如果分配失败，说明内存不足，需要将状态设置为`conn_swallow`，该状态用来读取输入缓冲区中剩下的数据并丢弃它们。

接下来，如果请求的是`cas`指令，则将请求cas标记也填入item中。最后，切换状态到`conn_nread`，进入下一次自动机循环。

##conn_nread
进入到该状态后，首先从`conn`的输入缓冲区中读取数据到刚刚创建的item的data字段中，如果从输入缓冲区中读取的字节数没有达到指定的data length，说明还可能有部分数据没有从socket中读出，所以需要继续调用`read()`函数从socket中读取数据，直到item的data数据长度已经达到了data length，此时将进入`complete_nread()`。

##complete_nread
此时，我们已经通过以上的操作生成了一个基本完整的item对象了，并且对象的data字段也已经填好，接下来就需要根据不同的命令语义进行后续处理了。

对item的完整性进行检查后进入到`store_item()`中，顾名思义，这里就是要把刚刚生成的这个item真正“存储”下来，注意如果已经存在相同key的对象，则根据不同的命令语义需要不同的处理：

- 对于`add`命令，如果已经存在相同key的对象，则调用`do_item_update()`更新已存在的对象，把它移到LRU队尾；如果不存在，则在最后一个else分支中调用`do_item_link()`把新的item添加到HashTable和LRU队列中
- 对于`replace`、`append`等命令，显然是对原有对象的更新操作，而如果查询到原来并不存在一个具有相同key的item，则这里什么操作都不做；否则，对于`replace`命令，在最后的else分支中直接用新item替换原有item即可，而对于`append`命令，在检查CAS后（如果需要）创建一个原有对象的副本，并对它进行修改，然后删除原对象，插入新对象。
- 对于`cas`指令，对比请求中所包含的对象的cas标记与原有对象的cas标记，如果它们相同，则可以调用`item_replace()`用新item替换旧item；如果不同，说明该对象已经被修改，此次操作被拒绝。

##反馈结果
在上述对象生成和存储操作完成后，需要将操作的结果返回给客户端，这里使用`out_string()`向客户端反馈信息。

反馈的信息是一个字符串，首先构造一个`msghdr`，然后将字符串拷贝到conn的输出缓存`wbuf`，接着切换状态到`conn_write`，并把输出完成后的状态`write_and_go`预先设定为`conn_new_cmd`，意思是待自动机完成`conn_write`状态的处理后，就立刻进入处理下一个请求的流程中。

在`conn_write`状态下，组装一个简单的response后，直接fall through到`conn_mwrite`中，因为这里的反馈字符串和get命令所返回的key-value本质上都是字符串，所以接下来与get命令最后的传输过程一样，将response输出到连接socket上，只不过这里输出的仅仅是一条更简单的字符串而已（get命令输出的是由key-value组成的复杂格式化字符串）。