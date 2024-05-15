### 1 线程池是什么？解决了什么问题？

线程池管理固定数量的多个线程，负责创建、挂起、唤醒进程。（原因是机器资源有限）     <br>

1 单生产者、多消费者场景：针对耗时操作、阻塞操作。生产者抛出任务到队列中(不需要立刻知道结果)，消费者从队列中取出任务处理，部分线程处理任务后会将结果放到另一个队列中，通过消息的方式通知生产者，然后生产者从队列中取出结果。     <br>

任务：上下文、callback(执行任务的方式)   <br>
任务队列：解决生成者和消费者(线程与线程之间)的耦合，线程与线程之间传递数据的媒介(选择队列的原因是方便加锁)。   <br>
         lock(&mtx) --- 操作临界资源 --- unlock(&mtx)   【临界资源执行时间越短，加锁效率越高】    <br>
线程调度(管理)：根据队列状态进行管理，从无到有、唤醒休眠的进程；从有到无，让线程休眠   <br>

2 多生产者，多消费者场景：充分利用系统资源(cpu)，并发执行无差别的任务。可能会有多个队列，生产者把任务放入其中一个队列中，消费者取出任务来处理。     <br>

<br/>

### Redis io线程池工作原理以及解决的问题

redis业务处理(命令处理)是 **单线程**    <br>
什么时候需要使用io线程池：多条并发连接与redis进行交互时，并且io操作比较耗时时，需要考虑io多线程(io线程池)     <br>

命令处理的步骤：    <br>
1 感知到客户端给redis发送数据;     <br>
2 从内核协议栈中读出客户端发送的命令;   <br>
3 解析协议     <br>
4 执行命令    <br>
5 得到命令处理结果，加密协议    <br>
6 发送处理结果     <br>

io线程池主要处理2、3(read)步，5、6(write(send))步的问题，read()、write()是系统调用，会涉及io操作。第4步都是在主线程中处理。       <br>

【ps：内核协议栈专门处理网络数据。首先 网卡 接收到网络数据(bit数字)，通过 网卡驱动 写入到环形缓冲区(在内核)，然后网卡驱动会从环形缓冲区读数据，读完后按照TCP/IP协议栈依次往内核协议栈(在内核)传数据——具体来说就是，首先数据链路层取出mac地址，到IP层，到TCP层，定位到具体的socket，每个socket都会有缓冲区，数据会写入缓冲区中。然后用户层会通过一些方式感知到内核的socket缓冲区中有数据，然后通过read()把数据读取出来，然后数据就会到达用户态，就可以进行处理了。】    <br>

全局队列的作用：收集io任务  <br>
每个线程需要有一个队列：避免锁竞争  <br>
主线程作为io线程的一部分：充分利用系统资源，主线程如果不处理io就会一直等待，直到其他io线程处理结束     <br>

redis有一个主线程(负责命令处理)，当感知到数据时，先不进行2、3步。先把数据抛到io线程池，io线程池中有多个io线程(主线程也会作为线程池的一部分，参与线程池的工作)。    <br>
详细来说就是，主线程会把数据先放入 全局队列，接下来每个io线程都会准备一个队列，然后在全局队列当中进行负载均衡(全局队列中的任务分配到io线程的队列中，采用round-robin(时间片轮转（RR）调度算法，抢占式CPU调度算法)的方式进行分配)   <br>

```
aemain()【ae.c】  网络处理的事件循环 ---> aeProcessEvents()  处理事件
int aeProcessEvents()
{
    ...
    //numevents  同时感知多少事件。以fd为单位，同时有多少条连接，最多就能感知多少个事件
    numevents = aeApiPoll(...); //io多路复用的封装，感知具体io的就绪事件(比如客户端发送的数据)，对应linux系统的epoll
    ...
    fe = &eventLoop->events[fd];
    ...
    fe->rfileProc(...); //触发读的回调，这里面进行2、3步
    ...
}

createClient()【networking.c】  客户端与服务器连接后，服务器接收连接，创建Client，并设置读的回调。 ---> readQueryFromClient() 读的回调
{
    ...
    if (postponeClientRead(c)) return; //如下所示
    ...
    nread = connRead(...); //对应第2步，读客户端发送的命令，调用read()，从协议栈中把数据读到用户态
    ...
    if (processInputBuffer(c) == C_ERR) {...} //对应第3步，进行协议解析
    //processInputBuffer()方法中的 processInlineBuffer()、processMultibulkBuffer() 负责解析协议
    ...
}
【注：执行redis具体命令的方法中，有setGenericCommand()，负责第5、6步】

int postponeClientRead() 如果开启了io多线程，在这个接口中收集连接，放到全局队列中
{
    ...
    listAddNodeHead(server.clients_pending_read, c);//clients_pending_read是全局队列，c是连接
    ...
}

int handleClientsWithPendingReadsUsingThreads()   把全局队列中的连接数据分发到io线程队列
{
    ...
    int target_id = item_id % server.io_threads_num; //通过取余的方式，也就是rr调度算法来分配
    listAddNodeTail(io_threads_list[target_id], c); //io_threads_list是数组，就是io线程的队列，
                                                    //将全局队列中的任务，依次分配给每个io线程的队列当中
    ...
}
【注：io_threads_list[0]是主线程，主线程也会作为io线程，去处理一部分任务，处理完后等待其他io线程处理结束】


io线程入口函数：
void *IOThreadMain()
{
    ...
    if (getIOPendingCount(id) != 0) break; //发现队列中有任务
    ...
    listRewind(io_threads_list[id], &li); //分别把任务取出来，开始执行任务
    ...
    setIOPendingCount(id, 0); //io线程已经完成任务，阻塞io线程
    //之后就把执行权让给主线程，让主线程执行命令了
}

```


### skynet 线程池工作原理以及解决的问题





### workflow 线程池工作原理以及解决的问题





