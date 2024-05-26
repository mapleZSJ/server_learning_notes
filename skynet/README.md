skynet：   轻量级服务器框架    actor框架+基础组件 网络(基于reactor模型)接入方案  lua虚拟机实现抽象的进程   时间轮(定时)  <br>
【不要使用共享内存进行通信，而要通过通信来共享内存】     <br>

<br/>

### 三个核心要素
actor、消息、携程    <br>

抽象actor并发模型 --- 抽象进程（用户层的进程） --- 独立的运行环境     <br>

actor:      <br>
lua虚拟机：通过 lua虚拟机 在用户层构建【也就是说每个抽象进程都是lua虚拟机、都是独立运行的】     <br>
消息队列：通过 消息 驱动运行，消息队列按顺序存放消息              <br>
回调函数：为每个类型的消息设置 回调函数，从消息队列中取出 消息 作为参数去 调用回调函数，运行环境是在lua虚拟机中，通过消息驱动  <br>


消息:    <br>
网络消息：某个fd消息必然会路由到某个actor中去处理，每个类型的消息都有对应的actor    <br>
actor之间发送消息：通过消息共享、交换数据   <br>
定时消息：处理某些延时任务      <br>



协程:       <br>
实现同步非阻塞的逻辑【使用消息为驱动方式会涉及到 异步调用】   <br>
要实现 同步，要阻塞一个执行体，不能阻塞线程，所以选择阻塞协程   <br>
满足条件 唤醒、不满足条件 挂起  ---> 比如有完整的数据包，才会开展业务逻辑  <br>
【例：actor向redis发送消息请求数据后挂起，当redis返回数据后唤醒actor（是异步的流程）。通过 协程 可以同步处理数据（如：local name = redis:hget(...)）;而不用注册回调函数，使得获取数据后要在另一个业务逻辑中去处理】   <br>
            


通信会涉及到actor与actor之间的通信、以及网络模块与actor之间的通信：   <br>
网络模块一般是reactor网络模型（非阻塞IO），也就是异步事件的模型。网络会发送事件，将事件包装成消息后，插入到具体的actor的消息队列中;   <br>
抽象进程本质是在一个进程中，actor给另一个actor发送消息，本质上是指针的移动 ---> 把actor产生的消息的指针插入到另一个actor的消息队列中;    <br>
如果有定时任务，会有专门的定时线程。actor抛一个具体的定时任务，定时线程仅知道这个任务，过期时定时线程会抛消息给具体的actor;   <br>


<br/>


### actor编程思路


怎么基于actor模型实现业务  <br>
1抽象具体业务对应actor     <br>
2该业务的计算量（消息交互频次怎么样）   <br>

1 actor确定：   <br>

功能独立性，可以单独测试    <br>
大致估计运算的密集程度；如果某个业务逻辑很多，可以拆分到多个actor中，可以充分利用多个线程(线程去调度actor)，充分利用多核    <br>
考虑lua gc压力：存放对象越多，gc压力越大   <br>

【例】猜数字游戏：满3人开始游戏，每人猜测规定范围内的数字，不能退出，猜中后结束    <br>
抽象业务逻辑对应actor:   <br>
actor：多个玩家(每个玩家对应一个actor)、一个大厅(玩家准备，大厅满3个人创建房间，把三个用户数据导到房间actor)、一个房间(玩游戏的过程中，玩家actor直接把数据转发到房间actor)   <br>


2 接口设计：    <br>

agent(玩家)：     <br>
职责  ---  游戏登录（游戏登入、游戏登出）；  游戏操作（准备游戏、猜数字）  <br>
接口(暴露给网络等调用)  ---  登录接口、游戏准备接口（委托给大厅实现）、猜数字接口（委托给房间实现）    <br>

hall(大厅)：      <br>
职责  ---  匹配规则、创建房间    <br>
接口  ---  游戏准备接口（实现匹配规则）   <br>

