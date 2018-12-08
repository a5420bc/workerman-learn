# connection

## TcpConnection

####构造

tcpconnection连接被设置为非阻塞，也就是说无论是read还是write时间都会直接返回不会等待写入到内核socket缓冲区，同时会添加read事件等待对端发送连接请求。

#### send

1. 协议封包
2. 将buffer写入socket缓冲区
3. 写入未完成则注册write event等待内核write事件
4. 检查buffer full

bufferIsFull和checkBufferWillFull

bufferIsFull是如果当前缓存已满则会拒绝本次写入，并触发onError

checkBufferWillFull只是检查在本次缓存写入后，是否已满，并触发onBufferFull

正如手册上提及的如果缓存未满不管本次要写入多大的数据都可以写入成功，这里的逻辑是这样，如果考虑缓存未满到底需要写多少数据才会满的话，逻辑过于复杂，还要考虑将一个应用协议包截成数段发送等问题。

另一点，此时典型的是一个事件驱动异步编程，当一次fwrite调用无法写入所有的数据时，直接注册了一个write event并通过该事件来继续写入数据。

### baseRead

![base_read](C:\Users\viruser.v-desktop\Desktop\workerman\base_read.png)

## UdpConnection

## AsyncTcpConnection

