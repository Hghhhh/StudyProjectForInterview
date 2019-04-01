mount命名空间

UTS命名空间

[IPC namespaces](http://lwn.net/Articles/187274/) 

[PID namespaces](http://lwn.net/Articles/259217/) 

[Network namespaces](http://lwn.net/Articles/219794/)

[User namespaces](http://lwn.net/Articles/528078/)



LDAP是轻量[目录访问协议](https://baike.baidu.com/item/%E7%9B%AE%E5%BD%95%E8%AE%BF%E9%97%AE%E5%8D%8F%E8%AE%AE)，英文全称是Lightweight Directory Access Protocol，一般都简称为LDAP。它是基于X.500标准的，但是简单多了并且可以根据需要定制。与X.500不同，LDAP支持TCP/IP，这对访问Internet是必须的。LDAP的核心规范在RFC中都有定义，所有与LDAP相关的RFC都可以在LDAPman RFC网页中找到。







```
`class` `A``{``        ``int` `a;``        ``short` `b;``        ``int` `c;``        ``char` `d;``};``class` `B``{``        ``double` `a;``        ``short` `b;``        ``int` `c;``        ``char` `d;``};`
```

  在32位机器上用gcc编译以上代码，求sizeof(A),sizeof(B)分别是多少。

根据以下条件进行计算：
 1、  结构体的大小等于结构体内最大成员大小的整数倍
 2、    结构体内的成员的首地址相对于结构体首地址的偏移量是其类型大小的整数倍，比如说double型成员相对于结构体的首地址的地址偏移量应该是8的倍数。
   3、  为了满足规则1和2编译器会在结构体成员之后进行字节填充！

   A中，a占4个字节，b本应占2个字节，但由于c占4个字节，为了满足条件2，b多占用2个字节，为了满足条件1，d占用4个字节，一共16个字节。
   B中，a占8个字节，b占2个字节，但由于c占4个字节，为了满足条件2，b多占用2个字节，
   即abc共占用8+4+4=16个字节，
 为了满足条件1，d将占用8个字节，一共24个字节。  