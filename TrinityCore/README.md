
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



