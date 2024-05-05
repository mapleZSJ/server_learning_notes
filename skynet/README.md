### 三个核心要素
actor、消息、携程    <br>

抽象actor并发模型 --- 抽象进程（用户层的进程） --- 独立的运行环境     <br>

actor:      <br>
lua虚拟机：通过 lua虚拟机 在用户层构建【也就是说每个抽象进程都是lua虚拟机、都是独立运行的】     <br>
消息队列：通过 消息 驱动运行，消息队列按顺序存放消息              <br>
回调函数：为每个类型的消息设置 回调函数，从消息队列中取出 消息 作为参数去 调用回调函数，运行环境是在lua虚拟机中  <br>


消息:    <br>


协程:       <br>
实现同步非阻塞的逻辑【使用消息为驱动方式会涉及到 异步调用】   <br>
满足条件 唤醒、不满足条件 挂起  ---> 比如有完整的数据包，才会开展业务逻辑  <br>
【例：actor向redis发送消息请求数据后挂起，当redis返回数据后唤醒actor（是异步的流程）。通过协程可以同步处理数据（如：local name = redis:hget(...)）;而不用注册回调函数，使得获取数据后要在另一个业务逻辑中去处理】   <br>
            


通信会涉及到actor与actor之间的通信、以及网络模块与actor之间的通信：   <br>
网络模块一般是reactor网络模型（非阻塞IO），也就是异步事件的模型。网络会发送事件，将事件包装成消息后，插入到具体的actor的消息队列中;   <br>
抽象进程本质是在一个进程中，actor给另一个actor发送消息，本质上是指针的移动 ---> 把actor产生的消息的指针插入到另一个actor的消息队列中;    <br>
如果有定时任务，会有专门的定时线程。actor抛一个具体的定时任务，定时线程仅知道这个任务，过期时定时线程会抛消息给具体的actor;   <br>


<br/>


### actor编程思路

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


其他：   <br>
Actor - 本质上是做功能抽象  <br>
优化：当玩家多时，一个actor处理多个连接(玩家)，根据不同的客户端id，创建结构来存储玩家数据；一个actor处理多个房间    <br>
skynet通过newservice()、uniqueservice()启动进程，一般会在skynet启动的时候提前启动actor，因为这两个方法都是阻塞的操作   <br>


<br/>


### skynet底层运行原理

1 fd怎么绑定actor    <br>

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


