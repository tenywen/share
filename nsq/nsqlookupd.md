## nsqlookupd 

**nsqlookupd记录nsqd的网络拓扑结构。记录nsqd的网络地址，拥有topic，channel信息** 

**nsqd**
nsqd启动后，会有一个goroutine lookupLoop()函数，将nsqd中topic，channel信息向nsqdlookupd 注册。

**client**
client可以向nsqlookupd 请求获得topic，channel信息
