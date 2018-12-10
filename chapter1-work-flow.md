# 总体流程

## Workerman版本为3.5.2

## 主方法

workerman的入口在Worker.php中的runAll方法中
该方法中包含以下事件

1. 检查当前运行环境(checkSapiEnv)
2. 初始化worker类成员变量(init)
3. 解析命令输入(parseCommand)
4. 守护进程化(daemonize)
5. 初始化子进程实体(initWorkers)
6. 信号安装(installSignal)
7. 保存进程ID
8. UI展示(displayUI)
9. fork子进程(forkWorkers)
10. 重定向输入输出(resetStd)
11. 监控进程状态(monitorWorkers)

## checkSapiEnv

此方法中判断当前环境是否为cli(command-line interface),如果不是在cli环境中需要终止程序的运行

## init

1. 设置pidFile(主进程ID文件)
2. 设置logFile(日志文件),不存在就创建
3. 初始化全局统计时间,$_globalStatistics
4. 设置全部统计文件statisticsFile
5. 设置进程标题
6. 初始化所有worker的id信息,此处主要是设置id对pid的关联关系
7. 初始化定时器

## 命令行解析

解析启动命令确定响应事件

1. 错误显示提示,如果用户输入命令错误提示正确使用方法

2. 确认当前的服务是否已经开启,start和restart不依赖服务是否启动,但是其他的命令都是需要服务启动才生效

   ```php
   $master_pid      = is_file(self::$pidFile) ? file_get_contents(self::$pidFile) : 0;

   $master_is_alive = $master_pid && @posix_kill($master_pid, 0) && posix_getpid() != $master_pid;
   ```

   对主进程是否存活进行判断

   1. 如果主进程启动完成过,那么pidFile会存入主进程pid但不能证明主进程目前还存活
   2. posix_kill确认主进程存活但是posix_kill并不算靠谱,因为linux系统会重复使用进程id
   3. posix_getpid() != $master_pid因以上的原因如果当前的进程id为文件中记录id那么主进程必然不存活

##守护进程化
当前就是在处理守护进程,此处说明一下守护进程的实现

```php
protected static function daemonize()
{
    if (!self::$daemonize) {
        return;
    }
    //将默认权限掩码修改为0,意味着即将要创建的文件的权限都是777
    umask(0);
    // 子进程
    $pid = pcntl_fork();
    // 子进程创建失败
    if (-1 === $pid) {
        throw new Exception('fork fail');
    } elseif ($pid > 0) {  //说明当前进程是父进程,只有在父进程中fork才会返回pid
        // 关闭父进程，让子进程成为孤儿进程被init进程收养
        exit(0);
    }
    // 将子进程作为进程组的leader,开启一个新的会话,脱离之前的会话和进程组。即使用户logout也不会终止
    if (-1 === posix_setsid()) {
        throw new Exception("setsid fail");
    }
    // Fork again avoid SVR4 system regain the control of terminal.
    // 避免svr4系统重新获取控制终端
    $pid = pcntl_fork();
    if (-1 === $pid) {
        throw new Exception("fork fail");
    } elseif (0 !== $pid) {
        // 如果不是孙子进程，直接干掉。让孙子进程成为孤儿进程被init进程（1号进程收养）
        exit(0);
    }
}
```
> 注释来源https://segmentfault.com/a/1190000007420468

补充说明:

1. umasklinux默认情况会设置文件掩码也就是说会控制文件的创建权限(-rwxrwxrwx),如果不将umask设置为0,用户创建文件后，可能获得的也不是想要的权限，因为掩码会自动的剔除对应的权限

   假设当前掩码是0022,那么默认文件就只有755权限

2. 当退出了进程终端后，会话首进程会发送SIGHUP信号给所有的前台和后台进程,而进程默认接受到该信号的处理方案是终止进程,所以通过posix_setsid()新建一个会话并脱离当前的控制终端的会话,那么就不会接受到SIGHUP信号

参考资料:

如果通过以上的注释仍不能理解推荐看,UNIX环境高级编程     第8章　进程控制  第9章  进程关系 第10章 信号 第13章　守护进程(本章有使用线程实现对本例理解用处不太)

##初始化子进程实体

1. 计算最长进程名
2. 计算最长worker用户名
3. 计算最长套接字名称
4. 创建listen套接字(不开启reusePort)

> 依据官方文档reusePort功能需要php7以上才能支持但实际上还需要linux版本的支持(内核版本3.9)
> 关于启用reusePort和不启用的区别可以参考http://www.ywnds.com/?p=4543

