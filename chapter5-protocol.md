# Protocol

Protocol interface中定义了三个方法

1. input 检测数据包完整性并返回包长
2. encode数据包封包
3. decode数据包拆包

任何一个用户想扩展协议必须实现这三个方法,接下来通过一个workerman中的Text协议来进行一次分析

Text协议以\n作为一个数据包的分割符

如aaaa\nbbbbb\n则此处有两个数据包

## input

此处的input函数通过strpos寻找\n的位置如果找到了，则说明至少获取到了一个数据包,若找不到则说明不足一个数据包返回0

```php
$pos = strpos($buffer, "\n");
if ($pos === false) {
    return 0;
}
return $pos + 1;
```

## encode

封包协议也非常简单就是在数据后添加一个"\n"符号作为一个包的结束

## decode

拆包也就是包"\n"结束符过滤掉

可以参考官方文档来扩展协议
[参考资料]: http://doc.workerman.net/protocols/how-protocols.html	"官方协议添加实例"