room(房间)：      <br>
职责  ---  实现游戏规则   <br>
接口  ---  猜数字接口、登入接口（玩家上线、在线）、登出接口（玩家掉线）、广播接口（广播玩家状态，比如轮到谁猜）   <br>


【首先main.lua文件（一个actor），先监听端口，启动唯一的redis服务和hall服务(进程)，向监听端口注册回调函数，当有连接建立时就会调用回调函数。回调函数中会接受连接，为连接创建一个agent进程(actor)。    <br>
agent进程中，将连接与agent进程绑定(通过clientfd，保证这个连接发送的所有消息都会路由到这个actor去处理)。创建一个协程去处理网络消息，从消息队列中读出完整数据包后，业务逻辑分发(登录、操作等)。登录(redis没有用户数据则注册)，游戏准备(消息发送到大厅去处理，大厅处理完后返回状态)，猜数字(发送到房间去处理)。    <br>
hall进程(actor)，实现匹配规则，准备一个队列，每次有用户准备都插入队列中，当队列长度大于等于3时，创建房间，然后取三个用户推到房间。   <br>
room进程(actor)，实现游戏规则，把三个玩家放到房间，生成需要玩家猜的随机数，随机某个人开始猜数字；实现猜数字逻辑；处理掉线、上线的情况。】   <br>


怎么实现业务逻辑的分发？   <br>
1 网络协程 不断从协程中拿取网络数据   <br>
2 根据命令查表的方式分发业务逻辑    <br>

```
agent.lua
skynet.start(function()
    skynet.fork(process_socket_events)  --启动一个协程，专门用来处理网络交互命令
end)

function process_socket_events()   --网络协程
    local data = socket.readline(clientfd)  --不断从连接中取网络数据
    --接下来就是以空格作为分割符，分割data取命令cmd和参数，根据cmd调用到相应的方法
    skynet.fork(CMD[cmd], ...)  --分发逻辑，根据cmd调用对应的方法
end

【注：actor之间的消息发送（如：agent向hall发送消息）
skynet.send()  不需要知道结果
skynet.call()  需要知道结果

例：
【agent】skynet.call(hall, "lua", "ready", client)
【hall】skynet.dispatch("lua", function(session, address, cmd, ...)   --接收其他actor发送过来的消息
end)
】
```

其他：   <br>
Actor - 本质上是做功能抽象  <br>
优化：当玩家多时，一个actor处理多个连接(玩家)【agent_pool，实现一个负载均衡的算法，管理多个agent的进程，提前创建actor】，根据不同的客户端id，创建结构来存储玩家数据；一个actor处理多个房间    <br>
skynet通过newservice()、uniqueservice()启动进程，一般会在skynet启动的时候提前启动actor，因为这两个方法都是阻塞的操作   <br>


<br/>


### skynet底层运行原理

1 fd怎么绑定actor    <br>
通过epoll_ctl把fd、actor地址包装成epoll_event，放到内核的协议栈红黑树中(epoll)；当这条连接网络事件触发，从红黑树拷贝到就绪队列，然后通过epoll_wait把就绪事件拷贝到用户态。     <br>

socket.start(fd，func) --【socket.lua文件】   <br>
监听端口时，传 listenfd、回调函数 两个参数；用户连接时，只需要绑定，只传clientfd(绑定 clientfd agent 网络消息)   <br>

