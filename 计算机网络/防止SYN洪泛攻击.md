#SYN洪泛攻击

TCP SYN[泛洪](https://baike.baidu.com/item/%E6%B3%9B%E6%B4%AA)发生在OSI第四层，这种方式利用TCP协议的特性，就是[三次握手](https://baike.baidu.com/item/%E4%B8%89%E6%AC%A1%E6%8F%A1%E6%89%8B)。

**攻击者发送TCP SYN，[SYN](https://baike.baidu.com/item/SYN/8880122)是[TCP](https://baike.baidu.com/item/TCP/33012)三次握手中的第一个数据包，而当服务器返回ACK后，该攻击者就不对其进行再确认，那这个TCP连接就处于[挂起状态](https://baike.baidu.com/item/%E6%8C%82%E8%B5%B7%E7%8A%B6%E6%80%81)，也就是所谓的半连接状态，服务器收不到再确认的话，还会重复发送ACK给攻击者。**

这样更加会浪费服务器的资源。攻击者就对服务器发送非常大量的这种TCP连接，由于每一个都没法完成三次握手，所以在服务器上，这些TCP连接会因为挂起状态而消耗CPU和内存，最后服务器可能[死机](https://baike.baidu.com/item/%E6%AD%BB%E6%9C%BA)，就无法为正常用户提供服务了。

##防止方法：

 一、可缩短 SYN Timeout时间，可以通过缩短从接收到SYN报文到确定这个报文无效并丢弃该连接的时间，可以降低服务器负荷。

二、设置SYN Cookie，给每个请求连接的IP地址分配一个Cookie，如果短时间内收到同一个IP的重复SYN报文，则以后从这个IP地址来的包会被丢弃。  

修改linux这个`/etc/sysctl.conf`文件中的一些tcp配置可以有效防止syn洪泛攻击：

```java
net.ipv4.tcp_synack_retries = 0 #默认值是5
对于远端的连接请求SYN，内核会发送SYN ＋ ACK数据报，以确认收到上一个 SYN连接请求包。这是所谓的三次握手( threeway handshake)机制的第二个步骤。这里决定内核在放弃连接之前所送出的 SYN+ACK 数目。不应该大于255，默认值是5，对应于180秒左右时间。(可以根据上面的tcp_syn_retries来决定这个值)
```

更多配置信息可以看下面的文章：

[linux中的一些TCP配置来防止syn泛洪攻击](https://www.jb51.net/article/114073.htm)

[linux中一些TCP配置](https://blog.csdn.net/iteye_11158/article/details/81925517)