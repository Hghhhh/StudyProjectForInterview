#三次握手
我们先明确两个定义： 
1，client为数据发送方 
2，server为数据接收方

好，下面进行三次握手的总结： 
1，client想要向server发送数据，请求连接。这时client想服务器发送一个数据包，其中同步位（SYN）被置为1，表明client申请TCP连接，序号为j。 
2，当server接收到了来自client的数据包时，解析发现同步位为1，便知道client是想要简历TCP连接，于是将当前client的IP、端口之类的加入未连接队列中，并向client回复接受连接请求，想client发送数据包，其中同步位为1，并附带确认位ACK=j+1，表明server已经准备好分配资源了，并向client发起连接请求，请求client为建立TCP连接而分配资源。 
3，client向server回复一个ACK，并分配资源建立连接。server收到这个确认时也分配资源进行连接的建立

那么问题来了
为什么需要第三次握手？
第三次握手失败了怎么办？
三次握手有什么缺陷可以被黑客利用，用来对服务器进行攻击？
怎么防范这种攻击？
接下来进行一一解答。 
##1，为什么需要第三次握手？ 
答：如果没有第三次握手，可能会出现如下情况：

如果只有两次握手，那么server收到了client的SYN=1的请求连接数据包之后，便会分配资源并且向client发送一个确认位ACK回复数据包。
那么，如果在client与server建立连接的过程中，由于网络不顺畅等原因造成的通信链路中存在着残留数据包，即client向server发送的请求建立连接的数据包由于数据链路的拥塞或者质量不佳导致该连接请求数据包仍然在网络的链路中，这些残留数据包会造成如下危害
危害：当client与server建立连接，数据发送完毕并且关闭TCP连接之后，如果链路中的残留数据包才到达server，那么server就会认为client重新发送了一次连接申请，便会回复ACK包并且分配资源。并且一直等待client发送数据，这就会造成server的资源浪费。

##2，第三次握手失败了怎么办？ 
答：当client与server的第三次握手失败了之后，即client发送至server的确认建立连接报文段未能到达server，server在等待client回复ACK的过程中超时了，那么server会向client发送一个RTS报文段并进入关闭状态，即：并不等待client第三次握手的ACK包重传，直接关闭连接请求，这主要是为了防止泛洪攻击，即坏人伪造许多IP向server发送连接请求，从而将server的未连接队列塞满，浪费server的资源。

##3，三次握手有什么缺陷可以被黑客利用，用来对服务器进行攻击？ 
答：黑客仿造IP大量的向server发送TCP连接请求报文包，从而将server的半连接队列（上文所说的未连接队列，即server收到连接请求SYN之后将client加入半连接队列中）占满，从而使得server拒绝其他正常的连接请求。即拒绝服务攻击

##4 , 怎么防范这种攻击？ 
1，缩短服务器接收客户端SYN报文之后的等待连接时间，即SYN timeout时间，也就是server接收到SYN报文段，到最后放弃此连接请求的超时时间，将SYN timeout设置的更低，便可以成倍的减少server的负荷，但是过低的SYN timeout可能会影响正常的TCP连接的建立，一旦网络不通畅便可能导致client连接请求失败

2，SYN cookie + SYN proxy 无缝集成（较好的解决方案）

SYN cookie：当server接收到client的SYN之后，不立即分配资源，而是根据client发送过来的SYN包计算出一个cookie值，这个cookie值用来存储server返回给client的SYN+ACK数据包中的初始序列号，当client返回第三次握手的ACK包之后进行校验，如果校验成功则server分配资源，建立连接。
SYN proxy代理，作为server与client连接的代理，代替server与client建立三次握手的连接，同时SYN proxy与client建立好了三次握手连接之后，确保是正常的TCP连接，而不是TCP泛洪攻击，那么SYN proxy就与server建立三次握手连接，作为代理（网关？）来连通client与server。（类似VPN了解一下。）

#四次挥手
下面对四次挥手的过程进行总结： 
1，（我客户端已经没有话跟你说了） 当client所要发送的数据发送完毕之后（这里是接收到了来自于server方的，对于clien发送的最后一个数据包的收到确认ACK包之后），向server请求关闭连接，因此client向server发送一个FIN数据包，表明client数据发送完了，申请断开。并且自身进入等待结束连接状态FIN_WAIT-1 
2，（我服务器知道你没话说了，但是你等等我，我不确定是否没话跟你说了，等我的回复）当server收到了来自client的FIN请求包之后，向client回复一个确认报文ACK，同时进入关闭等待状态（CLOSE -WAIT），这时TCP连接的server便会向上层应用发送通知，表明client数据发送完毕，是否需要发送数据给client，这时TCP连接已经处于半关闭状态了，因为client已经没有数据要发送了。同时client收到了来自server的确认报文之后，便会进入FIN-WAIT-2状态。 
3，（我服务器的上层应用也没话说了，咱们可以断开连接了） TCP连接的server收到上层应用的指令表明没有数据要发送之后，会向client发送一个FIN请求数据包，其中确认位ACK=1，表明响应client的关闭连接请求，同时FIN=1表明server也准备好断开TCP连接了。 
4，*（好的，那么咱们断开连接吧）***client收到了来自server的FIN数据包之后，知道server已经准备好断开连接了，于是向server发送一个确认数据包ACK，告诉server，可以关闭资源断开连接了。同时自身进入TIME-WAIT阶段，这个阶段将持续2MSL（MSL：报文在网络链路中的最长生命时长），在等待2MSL之后，client将会关闭资源。 
5，serve日收到了来自于client的最后一个确认断开连接数据包之后便会直接进入TCP关闭状态，进行资源的回收。

那么问题来了： 
##1，为什么要四次挥手 
答：前两次挥手是为了断开client至server的连接，后两次挥手是为了断开server至client的连接，如果没有第四次挥手，会出现如下状况：

server发送FIN数据包并携带ACK至client之后直接断开连接，如果client没有收到这个FIN数据包，那么client会一直处于等待关闭状态，这是为了确保TCP协议是面向连接安全有保证锝。
上面解释了为什么不是三次挥手，同理，两次挥手也是不安全的。不能保证server与client都能正确关闭连接释放资源，而不会造成资源浪费。
##2，四次挥手之后client为什么还要等待2MSL的时间才释放资源关闭连接？ 
答：

如果client第四次挥手的确认报文段没有被server接收，那么server便会重发第三次挥手的FIN报文段，因此client要停留2MSL的时长来处理可能会重复收到的报文段。

让之前建立的client-server通信过程中或者是挥手过程中由于网络不通畅产生的滞留报文段失效。如果不等待2MSL，那么建立新连接之后，可能会收到上一次连接的旧报文段，可能会造成混乱。


原文：https://blog.csdn.net/scuzoutao/article/details/81774100 
