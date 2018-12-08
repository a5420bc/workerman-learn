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

对base_read在做一个简要的说明

若设置了ssl连接且ssl连接尚未完成，那么当前连接上的所有数据都是为了ssl握手，所以baseRead只需要考虑握手认证，不需要处理onMessage事件，同时若完成ssl连接后有sendBuffer则设置write定时event

当未设置应用层协议时，数据是不需要拆包

若需要应用协议进行拆包则需要先获取包长，若有数个包则直至处理至数据不足为止

若socket不可用则需要检查destroy连接防止维护早已断开的socket浪费服务器资源

关于pause，可以看到读取数据时是考虑了当前的是否设置了暂停读取的，在pauseRecv方法中直接的把base_read事件去除了，那么为什么还需要考虑呢？因为就算当前连接设置了暂停读取，在这个时间段中仍能可能受到客户端发过来的大量请求，所以需要为这些后续的请求进行判断，是否需要读取数据

### baseWrite

1. 当sendBuffer为空时会去除write event减轻eventloop的压力
2. 由于服务端主动关闭时可能还有内容需要发出会等到内容发出再销毁连接，但是不会理会服务端的接受数据

### pauseRecv  

删除read event事件并设置标识

### resumeRecv

重新添加read event 事件并设置标识

应用层的暂停接受是利用了tcp的滑动窗口，当服务端一直不接受数据那么socket的read buffer就会满，将会导致tcp的窗口大小为零，最终迫使客户端不发送数据，控制流量

## UdpConnection

由于udp协议不面向连接也就没有收到数据包不完整或者是粘包这个问题,直接发送，同时由于底层未能向tcp那样有流量控制协议，因此workerman也未提供流量控制

## AsyncTcpConnection

提供异步的客户端连接

### __construct

构造函数基本上就是把worker类中的listen方法的建立连接照搬了一遍

### connect

连接建立需要时间，主要的开销是在网络连接上，因此设置了一个连接确认的回调函数

## checkConnection

通过确认写就绪事件来确认连接是否建立成功，其余设置基本与worker一节当中建立连接相同，唯一需要注意的是sslHandShakeCompleted属性是被设置为true这是因为AsyncTcpConnection是基于TcpConnection,所以如果需要建立的是ssl连接那么baseRead方法中会对该属性进行确认，TcpConnection可读是需要ssl握手成功后才可以读取应用数据，而作为客户端的一方必须得先发送握手信息，所以此处需要设置该字段为true。