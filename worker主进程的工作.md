# 主进程工作

总体流程中提及到主进程中monitorWorkers是在监听信号和子进程退出时间,现在来仔细分析一下,主进程是怎么完成这个工作的

## 信号监听

在workerman启动的流程中说到了installSignal以及parseCommand，parseCommand中提及了几个命令，现在就来说明一下和信号相关的命令(reload,stop,connection,status)

```php
            pcntl_signal_dispatch();
            // Suspends execution of the current process until a child has exited, or until a signal is delivered
            $status = 0;
            $pid    = pcntl_wait($status, WUNTRACED);
            // Calls signal handlers for pending signals again.
            pcntl_signal_dispatch();

```

这段代码的作用就是监听信号、子进程退出,并且installSignal中也已经预先设置收到信号的默认处理方式

```php
    /**
     * Signal handler.
     *
     * @param int $signal
     */
    public static function signalHandler($signal)
    {
        switch ($signal) {
            // Stop.
            case SIGINT:
                self::stopAll();
                break;
            // Reload.
            case SIGUSR1:
                self::$_pidsToRestart = self::getAllWorkerPids();
                self::reload();
                break;
            // Show status.
            case SIGUSR2:
                self::writeStatisticsToStatusFile();
                break;
            // Show connection status.
            case SIGIO:
                self::writeConnectionsStatisticsToStatusFile();
                break;
        }
    }
```

我们来结合monitorWorkers的代码来逐一分析

### reload信号(SIGUSR1)

workerman通过reload方法来处理这个信号事件，代码分为两个部分

1. 父进程响应
2. 子进程响应

通过masterPid来判断当前接受到信号的是父进程还是子进程，然后在做不同的处理

1. 父进程

   1. 判断当前的进程状态允许的状态有STARTING、RUNNING，假设主进程状态为SHUTDOWN那么必然不能切回到RELOADING状态
   2. 执行onMasterReload用户回调事件
   3. 重置idMap(worker和进程ID的隐射关系)，事实上在monitor的过程中程序有重新设置idMap那么为什么还需要initId呢?因为reload过程中可以修改worker的count从而改变了worker的子进程个数，monitor中只是简单的将关联关系修改为0，此时就会出现idMap中个数与worker的count不一致的现象，所以需要重新设置。
   4. 根据worker的reloadable属性区分对待子进程
      1. reloadable为true，证明需要重启进程，将进程id保存到reloadable_pidarray中
      2. 直接发送SIGUSR1信号
   5. 发送一个信号给子进程并设置超时强制杀死信号(SIGKILL是无法修改信号处理方式)

2. 子进程

   子进程中的每一个workers都有可能有多个worker类,父进程只是根据创建是count数量fork子进程，但是如果在子进程中new Worker那么不会增加进程的数量，而只是新建了一个worker类的实例

   1. 任意取一个worker的实例
   2. 执行onWorkerReload
   3. 执行重载操作(调用StopAll)

父进程每次只发送一个信号的原因是为了平滑重启,一个子进程一个子进程重启，防止一个把线上所有的工作进程都杀死，对线上造成影响,当主进程执行完reload方法后程序回到了monitor的循环中监听信号或者子进程退出，此时SIGUSR1信号被子进程处理一个子进程退出信号被monitor捕获

```php
	// If a child has already exited.
	if ($pid > 0) {
      // Find out witch worker process exited.
      foreach(self::$_pidMap as $worker_id = > $worker_pid_array) {
          if (isset($worker_pid_array[$pid])) {
              $worker = self::$_workers[$worker_id];
              // Exit status.
              if ($status !== 0) {
                  self::log("worker[".$worker - > name.
                  ":$pid] exit with status $status");
              }
 
            	// For Statistics.
            	if (!isset(self::$_globalStatistics['worker_exit_info'][$worker_id][$status])) {
                	self::$_globalStatistics['worker_exit_info'][$worker_id][$status] = 0;
            	}
            	self::$_globalStatistics['worker_exit_info'][$worker_id][$status]++;
 
            	// Clear process data.
            	unset(self::$_pidMap[$worker_id][$pid]);
 
            	// Mark id is available.
            	$id = self::getId($worker_id, $pid);
            	self::$_idMap[$worker_id][$id] = 0;
 
            	break;
        	}
    	}
    	// Is still running state then fork a new worker process.
    	if (self::$_status !== self::STATUS_SHUTDOWN) {
        	self::forkWorkers();
        	// If reloading continue.
        	if (isset(self::$_pidsToRestart[$pid])) {
            	unset(self::$_pidsToRestart[$pid]);
            	self::reload();
        	}
    	} else {
        	// If shutdown state and all child processes exited then master process exit.
        	if (!self::getAllWorkerPids()) {
            	self::exitAndClearAll();
        	}
    	}
    }
```

可以看到主进程处理了一个主进程的退出事件后就会判断当前退出进程是否在退出进程组中，在的话就会再次触发主进程的reload事件，达成平滑重启的功能

### 退出信号(SIGINT)

workerman通过stopAll方法来处理这个信号事件，代码分为两个部分

1. 父进程响应
2. 子进程响应

方法详解

1. 父进程行为
   1. ​



## 子进程退出

