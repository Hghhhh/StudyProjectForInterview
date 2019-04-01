摘自https://snailclimb.top

[基于 Java NIO 实现简单的 HTTP 服务器](http://www.tianxiaobo.com/2018/04/04/%E5%9F%BA%E4%BA%8E-Java-NIO-%E5%AE%9E%E7%8E%B0%E7%AE%80%E5%8D%95%E7%9A%84-HTTP-%E6%9C%8D%E5%8A%A1%E5%99%A8/)

https://github.com/Snailclimb/JavaGuide/blob/master/Java%E7%9B%B8%E5%85%B3/Java%20IO%E4%B8%8ENIO.md#%E4%B8%80-java-io%EF%BC%8C%E7%A1%AC%E9%AA%A8%E5%A4%B4%E4%B9%9F%E8%83%BD%E5%8F%98%E8%BD%AF

### [一　Java IO，硬骨头也能变软](https://mp.weixin.qq.com/s?__biz=MzU4NDQ4MzU5OA==&mid=2247483981&idx=1&sn=6e5c682d76972c8d2cf271a85dcf09e2&chksm=fd98542ccaefdd3a70428e9549bc33e8165836855edaa748928d16c1ebde9648579d3acaac10#rd)

**（1） 按操作方式分类结构图：**

[![按操作方式分类结构图：](https://camo.githubusercontent.com/50f105c85f6b42d643d46e1ac7bb0f855b92cd9d/68747470733a2f2f757365722d676f6c642d63646e2e786974752e696f2f323031382f352f31362f313633363764346664316365316234363f773d37323026683d3130383026663d6a70656726733d3639353232)](https://camo.githubusercontent.com/50f105c85f6b42d643d46e1ac7bb0f855b92cd9d/68747470733a2f2f757365722d676f6c642d63646e2e786974752e696f2f323031382f352f31362f313633363764346664316365316234363f773d37323026683d3130383026663d6a70656726733d3639353232)

**（2）按操作对象分类结构图**

[![按操作对象分类结构图](https://camo.githubusercontent.com/8957efacdf1cd4eac15d844da8353a7f77a3c863/68747470733a2f2f757365722d676f6c642d63646e2e786974752e696f2f323031382f352f31362f313633363764363733623065323638643f773d37323026683d35333526663d6a70656726733d3436303831)](https://camo.githubusercontent.com/8957efacdf1cd4eac15d844da8353a7f77a3c863/68747470733a2f2f757365722d676f6c642d63646e2e786974752e696f2f323031382f352f31362f313633363764363733623065323638643f773d37323026683d35333526663d6a70656726733d3436303831)

### [二　java IO体系的学习总结](https://blog.csdn.net/nightcurtis/article/details/51324105)

1. **IO流的分类：**

   - 按照流的流向分，可以分为输入流和输出流；
   - 按照操作单元划分，可以划分为字节流和字符流；
   - 按照流的角色划分为节点流和处理流。

2. **流的原理浅析:**

   java Io流共涉及40多个类，这些类看上去很杂乱，但实际上很有规则，而且彼此之间存在非常紧密的联系， Java Io流的40多个类都是从如下4个抽象类基类中派生出来的。

   - **InputStream/Reader**: 所有的输入流的基类，前者是字节输入流，后者是字符输入流。
   - **OutputStream/Writer**: 所有输出流的基类，前者是字节输出流，后者是字符输出流。

3. **常用的io流的用法**：http://ifeve.com/java-io/

4. [IO常见面试题](https://mp.weixin.qq.com/s?__biz=MzU4NDQ4MzU5OA==&mid=2247483985&idx=1&sn=38531c2cee7b87f125df7aef41637014&chksm=fd985430caefdd26b0506aa84fc26251877eccba24fac73169a4d6bd1eb5e3fbdf3c3b940261#rd)

## 一、NIO简介

**Java NIO** 是 **java 1.4** 之后新出的一套IO接口，这里的的新是相对于原有标准的Java IO和Java Networking接口。NIO提供了一种完全不同的操作方式。

**NIO中的N可以理解为Non-blocking，不单纯是New。**

**它支持面向缓冲的，基于通道的I/O操作方法。** 随着JDK 7的推出，NIO系统得到了扩展，为文件系统功能和文件处理提供了增强的支持。 由于NIO文件类支持的这些新的功能，NIO被广泛应用于文件处理。



## 二、NIO特性、与IO的区别

> **1 Channels and Buffers（通道和缓冲区）**

**IO是面向流的，NIO是面向缓冲区的**

- 标准的IO编程接口是面向字节流和字符流的。而NIO是面向通道和缓冲区的，数据总是从通道中读到buffer缓冲区内，或者从buffer缓冲区写入到通道中；（ NIO中的所有I/O操作都是通过一个通道开始的。）
- Java IO面向流意味着每次从流中读一个或多个字节，直至读取所有字节，它们没有被缓存在任何地方；
- Java NIO是面向缓存的I/O方法。 将数据读入缓冲器，使用通道进一步处理数据。 在NIO中，使用通道和缓冲区来处理I/O操作。

> **2 Non-blocking IO（非阻塞IO）**

**IO流是阻塞的，NIO流是不阻塞的**。

- Java NIO使我们可以进行非阻塞IO操作。比如说，单线程中从通道读取数据到buffer，同时可以继续做别的事情，当数据读取到buffer中后，线程再继续处理数据。写数据也是一样的。另外，非阻塞写也是如此。一个线程请求写入一些数据到某通道，但不需要等待它完全写入，这个线程同时可以去做别的事情。
- Java IO的各种流是阻塞的。这意味着，当一个线程调用read() 或 write()时，该线程被阻塞，直到有一些数据被读取，或数据完全写入。该线程在此期间不能再干任何事情了

> **3 Selectors（选择器）**

**NIO有选择器，而IO没有。**

- 选择器用于使用单个线程处理多个通道。因此，它需要较少的线程来处理这些通道。
- 线程之间的切换对于操作系统来说是昂贵的。 因此，为了提高系统效率选择器是有用的。

## 三、NIO核心组件

NIO包含下面几个核心的组件：

- **Channels**
- **Buffers**
- **Selectors**

整个NIO体系包含的类远远不止这三个，只能说这三个是NIO体系的“核心API”。

> ### 通道

通常来说NIO中的所有IO都是从 **Channel（通道）** 开始的。

**从通道进行数据读取** ：创建一个缓冲区，然后请求通道读取数据。

**从通道进行数据写入** ：创建一个缓冲区，填充数据，并要求通道写入数据。

在Java NIO中，主要使用的通道如下（涵盖了UDP 和 TCP 网络IO，以及文件IO）：

- DatagramChannel
- SocketChannel
- FileChannel
- ServerSocketChannel

> ### 缓冲区

在Java NIO中使用的核心缓冲区如下（覆盖了通过I/O发送的基本数据类型：byte, char、short, int, long, float, double ，long）：

- ByteBuffer
- CharBuffer
- ShortBuffer
- IntBuffer
- FloatBuffer
- DoubleBuffer
- LongBuffer

> ### 选择器

Java NIO提供了“选择器”的概念。这是一个可以用于监视多个通道的对象，如数据到达，连接打开等。因此，单线程可以监视多个通道中的数据。

如果应用程序有多个通道(连接)打开，但每个连接的流量都很低，则可考虑使用它。 例如：在聊天服务器中。

下面是一个单线程中Slector维护3个Channel的示意图：

![img](https://mmbiz.qpic.cn/mmbiz_png/hvUCbRic69sAdeibvW8NVWOIZ7l70qiaNPucziaKxIh2rjTWhmo7WiaZUrrHgDrEXviaicxD4SOTIZ1dh6IicictTkay5uw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

**要使用Selector的话，我们必须把Channel注册到Selector上，然后就可以调用Selector的select()方法。这个方法会进入阻塞，直到有一个channel的状态符合条件。当方法返回后，线程可以处理这些事件。**



## Buffer

**Java NIO Buffers用于和NIO Channel交互。 我们从Channel中读取数据到buffers里，从Buffer把数据写入到Channels.**

**Buffer本质上就是一块内存区**，可以用来写入数据，并在稍后读取出来。这块内存被NIO Buffer包裹起来，对外提供一系列的读写方便开发的接口。

在Java NIO中使用的核心缓冲区如下（覆盖了通过I/O发送的基本数据类型：byte, char、short, int, long, float, double ，long）：

- **ByteBuffer**
- **CharBuffer**
- **ShortBuffer**
- **IntBuffer**
- **FloatBuffer**
- **DoubleBuffer**
- **LongBuffer**

#### 利用Buffer读写数据，通常遵循四个步骤：

1. **把数据写入buffer；**
2. **调用flip；**
3. **从Buffer中读取数据；**
4. **调用buffer.clear()或者buffer.compact()。**

当写入数据到buffer中时，buffer会记录已经写入的数据大小。当需要读数据时，通过 **flip()**方法把buffer从写模式调整为读模式；在读模式下，可以读取所有已经写入的数据。

当读取完数据后，需要清空buffer，以满足后续写入操作。清空buffer有两种方式：调用 **clear()** 或 **compact()** 方法。**clear会清空整个buffer，compact则只清空已读取的数据**，未被读取的数据会被移动到buffer的开始位置，写入位置则近跟着未读数据之后。

### Buffer的容量，位置，上限（Buffer Capacity, Position and Limit）

**Buffer缓冲区实质上就是一块内存**，用于写入数据，也供后续再次读取数据。这块内存被NIO Buffer管理，并提供一系列的方法用于更简单的操作这块内存。

一个Buffer有三个属性是必须掌握的，分别是：

- **capacity容量**
- **position位置**
- **limit限制**

position和limit的具体含义取决于当前buffer的模式。capacity在两种模式下都表示容量。

**读写模式下position和limit的含义：**

![img](https://mmbiz.qpic.cn/mmbiz_png/hvUCbRic69sAtESZIMSs3Gotl5mLMJWotpZyyPmYeywGiaq8ictezCy7RAsibqndic7tKnADtO1gjIZFhMzvJYwByicQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

> ### 容量（Capacity）

作为一块内存，buffer有一个固定的大小，叫做capacit（容量）。也就是最多只能写入容量值得字节，整形等数据。一旦buffer写满了就需要清空已读数据以便下次继续写入新的数据。

> ### 位置（Position）

**当写入数据到Buffer的时候需要从一个确定的位置开始**，默认初始化时这个位置position为0，一旦写入了数据比如一个字节，整形数据，那么position的值就会指向数据之后的一个单元，position最大可以到capacity-1.

**当从Buffer读取数据时，也需要从一个确定的位置开始。buffer从写入模式变为读取模式时，position会归零，每次读取后，position向后移动。**

> ### 上限（Limit）

在写模式，limit的含义是我们所能写入的最大数据量，它等同于buffer的容量。

一旦切换到读模式，limit则代表我们所能读取的最大数据量，他的值等同于写模式下position的位置。换句话说，您可以读取与写入数量相同的字节数（限制设置为写入的字节数，由位置标记）。

### 二 Buffer的常见方法

| 方法                             | 介绍                                                         |
| -------------------------------- | ------------------------------------------------------------ |
| abstract Object array()          | 返回支持此缓冲区的数组 （可选操作）                          |
| abstract int arrayOffset()       | 返回该缓冲区的缓冲区的第一个元素的在数组中的偏移量 （可选操作） |
| int capacity()                   | 返回此缓冲区的容量                                           |
| Buffer clear()                   | 清除此缓存区。将position = 0;limit = capacity;mark = -1;     |
| Buffer flip()                    | flip()方法可以吧Buffer从写模式切换到读模式。调用flip方法会把position归零，并设置limit为之前的position的值。 也就是说，现在position代表的是读取位置，limit标示的是已写入的数据位置。 |
| abstract boolean hasArray()      | 告诉这个缓冲区是否由可访问的数组支持                         |
| boolean hasRemaining()           | return position < limit，返回是否还有未读内容                |
| abstract boolean isDirect()      | 判断个缓冲区是否为 direct                                    |
| abstract boolean isReadOnly()    | 判断告知这个缓冲区是否是只读的                               |
| int limit()                      | 返回此缓冲区的限制                                           |
| Buffer position(int newPosition) | 设置这个缓冲区的位置                                         |
| int remaining()                  | return limit - position; 返回limit和position之间相对位置差   |
| Buffer rewind()                  | 把position设为0，mark设为-1，不改变limit的值                 |
| Buffer mark()                    | 将此缓冲区的标记设置在其位置                                 |

###三 Buffer的使用方式/方法介绍

> ### 分配缓冲区（Allocating a Buffer）

为了获得缓冲区对象，我们必须首先分配一个缓冲区。在每个Buffer类中，allocate()方法用于分配缓冲区。

下面来看看ByteBuffer分配容量为28字节的例子：

```
ByteBuffer buf = ByteBuffer.allocate(28);
```

下面来看看另一个示例：CharBuffer分配空间大小为2048个字符。

```
CharBuffer buf = CharBuffer.allocate(2048);
```

> ### 写入数据到缓冲区（Writing Data to a Buffer）

**写数据到Buffer有两种方法：**

- 从Channel中写数据到Buffer
- 手动写数据到Buffer，调用put方法

下面是一个实例，演示从Channel写数据到Buffer：

```
 int bytesRead = inChannel.read(buf); //read into buffer.
```

通过put写数据：

```
buf.put(127);
```

put方法有很多不同版本，对应不同的写数据方法。例如把数据写到特定的位置，或者把一个字节数据写入buffer。看考JavaDoc文档可以查阅的更多数据。

> ### 翻转（flip()）

flip()方法可以吧Buffer从写模式切换到读模式。调用flip方法会把position归零，并设置limit为之前的position的值。 也就是说，现在position代表的是读取位置，limit标示的是已写入的数据位置。

> ### 从Buffer读取数据（Reading Data from a Buffer）

从Buffer读数据也有两种方式。

- 从buffer读数据到channel
- 从buffer直接读取数据，调用get方法

读取数据到channel的例子：

```
int bytesWritten = inChannel.write(buf);
```

调用get读取数据的例子：

```
byte aByte = buf.get();
```

get也有诸多版本，对应了不同的读取方式。

> ### rewind()

Buffer.rewind()方法将position置为0，这样我们可以重复读取buffer中的数据。limit保持不变。

> ### clear() and compact()

一旦我们从buffer中读取完数据，需要复用buffer为下次写数据做准备。只需要调用clear（）或compact（）方法。

如果调用的是clear()方法，position将被设回0，limit被设置成 capacity的值。换句话说，Buffer 被清空了。Buffer中的数据并未清除，只是这些标记告诉我们可以从哪里开始往Buffer里写数据。

如果Buffer还有一些数据没有读取完，调用clear就会导致这部分数据被“遗忘”，因为我们没有标记这部分数据未读。

针对这种情况，如果需要保留未读数据，那么可以使用compact。 因此 **compact()** 和 **clear()** 的区别就在于: **对未读数据的处理，是保留这部分数据还是一起清空** 。

> ### mark()与reset()方法

通过调用Buffer.mark()方法，可以标记Buffer中的一个特定position。之后可以通过调用Buffer.reset()方法恢复到这个position。例如：

```
buffer.mark();//call buffer.get() a couple of times, e.g. during parsing.buffer.reset();  //set position back to mark.    
```

> ### equals() and compareTo()

可以用eqauls和compareTo比较两个buffer

**equals():**

判断两个buffer相对，需满足：

- 类型相同
- buffer中剩余字节数相同
- 所有剩余字节相等

从上面的三个条件可以看出，equals只比较buffer中的部分内容，并不会去比较每一个元素。

**compareTo():**

compareTo也是比较buffer中的剩余元素，只不过这个方法适用于比较排序的。



## 四、Channel

###一 Channel（通道）介绍

**通常来说NIO中的所有IO都是从 Channel（通道） 开始的。**

- **从通道进行数据读取** ：创建一个缓冲区，然后请求通道读取数据。
- **从通道进行数据写入** ：创建一个缓冲区，填充数据，并要求通道写入数据。

**数据读取和写入操作图示：**

![img](https://mmbiz.qpic.cn/mmbiz_png/hvUCbRic69sAlWOQIdK09StRpvYGMKZBn74j91CGPgWlMUKngCRaOKSaf4MzxMqHEVzQpy6Lo2yicl3Q1rpOk9pw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

### Java NIO Channel通道和流非常相似，主要有以下几点区别：

- 通道可以读也可以写，流一般来说是单向的（只能读或者写，所以之前我们用流进行IO操作的时候需要分别创建一个输入流和一个输出流）。
- 通道可以异步读写。
- 通道总是基于缓冲区Buffer来读写。

### Java NIO中最重要的几个Channel的实现：

- **FileChannel：** 用于文件的数据读写
- **DatagramChannel：** 用于UDP的数据读写
- **SocketChannel：** 用于TCP的数据读写，一般是客户端实现
- **ServerSocketChannel:** 允许我们监听TCP链接请求，每个请求会创建会一个SocketChannel，一般是服务器实现

### 类层次结构：

下面的UML图使用Idea生成的。![img](https://mmbiz.qpic.cn/mmbiz_png/hvUCbRic69sAlWOQIdK09StRpvYGMKZBnCOs8Tic8ANsRpZF4t1rAD4nSSgQ8IaEYNFY14BF3pldrsQbCIAMNibicw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

###二 FileChannel的使用

**使用FileChannel读取数据到Buffer（缓冲区）以及利用Buffer（缓冲区）写入数据到FileChannel：**

```
package filechannel;import java.io.IOException;import java.io.RandomAccessFile;import java.nio.ByteBuffer;import java.nio.channels.FileChannel;public class FileChannelTxt {    public static void main(String args[]) throws IOException {        //1.创建一个RandomAccessFile（随机访问文件）对象，        RandomAccessFile raf=new RandomAccessFile("D:\\niodata.txt", "rw");        //通过RandomAccessFile对象的getChannel()方法。FileChannel是抽象类。        FileChannel inChannel=raf.getChannel();        //2.创建一个读数据缓冲区对象        ByteBuffer buf=ByteBuffer.allocate(48);        //3.从通道中读取数据        int bytesRead = inChannel.read(buf);        //创建一个写数据缓冲区对象        ByteBuffer buf2=ByteBuffer.allocate(48);        //写入数据        buf2.put("filechannel test".getBytes());        buf2.flip();        inChannel.write(buf);        while (bytesRead != -1) {            System.out.println("Read " + bytesRead);            //Buffer有两种模式，写模式和读模式。在写模式下调用flip()之后，Buffer从写模式变成读模式。            buf.flip();           //如果还有未读内容            while (buf.hasRemaining()) {                System.out.print((char) buf.get());            }            //清空缓存区            buf.clear();            bytesRead = inChannel.read(buf);        }        //关闭RandomAccessFile（随机访问文件）对象        raf.close();    }}
```

**运行效果：**![img](https://mmbiz.qpic.cn/mmbiz_png/hvUCbRic69sAlWOQIdK09StRpvYGMKZBngP7uFN0ELRQcdY8biaC8zVic9oIWS3gaXzwDvZeoBV7brg69ZGsrxUCg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

### 通过上述实例代码，我们可以大概总结出FileChannel的一般使用规则：

> #### 1. 开启FileChannel

**使用之前，FileChannel必须被打开** ，但是你无法直接打开FileChannel（FileChannel是抽象类）。需要通过 **InputStream** ， **OutputStream** 或 **RandomAccessFile** 获取FileChannel。

我们上面的例子是通过RandomAccessFile打开FileChannel的：

```
        //1.创建一个RandomAccessFile（随机访问文件）对象，        RandomAccessFile raf=new RandomAccessFile("D:\\niodata.txt", "rw");        //通过RandomAccessFile对象的getChannel()方法。FileChannel是抽象类。        FileChannel inChannel=raf.getChannel();
```

> #### 2. 从FileChannel读取数据/写入数据

从FileChannel中读取数据/写入数据之前首先要创建一个Buffer（缓冲区）对象，Buffer（缓冲区）对象的使用我们在上一篇文章中已经详细说明了，如果不了解的话可以看我的上一篇关于Buffer的文章。

**使用FileChannel的read()方法读取数据：**

```
        //2.创建一个读数据缓冲区对象        ByteBuffer buf=ByteBuffer.allocate(48);        //3.从通道中读取数据        int bytesRead = inChannel.read(buf);
```

**使用FileChannel的write()方法写入数据：**

```
        //创建一个写数据缓冲区对象        ByteBuffer buf2=ByteBuffer.allocate(48);        //写入数据        buf2.put("filechannel test".getBytes());        buf2.flip();        inChannel.write(buf);
```

> #### 3. 关闭FileChannel

完成使用后，FileChannel您必须关闭它。

```
channel.close（）;    
```

###三 SocketChannel和ServerSocketChannel的使用

**利用SocketChannel和ServerSocketChannel实现客户端与服务器端简单通信：**

**SocketChannel** 用于创建基于tcp协议的客户端对象，因为SocketChannel中不存在accept()方法，所以，它不能成为一个服务端程序。通过 **connect()方法** ，SocketChannel对象可以连接到其他tcp服务器程序。

客户端:

```
package socketchannel;import java.io.IOException;import java.net.InetSocketAddress;import java.nio.ByteBuffer;import java.nio.channels.SocketChannel;public class WebClient {    public static void main(String[] args) throws IOException {        //1.通过SocketChannel的open()方法创建一个SocketChannel对象        SocketChannel socketChannel = SocketChannel.open();        //2.连接到远程服务器（连接此通道的socket）        socketChannel.connect(new InetSocketAddress("127.0.0.1", 3333));        // 3.创建写数据缓存区对象        ByteBuffer writeBuffer = ByteBuffer.allocate(128);        writeBuffer.put("hello WebServer this is from WebClient".getBytes());        writeBuffer.flip();        socketChannel.write(writeBuffer);        //创建读数据缓存区对象        ByteBuffer readBuffer = ByteBuffer.allocate(128);        socketChannel.read(readBuffer);        //String 字符串常量，不可变；StringBuffer 字符串变量（线程安全），可变；StringBuilder 字符串变量（非线程安全），可变        StringBuilder stringBuffer=new StringBuilder();        //4.将Buffer从写模式变为可读模式        readBuffer.flip();        while (readBuffer.hasRemaining()) {            stringBuffer.append((char) readBuffer.get());        }        System.out.println("从服务端接收到的数据："+stringBuffer);        socketChannel.close();    }}
```

**ServerSocketChannel** 允许我们监听TCP链接请求，通过ServerSocketChannelImpl的 **accept()方法** 可以创建一个SocketChannel对象用户从客户端读/写数据。

服务端：

```
package socketchannel;import java.io.IOException;import java.net.InetSocketAddress;import java.nio.ByteBuffer;import java.nio.channels.ServerSocketChannel;import java.nio.channels.SocketChannel;public class WebServer {    public static void main(String args[]) throws IOException {        try {            //1.通过ServerSocketChannel 的open()方法创建一个ServerSocketChannel对象，open方法的作用：打开套接字通道            ServerSocketChannel ssc = ServerSocketChannel.open();            //2.通过ServerSocketChannel绑定ip地址和port(端口号)            ssc.socket().bind(new InetSocketAddress("127.0.0.1", 3333));            //通过ServerSocketChannelImpl的accept()方法创建一个SocketChannel对象用户从客户端读/写数据            SocketChannel socketChannel = ssc.accept();            //3.创建写数据的缓存区对象            ByteBuffer writeBuffer = ByteBuffer.allocate(128);            writeBuffer.put("hello WebClient this is from WebServer".getBytes());            writeBuffer.flip();            socketChannel.write(writeBuffer);            //创建读数据的缓存区对象            ByteBuffer readBuffer = ByteBuffer.allocate(128);            //读取缓存区数据            socketChannel.read(readBuffer);            StringBuilder stringBuffer=new StringBuilder();            //4.将Buffer从写模式变为可读模式            readBuffer.flip();            while (readBuffer.hasRemaining()) {                stringBuffer.append((char) readBuffer.get());            }            System.out.println("从客户端接收到的数据："+stringBuffer);            socketChannel.close();            ssc.close();        } catch (IOException e) {            e.printStackTrace();        }    }}
```

**运行效果：**

客户端：![img](https://mmbiz.qpic.cn/mmbiz_png/hvUCbRic69sAlWOQIdK09StRpvYGMKZBnky2z71bjyCSGibwV2JGiabRC6mrN8ArialuFWY7rKcMPkJyKoO7N7KJGQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

服务端：![img](https://mmbiz.qpic.cn/mmbiz_png/hvUCbRic69sAlWOQIdK09StRpvYGMKZBnZPgyOb8QGYRT3qTIeWEsNSic3T1rIBmXVgvb0DpqFEy2FibOnlNCqoIg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

### 通过上述实例代码，我们可以大概总结出SocketChannel和ServerSocketChannel的使用的一般使用规则：

**考虑到篇幅问题，下面只给出大致步骤，不贴代码，可以结合上述实例理解。**

### 客户端

> #### 1.通过SocketChannel连接到远程服务器
>
> #### 2.创建读数据/写数据缓冲区对象来读取服务端数据或向服务端发送数据
>
> #### 3.关闭SocketChannel

### 服务端

> #### 1.通过ServerSocketChannel 绑定ip地址和端口号
>
> #### 2.通过ServerSocketChannelImpl的accept()方法创建一个SocketChannel对象用户从客户端读/写数据
>
> #### 3.创建读数据/写数据缓冲区对象来读取客户端数据或向客户端发送数据
>
> #### 4. 关闭SocketChannel和ServerSocketChannel

###四 ️DatagramChannel的使用

DataGramChannel，类似于java 网络编程的DatagramSocket类；使用UDP进行网络传输， **UDP是无连接，面向数据报文段的协议，对传输的数据不保证安全与完整** ；和上面介绍的SocketChannel和ServerSocketChannel的使用方法类似，所以这里就简单介绍一下如何使用。

> #### 1.获取DataGramChannel

```
        //1.通过DatagramChannel的open()方法创建一个DatagramChannel对象        DatagramChannel datagramChannel = DatagramChannel.open();        //绑定一个port（端口）        datagramChannel.bind(new InetSocketAddress(1234));
```

上面代码表示程序可以在1234端口接收数据报。

> #### 2.接收/发送消息

**接收消息：**

先创建一个缓存区对象，然后通过receive方法接收消息，这个方法返回一个SocketAddress对象，表示发送消息方的地址：

```
ByteBuffer buf = ByteBuffer.allocate(48);buf.clear();channel.receive(buf);
```

**发送消息：**

由于UDP下，服务端和客户端通信并不需要建立连接，只需要知道对方地址即可发出消息，但是是否发送成功或者成功被接收到是没有保证的;发送消息通过send方法发出，改方法返回一个int值，表示成功发送的字节数：

```
ByteBuffer buf = ByteBuffer.allocate(48);buf.clear();buf.put("datagramchannel".getBytes());buf.flip();int send = channel.send(buffer, new InetSocketAddress("localhost",1234));
```

这个例子发送一串字符：“datagramchannel”到主机名为”localhost”服务器的端口1234上。

###五 Scatter / Gather

Channel 提供了一种被称为 Scatter/Gather 的新功能，也称为本地矢量 I/O。Scatter/Gather 是指在多个缓冲区上实现一个简单的 I/O 操作。正确使用 Scatter / Gather可以明显提高性能。

大多数现代操作系统都支持本地矢量I/O（native vectored I/O）操作。当您在一个通道上请求一个Scatter/Gather操作时，该请求会被翻译为适当的本地调用来直接填充或抽取缓冲区，减少或避免了缓冲区拷贝和系统调用；

Scatter/Gather应该使用直接的ByteBuffers以从本地I/O获取最大性能优势。

**Scatter/Gather功能是通道(Channel)提供的 并不是Buffer。**

- **Scatter:** 从一个Channel读取的信息分散到N个缓冲区中(Buufer).
- **Gather:** 将N个Buffer里面内容按照顺序发送到一个Channel.

### Scattering Reads

"scattering read"是把数据从单个Channel写入到多个buffer,如下图所示：

![img](https://mmbiz.qpic.cn/mmbiz_png/hvUCbRic69sAlWOQIdK09StRpvYGMKZBn2GfeqkBEQa2jvIfVicxk8ia4MyUibJpaecvd8r5AXibHPkbEeHe9Dawciag/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)示例代码：

```
ByteBuffer header = ByteBuffer.allocate(128);ByteBuffer body   = ByteBuffer.allocate(1024);ByteBuffer[] bufferArray = { header, body };channel.read(bufferArray);
```

read()方法内部会负责把数据按顺序写进传入的buffer数组内。一个buffer写满后，接着写到下一个buffer中。

举个例子，假如通道中有200个字节数据，那么header会被写入128个字节数据，body会被写入72个字节数据；

**注意：**

无论是scatter还是gather操作，都是按照buffer在数组中的顺序来依次读取或写入的；

### Gathering Writes

"gathering write"把多个buffer的数据写入到同一个channel中，下面是示意图：![img](https://mmbiz.qpic.cn/mmbiz_png/hvUCbRic69sAlWOQIdK09StRpvYGMKZBn1TR51MFuqvNU4p4iblWhPaWkOR4MBg68rZJmDkpT11uXOdmC4Xxyianw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

示例代码：

```
ByteBuffer header = ByteBuffer.allocate(128);ByteBuffer body   = ByteBuffer.allocate(1024);//write data into buffersByteBuffer[] bufferArray = { header, body };channel.write(bufferArray);
```

write()方法内部会负责把数据按顺序写入到channel中。

**注意：**

并不是所有数据都写入到通道，写入的数据要根据position和limit的值来判断，只有position和limit之间的数据才会被写入；

举个例子，假如以上header缓冲区中有128个字节数据，但此时position=0，limit=58；那么只有下标索引为0-57的数据才会被写入到通道中。

###六 通道之间的数据传输

在Java NIO中如果一个channel是FileChannel类型的，那么他可以直接把数据传输到另一个channel。

- **transferFrom()** :transferFrom方法把数据从通道源传输到FileChannel
- **transferTo()** :transferTo方法把FileChannel数据传输到另一个channel



## Selector

## 一 Selector（选择器）介绍

**Selector** 一般称 为**选择器** ，当然你也可以翻译为 **多路复用器** 。它是Java NIO核心组件中的一个，用于检查一个或多个NIO Channel（通道）的状态是否处于可读、可写。如此可以实现单线程管理多个channels,也就是可以管理多个网络链接。



**使用Selector的好处在于：** 使用更少的线程来就可以来处理通道了， 相比使用多个线程，避免了线程上下文切换带来的开销。

## 二 Selector（选择器）的使用方法介绍

### 1. Selector的创建

通过调用Selector.open()方法创建一个Selector对象，如下：

```
Selector selector = Selector.open();
```

这里需要说明一下

### 2. 注册Channel到Selector

```
channel.configureBlocking(false);SelectionKey key = channel.register(selector, Selectionkey.OP_READ);
```

**Channel必须是非阻塞的**。 所以FileChannel不适用Selector，因为FileChannel不能切换为非阻塞模式，更准确的来说是因为FileChannel没有继承SelectableChannel。Socket channel可以正常使用。

**SelectableChannel抽象类** 有一个 **configureBlocking（）** 方法用于使通道处于阻塞模式或非阻塞模式。

```
abstract SelectableChannel configureBlocking(boolean block)  
```

### **注意：**

**SelectableChannel抽象类**的**configureBlocking（）** 方法是由 **AbstractSelectableChannel抽象类**实现的，**SocketChannel、ServerSocketChannel、DatagramChannel**都是直接继承了 **AbstractSelectableChannel抽象类** 。 大家有兴趣可以看看NIO的源码，各种抽象类和抽象类上层的抽象类。我本人暂时不准备研究NIO源码，因为还有很多事情要做，需要研究的同学可以自行看看。

**register()** 方法的第二个参数。这是一个“ **interest集合** ”，意思是在**通过Selector监听Channel时对什么事件感兴趣**。可以监听四种不同类型的事件：

- **Connect**
- **Accept**
- **Read**
- **Write**

通道触发了一个事件意思是该事件已经就绪。比如某个Channel成功连接到另一个服务器称为“ **连接就绪** ”。一个Server Socket Channel准备好接收新进入的连接称为“ **接收就绪**”。一个有数据可读的通道可以说是“ **读就绪** ”。等待写数据的通道可以说是“ **写就绪**”。

这四种事件用SelectionKey的四个常量来表示：

```
SelectionKey.OP_CONNECTSelectionKey.OP_ACCEPTSelectionKey.OP_READSelectionKey.OP_WRITE
```

如果你对不止一种事件感兴趣，使用或运算符即可，如下：

```
int interestSet = SelectionKey.OP_READ | SelectionKey.OP_WRITE;
```

### 3. SelectionKey介绍

一个SelectionKey键表示了一个特定的通道对象和一个特定的选择器对象之间的注册关系。

```
key.attachment(); //返回SelectionKey的attachment，attachment可以在注册channel的时候指定。key.channel(); // 返回该SelectionKey对应的channel。key.selector(); // 返回该SelectionKey对应的Selector。key.interestOps(); //返回代表需要Selector监控的IO操作的bit maskkey.readyOps(); // 返回一个bit mask，代表在相应channel上可以进行的IO操作。
```

> **key.interestOps():**

我们可以通过以下方法来判断Selector是否对Channel的某种事件感兴趣

```
int interestSet = selectionKey.interestOps(); boolean isInterestedInAccept = (interestSet & SelectionKey.OP_ACCEPT) == SelectionKey.OP_ACCEPT；boolean isInterestedInConnect = interestSet & SelectionKey.OP_CONNECT;boolean isInterestedInRead = interestSet & SelectionKey.OP_READ;boolean isInterestedInWrite = interestSet & SelectionKey.OP_WRITE;
```

> **key.readyOps()**

ready 集合是通道已经准备就绪的操作的集合。JAVA中定义以下几个方法用来检查这些操作是否就绪.

```
//创建ready集合的方法int readySet = selectionKey.readyOps();//检查这些操作是否就绪的方法key.isAcceptable();//是否可读，是返回 trueboolean isWritable()：//是否可写，是返回 trueboolean isConnectable()：//是否可连接，是返回 trueboolean isAcceptable()：//是否可接收，是返回 true
```

**从SelectionKey访问Channel和Selector很简单。如下：**

```
Channel channel = key.channel();Selector selector = key.selector();key.attachment();
```

可以将一个对象或者更多信息附着到SelectionKey上，这样就能方便的识别某个给定的通道。例如，可以附加 与通道一起使用的Buffer，或是包含聚集数据的某个对象。使用方法如下：

```
key.attach(theObject);Object attachedObj = key.attachment();
```

还可以在用register()方法向Selector注册Channel的时候附加对象。如：

```
SelectionKey key = channel.register(selector, SelectionKey.OP_READ, theObject);
```

### 4. 从Selector中选择channel(Selecting Channels via a Selector)

选择器维护注册过的通道的集合，并且这种注册关系都被封装在SelectionKey当中.

> **Selector维护的三种类型SelectionKey集合：**

- 

  **已注册的键的集合(Registered key set)**

  所有与选择器关联的通道所生成的键的集合称为已经注册的键的集合。并不是所有注册过的键都仍然有效。这个集合通过 **keys()** 方法返回，并且可能是空的。这个已注册的键的集合不是可以直接修改的；试图这么做的话将引发java.lang.UnsupportedOperationException。

- 

  **已选择的键的集合(Selected key set)**

  所有与选择器关联的通道所生成的键的集合称为已经注册的键的集合。并不是所有注册过的键都仍然有效。这个集合通过 **keys()** 方法返回，并且可能是空的。这个已注册的键的集合不是可以直接修改的；试图这么做的话将引发java.lang.UnsupportedOperationException。

- 

  **已取消的键的集合(Cancelled key set)**

  已注册的键的集合的子集，这个集合包含了 **cancel()** 方法被调用过的键(这个键已经被无效化)，但它们还没有被注销。这个集合是选择器对象的私有成员，因而无法直接访问。

  **注意：**当键被取消（ 可以通过**isValid( )** 方法来判断）时，它将被放在相关的选择器的已取消的键的集合里。注册不会立即被取消，但键会立即失效。当再次调用 **select( )**方法时（或者一个正在进行的select()调用结束时），已取消的键的集合中的被取消的键将被清理掉，并且相应的注销也将完成。通道会被注销，而新的SelectionKey将被返回。当通道关闭时，所有相关的键会自动取消（记住，一个通道可以被注册到多个选择器上）。当选择器关闭时，所有被注册到该选择器的通道都将被注销，并且相关的键将立即被无效化（取消）。一旦键被无效化，调用它的与选择相关的方法就将抛出CancelledKeyException。


> **select()方法介绍：**

在刚初始化的Selector对象中，这三个集合都是空的。 **通过Selector的select（）方法可以选择已经准备就绪的通道** （这些通道包含你感兴趣的的事件）。比如你对读就绪的通道感兴趣，那么select（）方法就会返回读事件已经就绪的那些通道。下面是Selector几个重载的select()方法：

- int select()：阻塞到至少有一个通道在你注册的事件上就绪了。
- int select(long timeout)：和select()一样，但最长阻塞时间为timeout毫秒。
- int selectNow()：非阻塞，只要有通道就绪就立刻返回。

**select()方法返回的int值表示有多少通道已经就绪,是自上次调用select()方法后有多少通道变成就绪状态。之前在select（）调用时进入就绪的通道不会在本次调用中被记入，而在前一次select（）调用进入就绪但现在已经不在处于就绪的通道也不会被记入**。例如：首次调用select()方法，如果有一个通道变成就绪状态，返回了1，若再次调用select()方法，如果另一个通道就绪了，它会再次返回1。如果对第一个就绪的channel没有做任何操作，现在就有两个就绪的通道，但在每次select()方法调用之间，只有一个通道就绪了。

**一旦调用select()方法，并且返回值不为0时，则 可以通过调用Selector的selectedKeys()方法来访问已选择键集合** 。如下：  Set selectedKeys=selector.selectedKeys();  进而可以放到和某SelectionKey关联的Selector和Channel。如下所示：

```java
Set selectedKeys = selector.selectedKeys();
Iterator keyIterator = selectedKeys.iterator();
while(keyIterator.hasNext()) {    
    SelectionKey key = keyIterator.next();    
    if(key.isAcceptable()) {        
    // a connection was accepted by a ServerSocketChannel.    
    } else if (key.isConnectable()) {
    // a connection was established with a remote server.    
    } else if (key.isReadable()) { 
    // a channel is ready for reading  
    } else if (key.isWritable()) {      
    // a channel is ready for writing  
    }    
    keyIterator.remove();
}
```

### 5. 停止选择的方法

选择器执行选择的过程，系统底层会依次询问每个通道是否已经就绪，这个过程可能会造成调用线程进入阻塞状态,那么我们有以下三种方式可以唤醒在select（）方法中阻塞的线程。

- **wakeup()方法** ：通过调用Selector对象的wakeup（）方法让处在阻塞状态的select()方法立刻返回 该方法使得选择器上的第一个还没有返回的选择操作立即返回。如果当前没有进行中的选择操作，那么下一次对select()方法的一次调用将立即返回。
- **close()方法** ：通过close（）方法关闭Selector， 该方法使得任何一个在选择操作中阻塞的线程都被唤醒（类似wakeup（）），同时使得注册到该Selector的所有Channel被注销，所有的键将被取消，但是Channel本身并不会关闭。

## 三 模板代码

**一个服务端的模板代码：**

有了模板代码我们在编写程序时，大多数时间都是在模板代码中添加相应的业务代码

```java
ServerSocketChannel ssc = ServerSocketChannel.open();
ssc.socket().bind(new InetSocketAddress("localhost",8080));
ssc.configureBlocking(false);
Selector selector = Selector.open();ssc.register(selector,SelectionKey.OP_ACCEPT);
while(true) {    
    int readyNum = selector.select();    
    if (readyNum == 0) {        
    	continue;    
    }    
    Set<SelectionKey> selectedKeys = selector.selectedKeys();  
    Iterator<SelectionKey> it = selectedKeys.iterator();    
    while(it.hasNext()) {       
    SelectionKey key = it.next();        
    if(key.isAcceptable()) {          
    // 接受连接    
    } else if (key.isReadable()) {    
    // 通道可读       
    } else if (key.isWritable()) {     
    // 通道可写        
    }       
    	it.remove();
    }
}
```

## 四 客户端与服务端简单交互实例

**服务端：**

```java
package selector;import java.io.IOException;
import java.net.InetSocketAddress;import java.nio.ByteBuffer;
import java.nio.channels.SelectionKey;
import java.nio.channels.Selector;
import java.nio.channels.ServerSocketChannel;
import java.nio.channels.SocketChannel;
import java.util.Iterator;
import java.util.Set;
public class WebServer {    
    public static void main(String[] args) {       
    try {            
        ServerSocketChannel ssc = ServerSocketChannel.open();  
        ssc.socket().bind(new InetSocketAddress("127.0.0.1", 8000));            		    ssc.configureBlocking(false);           
        Selector selector = Selector.open();        
        // 注册 channel，并且指定感兴趣的事件是 Accept            
        ssc.register(selector, SelectionKey.OP_ACCEPT);            
        ByteBuffer readBuff = ByteBuffer.allocate(1024);          
        ByteBuffer writeBuff = ByteBuffer.allocate(128);                          			writeBuff.put("received".getBytes());            
        writeBuff.flip();            
        while (true) {                
            int nReady = selector.select();         
            Set<SelectionKey> keys = selector.selectedKeys();        
            Iterator<SelectionKey> it = keys.iterator();         
            while (it.hasNext()) {                  
            SelectionKey key = it.next();                 
            it.remove();                   
            if (key.isAcceptable()) {                   
            // 创建新的连接，并且把连接注册到selector上，而且，                      
            // 声明这个channel只对读操作感兴趣。                      
                SocketChannel socketChannel = ssc.accept();                            					socketChannel.configureBlocking(false);                        						socketChannel.register(selector, SelectionKey.OP_READ);          
        	}                    
            else if (key.isReadable()) {       
                SocketChannel socketChannel = (SocketChannel) key.channel();                       	   readBuff.clear();             
                socketChannel.read(readBuff);    
                readBuff.flip();                     
                System.out.println("received : " + new String(readBuff.array()));                       key.interestOps(SelectionKey.OP_WRITE);                   
            }                 
            else if (key.isWritable()) {                 
                writeBuff.rewind();                    
                SocketChannel socketChannel = (SocketChannel) key.channel();                        	socketChannel.write(writeBuff);                        			            			key.interestOps(SelectionKey.OP_READ);   
            }               
          }          
        }       
     } catch (IOException e) {      
           e.printStackTrace();       
     }  
    }
}
```

**客户端：**

```JAVA
package selector;import java.io.IOException;
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.SocketChannel;
public class WebClient {    
    public static void main(String[] args) throws IOException {        
        try {            
            SocketChannel socketChannel = SocketChannel.open(); 
            socketChannel.connect(new InetSocketAddress("127.0.0.1", 8000)); 
            ByteBuffer writeBuffer = ByteBuffer.allocate(32);      
            ByteBuffer readBuffer = ByteBuffer.allocate(32);     
            writeBuffer.put("hello".getBytes());         
            writeBuffer.flip();         
            while (true) {        
                writeBuffer.rewind();   
                socketChannel.write(writeBuffer); 
                readBuffer.clear();           
                socketChannel.read(readBuffer); 
            }       
        } catch (IOException e) { 
        }  
    }
}
```

**运行结果：**

先运行服务端，再运行客户端，服务端会不断收到客户端发送过来的消息。



## 一 文件I/O基石：Path

Java7中文件IO发生了很大的变化，专门引入了很多新的类来取代原来的基于java.io.File的文件IO操作方式：

```
import java.nio.file.DirectoryStream;
import java.nio.file.FileSystem;
import java.nio.file.FileSystems;
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.Paths;
import java.nio.file.attribute.FileAttribute;
import java.nio.file.attribute.PosixFilePermission;
import java.nio.file.attribute.PosixFilePermissions;·

......等等
```



我们将从下面几个方面来学习Path类:

- 创建一个Path
- File和Path之间的转换，File和URI之间的转换
- 获取Path的相关信息
- 移除Path中的冗余项

### 1 创建一个Path

创建Path实例可以通过 **Paths工具类** 的 **get（）方法：**

```
//使用绝对路径
 Path path= Paths.get("c:\\data\\myfile.txt");
//使用相对路径
Path path = Paths.get("/home/jakobjenkov/myfile.txt");
```

下面这种创建方式和上面等效：

```
Path path = FileSystems.getDefault().getPath("c:\\data\\myfile.txt");
```



### 2 File和Path之间的转换，File和URI之间的转换

```
File file = new File("C:/my.ini");
Path p1 = file.toPath();
p1.toFile();
file.toURI();
```

### 3 获取Path的相关信息

```
        //使用Paths工具类的get()方法创建
        Path path = Paths.get("D:\\XMind\\bcl-java.txt");
/*        //使用FileSystems工具类创建
        Path path2 = FileSystems.getDefault().getPath("c:\\data\\myfile.txt");*/
        System.out.println("文件名：" + path.getFileName());
        System.out.println("名称元素的数量：" + path.getNameCount());
        System.out.println("父路径：" + path.getParent());
        System.out.println("根路径：" + path.getRoot());
        System.out.println("是否是绝对路径：" + path.isAbsolute());
        //startsWith()方法的参数既可以是字符串也可以是Path对象
        System.out.println("是否是以为给定的路径D:开始：" + path.startsWith("D:\\") );
        System.out.println("该路径的字符串形式：" + path.toString());
```

**结果：**

```
文件名：bcl-java.txt
名称元素的数量：2
父路径：D:\XMind
根路径：D:\
是否是绝对路径：true
是否是以为给定的路径D:开始：true
该路径的字符串形式：D:\XMind\bcl-java.txt
```



### 4 移除冗余项

某些时候在我们需要处理的Path路径中可能会有一个或两个点

- .表示的是当前目录
- ..表示父目录或者说是上一级目录：

下面通过实例来演示一下使用Path类的normalize()和toRealPath()方法把.和..去除。

- **normalize()** : 返回一个路径，该路径是冗余名称元素的消除。
- **toRealPath()** : 融合了toAbsolutePath()方法和normalize()方法

```
//.表示的是当前目录
Path currentDir = Paths.get(".");
System.out.println(currentDir.toAbsolutePath());//输出C:\Users\Administrator\NIODemo\.
Path currentDir2 = Paths.get(".\\NIODemo.iml");
System.out.println("原始路径格式："+currentDir2.toAbsolutePath());
System.out.println("执行normalize（）方法之后："+currentDir2.toAbsolutePath().normalize());
System.out.println("执行toRealPath()方法之后："+currentDir2.toRealPath());

//..表示父目录或者说是上一级目录：
Path currentDir3 = Paths.get("..");
System.out.println("原始路径格式："+currentDir3.toAbsolutePath());
System.out.println("执行normalize（）方法之后："+currentDir3.toAbsolutePath().normalize());
System.out.println("执行toRealPath()方法之后："+currentDir3.toRealPath());
```

**结果：**

```
C:\Users\Administrator\NIODemo\.
原始路径格式：C:\Users\Administrator\NIODemo\.\NIODemo.iml
执行normalize（）方法之后：C:\Users\Administrator\NIODemo\NIODemo.iml
执行toRealPath()方法之后：C:\Users\Administrator\NIODemo\NIODemo.iml
原始路径格式：C:\Users\Administrator\NIODemo\..
执行normalize（）方法之后：C:\Users\Administrator
执行toRealPath()方法之后：C:\Users\Administrator
```



[![NIODemo.iml文件的位置](https://user-gold-cdn.xitu.io/2018/5/16/1636923259d41d36?w=577&h=164&f=png&s=14767)](https://user-gold-cdn.xitu.io/2018/5/16/1636923259d41d36?w=577&h=164&f=png&s=14767)NIODemo.iml文件的位置

## 二 拥抱Files类

Java NIO中的Files类（java.nio.file.Files）提供了多种操作文件系统中文件的方法。本节教程将覆盖大部分方法。Files类包含了很多方法，所以如果本文没有提到的你也可以直接查询JavaDoc文档。

java.nio.file.Files类是和java.nio.file.Path相结合使用的

### 1 检查给定的Path在文件系统中是否存在

通过 **Files.exists()** 检测文件路径是否存在：

```
Path path = Paths.get("D:\\XMind\\bcl-java.txt");

 boolean pathExists =
         Files.exists(path,
                 new LinkOption[]{LinkOption.NOFOLLOW_LINKS});
 System.out.println(pathExists);//true
```



注意Files.exists()的的第二个参数。它是一个数组，这个参数直接影响到Files.exists()如何确定一个路径是否存在。在本例中，这个数组内包含了LinkOptions.NOFOLLOW_LINKS，表示检测时不包含符号链接文件。

### 2 创建文件/文件夹

- **创建文件：**

通过 **Files.createFile()** 创建文件,

```
Path target2 = Paths.get("C:\\mystuff.txt");
try {
    if(!Files.exists(target2))
        Files.createFile(target2);
} catch (IOException e) {
    e.printStackTrace();
}
```



- **创建文件夹：**

  - 通过 **Files.createDirectory()** 创建文件夹
  - 通过 **Files.createDirectories()** 创建文件夹

  Files.createDirectories()会首先创建所有不存在的父目录来创建目录，而Files.createDirectory()方法只是创建目录，如果它的上级目录不存在就会报错。比如下面的程序使用**Files.createDirectory()** 方法创建就会报错，这是因为我的D盘下没有data文件夹，加入存在data文件夹的话则没问题。

  ```
  Path path = Paths.get("D://data//test");
  
  try {
      Path newDir = Files.createDirectories(path);
  } catch(FileAlreadyExistsException e){
      // the directory already exists.
  } catch (IOException e) {
      //something else went wrong
      e.printStackTrace();
  }
  ```

### 3 删除文件或目录

通过 **Files.delete()方法** 可以删除一个文件或目录：

```
Path path = Paths.get("data/subdir/logging-moved.properties");

try {
    Files.delete(path);
} catch (IOException e) {
    //deleting file failed
    e.printStackTrace();
}
```



### 4 把一个文件从一个地址复制到另一个位置

通过Files.copy()方法可以吧一个文件从一个地址复制到另一个位置

```
Path sourcePath      = Paths.get("data/logging.properties");
Path destinationPath = Paths.get("data/logging-copy.properties");

try {
    Files.copy(sourcePath, destinationPath);
} catch(FileAlreadyExistsException e) {
    //destination file already exists
} catch (IOException e) {
    //something else went wrong
    e.printStackTrace();
}
```

copy操作还可可以强制覆盖已经存在的目标文件，只需要将上面的copy()方法改为如下格式：

```
Files.copy(sourcePath, destinationPath,
        StandardCopyOption.REPLACE_EXISTING);
```



### 5 获取文件属性

```
Path path = Paths.get("D:\\XMind\\bcl-java.txt");
System.out.println(Files.getLastModifiedTime(path));
System.out.println(Files.size(path));
System.out.println(Files.isSymbolicLink(path));
System.out.println(Files.isDirectory(path));
System.out.println(Files.readAttributes(path, "*"));
```

结果：

```
2016-05-18T08:01:44Z
18934
false
false
{lastAccessTime=2017-04-12T01:42:21.149351Z, lastModifiedTime=2016-05-18T08:01:44Z, size=18934, creationTime=2017-04-12T01:42:21.149351Z, isSymbolicLink=false, isRegularFile=true, fil
```



### 6 遍历一个文件夹

```
Path dir = Paths.get("D:\\Java");
try(DirectoryStream<Path> stream = Files.newDirectoryStream(dir)){
    for(Path e : stream){
        System.out.println(e.getFileName());
    }
}catch(IOException e){

}
```

结果：

```
apache-maven-3.5.0
Eclipse
intellij idea
Jar
JDK
MarvenRespository
MyEclipse 2017 CI
Nodejs
RedisDesktopManager
solr-7.2.1
```



上面是遍历单个目录，它不会遍历整个目录。遍历整个目录需要使用：Files.walkFileTree().Files.walkFileTree()方法具有递归遍历目录的功能。

### 7 遍历整个文件目录：

walkFileTree接受一个Path和FileVisitor作为参数。Path对象是需要遍历的目录，FileVistor则会在每次遍历中被调用。

FileVisitor需要调用方自行实现，然后作为参数传入walkFileTree().FileVisitor的每个方法会在遍历过程中被调用多次。如果不需要处理每个方法，那么可以继承它的默认实现类SimpleFileVisitor，它将所有的接口做了空实现。

```
public class WorkFileTree {

    public static void main(String[] args) throws IOException{
        Path startingDir = Paths.get("D:\\apache-tomcat-9.0.0.M17");
        List<Path> result = new LinkedList<Path>();
        Files.walkFileTree(startingDir, new FindJavaVisitor(result));
        System.out.println("result.size()=" + result.size());
    }

    private static class FindJavaVisitor extends SimpleFileVisitor<Path>{
        private List<Path> result;
        public FindJavaVisitor(List<Path> result){
            this.result = result;
        }
        @Override
        public FileVisitResult visitFile(Path file, BasicFileAttributes attrs){
            if(file.toString().endsWith(".java")){
                result.add(file.getFileName());
            }
            return FileVisitResult.CONTINUE;
        }
    }
}
```

上面这个例子输出了我的D:\apache-tomcat-9.0.0.M17也就是我的Tomcat安装目录下以.java结尾文件的数量。

**结果：**

```
result.size()=4
```



Files类真的很强大，除了我讲的这些操作之外还有其他很多操作比如：读取和设置文件权限、更新文件所有者等等操作。

我这里就介绍这么多了，如果想要详细了解的可以自行查阅官方文档或者相关书籍。