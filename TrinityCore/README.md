
mmo开源模块框架

### 地图模块管理

地图模块  1.AOI(地图数据管理)    2.AABB算法(碰撞检测)   3.A* 算法(寻路)

整体架构：   <br>
一个auth服务器，主要用来登录注册   <br>
多个world server     <br>
【客户端先登录auth，再登录world server】


1、框架中的线程模型

线程模型主要就是看world sever如何处理整个游戏的业务逻辑

首先最外层有多个线程(acceptor)来接收连接，之后会有多个网络线程(network)来读写数据。

acceptor每一个线程都会有_newsockets数组，采用负载均衡算法，选择线程中管理连接数最少的，将连接放入_newsockets中。

接下来network线程会不断地把_newsockets放到_sockets里面，然后在每一个线程中开始读写_sockets数据，然后就会产生很多数据包，再将数据包放入队列(msg queue)当中。

之后会有一个主线程main，不断从msg queue中拉取数据包来处理。主线程发送数据时，会把数据放入一个队列当中(msg queue per conn)，每一个连接会对应一个队列。队列会去找当前的socket是哪一个network线程去管理了，然后经由这一个network线程将数据发送出去。

【出现这些复杂流程的原因是 网络中使用了boost.asio，它是一种异步IO的处理，为了避免多线程的处理，所以统一让network线程发送消息】

此时msg queue还有部分的数据没有在main线程中处理，因为一些网络包是在地图当中触发的，比如攻击怪物，所以还会有地图map线程来管理地图。【通常在mmo中管理地图不会用线程，而是进程。因为使用多线程的话，当某一块地图的业务逻辑出错，可能会导致整个world server宕机】

msg queue中有部分数据（就是地图中实现的功能所发送的数据包）会路由到map线程，有一个map queue(里面存放服务器加载的多少块地图)，由map线程不断从map queue中取出map对象并执行对应的功能。map线程发送数据时，也会将数据放入msg queue per conn队列当中。【因为msg queue per conn队列是每一条连接对应一个队列，所以如果是某一个连接的消息，会路由到那条连接对应的队列当中。】

```
worldserver/Main.cpp
extern int main()
{
    //注：在boost.asio中核心的对象是IoContext（相当于reactor模型中的reactor对象）
    //看acceptor线程
    std::shared_ptr<Trinity::ThreadPool> threadPool = std::make_shared<Trinity::ThreadPool>(numThreads);//创建一个线程池
    for (int i = 0; i < numThreads; ++i)
        threadPool->PostWork([ioContext]() {ioContext->run();});//线程池对应的都是一个ioContext，就是在这里处理acceptor，用来接收连接
}

Networking/AsyncAcceptor.h  异步接收连接（就是在上述线程池中设置的）


Networking/NetworkThread.h  读写数据
void run()
{
    _updateTimer.async_wait([this](boost::system::error_code const&) {Update();});//每隔一毫秒执行一次Update()

    _newSockets.clear();//把newSockets放到sockets里面
    _sockets.clear();
}

void Update()
{
    if (!sock->Update()) {} //不断地从具体连接中读取数据，在这里接收完数据之后，会把数据放入msg queue当中
}


主线程怎么从消息队列当中取出消息去处理：


```

弊端：   <br>
1 main线程没做任何事，在阻塞等待，其实main线程可以作为map线程组的一部分（具体参考redis）    <br>
2 map线程不安全，当其中一个map线程业务逻辑出错时，可能会导致整个进程宕机（map多进程处理更优，进程的健壮性比线程高）  <br>



### 如何驱动地图模块中的数据
