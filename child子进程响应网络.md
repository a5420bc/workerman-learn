# 子进程工作

## 子进程初始化

1. 自动加载路径设置
2. 如果有监听的端口，则为监听设置事件处理器(event-loop)
3. 将父进程的信号全部重置,添加到事件处理器上
4. 处理定时器
5. 执行onWorkerStart回调函数
6. 执行事件处理器的循环

可以看到子进程的最主要的工作是执行事件处理器,但是子进程不是**一个**worker实例,子进程是event-loop实例，其中最主要运行的正是全局静态的event-loop，而worker实例只是作为一个具体的消息接收、发送的处理过程。

## event-loop

以select系统调用为例，说明一下IO多路复用的实现

### 初始化

stream_socket_pair创建一对unix套接字,并将其读端加入到事件队列中(事件循环如果不存在任何一个句柄则会空转，主要针对于只在worker进程中使用定时器，而不加入tcp/udp/http等网络事件)。

### 添加和删除事件

1. READ(为当前句柄可读，一般是网络可读事件发生)

2. WRITE

3. EXPECT(带外数据)

4. SIGNAL(信号)

5. TIMER(分为一次性定时器和永久定时器)


###事件轮询

以loop函数源码进一步解释一下这些事件是如何被触发的

```php
    public function loop()
    {
        $e = null;
        while (1) {
            if(DIRECTORY_SEPARATOR === '/') {
                // Calls signal handlers for pending signals
                pcntl_signal_dispatch();
            }

            $read  = $this->_readFds;
            $write = $this->_writeFds;
            $except = $this->_writeFds;

            // Waiting read/write/signal/timeout events.
            $ret = @stream_select($read, $write, $except, 0, $this->_selectTimeout);

            if (!$this->_scheduler->isEmpty()) {
                $this->tick();
            }

            if (!$ret) {
                continue;
            }

            if ($read) {
                foreach ($read as $fd) {
                    $fd_key = (int)$fd;
                    if (isset($this->_allEvents[$fd_key][self::EV_READ])) {
                        call_user_func_array($this->_allEvents[$fd_key][self::EV_READ][0],
                            array($this->_allEvents[$fd_key][self::EV_READ][1]));
                    }
                }
            }

            if ($write) {
                foreach ($write as $fd) {
                    $fd_key = (int)$fd;
                    if (isset($this->_allEvents[$fd_key][self::EV_WRITE])) {
                        call_user_func_array($this->_allEvents[$fd_key][self::EV_WRITE][0],
                            array($this->_allEvents[$fd_key][self::EV_WRITE][1]));
                    }
                }
            }

            if($except) {
                foreach($except as $fd) {
                    $fd_key = (int) $fd;
                    if(isset($this->_allEvents[$fd_key][self::EV_EXCEPT])) {
                        call_user_func_array($this->_allEvents[$fd_key][self::EV_EXCEPT][0],
                            array($this->_allEvents[$fd_key][self::EV_EXCEPT][1]));
                    }
                }
            }
        }
    }
```

stream_select只能响应读写异常事件，对于定时器是不能生效的，因此在workerman中每加入一个定时器，select都会与当前的所有超时时间进行比较找出最小的，并将其设置为下一次的timeout，stream_select返回时，将会触发一次tick函数调用设置的回调函数,由此可以得知，如果在回调函数中设置了需要大量时间处理的任务，必然会影响select响应，阻塞整个进程处理，因为php是单进程单线程的。(官方的建议是对例如群里邮件这样的任务新建一个工作worker),同时stream_select也会被信号中断所以在loop中可以触发信号事件。

这里需要对比一下loop的pcntl_signal_dispatch和worker中的区别，在worker类中dispatch在pcntl_waitpid的前后都出现了，在worker一节中也说明过是为了防止信号处理不及时，那么在此处一个dispatch就够了？stream_select具有超时时间这是和pcntl_waitpid所不同的地方，也就是说它除了信号事件发生以外会返回而不是像waitpid长时间阻塞导致信号处理不及时。

stream_select本身的处理方式这里就不展开说明了可以参考(stream_select与socket_select本质上是一样的，区别在于操作的对象一个是socket的fd，一个是stream对象)

>https://juejin.im/entry/5b457e356fb9a04f8856be01
>
>http://php.net/manual/zh/function.stream-select.php
>
>UNIX高级环境编程第十四章 高级I/O中对select函数的描述

#### tick

tick方法用于执行定时器，从优先队列中查看定时最小的定时事件，比较是否到执行时间，如果可以执行则执行，并判断是否需要执行多次，单个定时器如果执行时间过长，会影响其他定时事件。一直取出定时事件，直至不满足执行时间的要求。