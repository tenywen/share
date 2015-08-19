简单比较thritf,grpc,nsq得出的几点

* 1.高并发的游戏服务器,使用短链接的rpc,简直蠢爆了！！

* 2.thrift 有三种服务端模式TSimpleServer,TThreadPoolServer,TNonblockingServer。TSimpleServer就是阻塞的单线程io模式。蠢！后两种都是基于线程池的实现。特别是TNonblockingServer是基于Java NIO模式(单独使用一个线程来处理事件的阻塞,linux上NIO 使用epoll来实现)。所以高并发，长连接还是不适合用thrift。

* 3.thrift的开发文档太少了。我是没找到有什么好的开发社区.

* 4.nsq这类消息队列。首先需要额外的协议开发，其次还需要考虑服务器有持久化的需求，因为nsq需要订阅方回覆ok.

所以俺还是选择grpc吧。再说一次短链接真的很蠢！