```
function socket.start(id,func)
    driver.start(id)    --lua-socket.c文件
    return connect(id, func)
end

通过 driver 看到 socket_server_start()中的 send_request()【工作线程】 --- 直接搜'R'，到socket_server.c文件，看ctrl_cmd()中的R【网络线程】 --- resume_socket()
【一般agent是工作线程，一般有一个单独的网络线程，许多工作线程是通过 管道(pipeline) 把数据发送到网络线程来处理的】

static int resume_socket()
{
    int id = request->id;  //id就是clientfd
    result->id = id;
    result->opaque = request->opaque;    //opaque就是具体指向的actor
    ...
    struct socket *s = &ss->slot[HASH_ID(id)];   //opaque会传到s中
    ...
    if(enable_read(ss, s, true)) {...}   //关键
    ...
}

enable_read() --- sp_enable()  //监听读事件

static int sp_enable()
{
    struct epoll_event ev;
    ...
    ev.data.ptr = ud;  //opaque，resume_socket()中创建的socket s   里面有clientfd、actor的地址
    if(epoll_ctl(efd, EPOLL_CTL_MOD, sock, &ev) == -1) {...}   //重点是&ev，通过epoll_ctl把&ev传到网络当中

    //epoll_ctl是系统调用，操作的是linux内核中epoll的红黑树， 把actor的地址传到红黑树的节点当中
    //未来事件&ev被触发时，会把数据拷贝到就绪队列(内核)，然后通过epoll_wait把事件取出来，从内核拷贝到用户态，从中找到actor的地址，自然就可以知道消息要发送给哪一个actor
    ...
}

```


2 actor是如何调度的    <br>
线程池的调度，线程池用来调度actor  <br>

例：一个网络线程、一个定时线程、线程池、全局队列、多个actor(actor里有消息队列)   <br>
线程池只作用在全局消息队列   <br>
全局消息队列组织的数据是 活跃的(有消息的)actor的消息队列，也就是说，全局消息队列中的元素实际上指向actor消息队列的指针  <br>
线程池的工作原理：不断的从全局消息队列中取出消息(actor的消息队列)。线程池中的一个线程会取出消息队列，然后从中取出一个消息，然后把消息作为参数传入回调函数中（以此驱动actor运行）。    <br>

分析线程池：  <br>
1、线程池中，生产者和消费者全部列举出来    <br>
网络线程、定时线程产生的数据就是生产者    <br>
线程池即使消费者，也是生产者(actor之间发送消息)  <br>

```
源码：skynet_start.c
thread_worker() --- skynet_context_message_dispatch()【skynet_server.c】
struct message_queue * skynet_context_message_dispatch()
{
    ...
    q = skynet_globalmq_pop();  //从全局消息队列中取出actor的消息队列
    ...
    uint32_t handle = skynet_mq_handle(q);  //取出actor的句柄
    struct skynet_context * ctx = skynet_handle_grab(handle);  //通过句柄找到运行环境(上下文)
    ...
    if(skynet_mq_pop(q, &msg)){}  //通过上下文，从消息队列中去取消息&msg
    ...
    dispatch_message(ctx, &msg);  //分发消息，把msg分发到actor的上下文中
    ...
}

static void dispatch_message()  //skynet_server.c
{
    ...
    reserve_msg = ctx->cb(ctx, ...);  //callback回调函数，运行环境是ctx(actor)，msg作为参数，驱动actor运行
    ...
}
```

2、线程池是怎么唤醒的？怎么休眠的？    <br>
```
//【skynet_start.c】

休眠(挂起)：
thread_worker()   
{
    ...
    pthread_cond_wait(&m->cond, &m->mutex);  //参数：条件变量，互斥锁  两个互相搭配使用
    ...
}

唤醒：
定时线程：  thread_timer() --- skynet_updatetime()【skynet_timer.c】 --- timer_update()  //执行定时任务，产生消息
                        --- wakeup()
static void wakeup()
{
    pthread_cond_signal();  //发送一个signal，从pthread_cond_wait处唤醒
}

网络线程：  thread_socket() --- wakeup()

线程池中的线程：不需要 唤醒线程池线程，原因是 线程池既是生产者又是消费者， 当一个线程使actor发送消息给另一个actor时，没必要自己休眠去唤醒另一个线程(多了两次系统调用)，这个线程等执行完毕后，直接自己去取另一个actor消息队列中的消息执行就可以了。
```


