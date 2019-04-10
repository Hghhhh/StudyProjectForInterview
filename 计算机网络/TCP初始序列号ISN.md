#TCP初始序列号

TCP初始化序列号不能设置为一个固定值，因为这样容易被攻击者猜出后续序列号，从而遭到攻击。

RFC1948中提出了一个较好的初始化序列号ISN随机生成算法。

ISN = M + F(localhost, localport, remotehost, remoteport).

M是一个计时器，这个计时器每隔4毫秒加1。

F是一个Hash算法，根据源IP、目的IP、源端口、目的端口生成一个随机数值。要保证hash算法不能被外部轻易推算得出，用MD5算法是一个比较好的选择。



转自：<https://blog.csdn.net/zhangqi_gsts/article/details/50617291>