> 计算以上的最长的长度是为了在cli界面上展示的美观点，也就是为了对齐

## 信号安装

1. SIGINT信号停止服务
2. SIGUSR1作为从起服务信号
3. SIGUSR2作为状态查看信号
4. SIGIO作为连接信息查看信号
5. 无视SIGPIPE信号解决因套接字(linux下写**socket**的程序的时候，如果尝试send到一个disconnected **socket**上，就会让底层抛出一个SIGPIPE信号)

## 保存进程ID

将进程ID保存到对应的pidFile中

## 展示UI

```php
        self::safeEcho("\033[1A\n\033[K-----------------------\033[47;30m WORKERMAN \033[0m-----------------------------\n\033[0m");
```

以本段文本做一下简要的解释说明:

除了为大家所周知的"\n"的ANSI的转义字符以外，ANSI还包括其他的转义字符,被称为转义序列

转义序列:ESC[

`**ESC**`（27 / [十六进制](https://zh.wikipedia.org/wiki/%E5%8D%81%E5%85%AD%E8%BF%9B%E5%88%B6) 0x1B）开头或者可以用\033的8进制表示

\033[1A		将当前光标移到上一行

\033[k  		清除整行

\033[47;30m 设置背景色，设置前景色

\033[0m		重置设置

[参考资料1]: https://segmentfault.com/a/1190000012666612
[参考资料2]: https://blog.csdn.net/wentyoon/article/details/64920457
[参考资料3]: https://zh.wikipedia.org/wiki/ANSI%E8%BD%AC%E4%B9%89%E5%BA%8F%E5%88%97

## fork子进程

1. 运行过程中wokerName有可能修改，所以需要重新计算
2. fork一个子进程

## 重定向输入输出

解释:

unix每个进程开启时必然会拥有标准输入、标准输出、错误输出三个文件描述符,当关闭了一个文件描述符再重新打开时，会获取当前打开的文件描述符的下一个作为fd(file desc),基于上诉原因fclose后fopen可以获取fclose的文件描述符，其次0/1/2三个文件描述符与标准输入、标准输出、错误输出是绑定的,也就是fd为0必然是标准输入。

## 监控进程状态

主进程通过无限循环响应信号事件以及处理子进程退出状态,从而维护子进程数量，以及控制服务的行为(退出、重启、显示全局信息)

下面以代码示例说明一下一些要点部分

```php
    /**
     * Monitor all child processes.
     *
     * @return void
     */
    protected static function monitorWorkers()
    {
        self::$_status = self::STATUS_RUNNING;
        while (1) {
            // Calls signal handlers for pending signals.
            /**
             * pcntl_wait接受了信号并进行信号处理之后，会开始响应主进程事件，这个过程中同样可能发生信号
             * 如果下一次循环不处理信号，那么信号处理会被延迟到再下一次pcntl_wait的返回
             */
            pcntl_signal_dispatch();
            // Suspends execution of the current process until a child has exited, or until a signal is delivered
            $status = 0;
            $pid    = pcntl_wait($status, WUNTRACED);
            // Calls signal handlers for pending signals again.
            pcntl_signal_dispatch();
            // If a child has already exited.
			if ($pid > 0) {
                //do the thing 
            } else {
                //do the thing 
            }
        }
    }
```

首先说明一下为什么需要pcntl_signal_dispatch函数调用,因为php不会主动处理信号事件，除非设置declare(tick=1),但是实际上declare也只是在每一个php代码执行过程中去调用一次pcntl_signal_dispatch，php对信号的实现并不能做到c语言一样可以直接响应而是必须要手动触发，这就是该函数存在的原因

说明一下为什么需要两次pcntl_signal_dispatch

* pcntl_wait($status, WUNTRACED)退出条件为
  * 发现子进程退出
  * 接受到了信号

既然pcntl_wait会因为接受信号而退出所以其后需要处理信号事件,其次pcntl_wait接受了信号并进行信号处理之后，会开始响应主进程事件，这个过程中同样可能发生信号如果下一次循环不处理信号，那么信号处理会被延迟到再下一次pcntl_wait的返回,如果是这样对信号处理延迟就非常大了，达不到用户对信号处理的预期

## 官方流程图

![workerman master woker模型](http://wenda.workerman.net/uploads/answer/20140815/5670ea17653a1a6e6811ed5148f77c96.png)