**通用资源标识符URI**

一个简单的格式化字符串--通过名称位置或其他特征识别一个资源

HTTP协议不对URI的长度作事先的限制，服务器必须能够处理任何他们提供资源的URI，并且应该能够处理无限长度的URIs，这种无效长度的URL可能会在客户端以基于GET方式的请求时产生。如果服务器不能处理太长的URI的时候，服务器就应该返回**414状态码**（此状态码代表Request-URI太长）



HTTP URL

在HTTP协议里，http模式（http scheme）被用于定位网络资源（resourse）位置。本节定义了http URLs这种特定模式（scheme）的语法和定义。

```http
http_URL="http://"host [":" port][abs_path["?"query]]
```

如果端口为空或未给出，就假定为80.

**URI比较**

当比较两个URI是否匹配时，客户应该对整个URI比较时应该区分大小写，并且一个字节一个字节比较，但下面有些特殊情况：

- 一个为空或未给定的端口等同于URI-refernece里默认的端口；
- 主机（host）名的比较必须不区分大小写
- 模式（scheme）名的比较必须是不区分大小写的
- 一个空的绝对路径(abs_path)等同于“/”

除了“保留（reserved）”和“不安全（unsafe）”字符集里的字符，其它字符和它们的“%HEXHEX”编码的效果一样。

例如：下面三个URI是等同的：

```http
http://abc.com:80/~smith/home.html
http://ABC.com/%7Rsmith/home.html
http://ABC.com:/%7esmith/home.html
```

消息主体：
对于响应消息，消息里是否包含消息主题依赖相应的请求方法和响应状态码。所有HEAD请求方法的请求的响应消息不能包含消息主体，即时实体头域出现在请求里。所有1XX（消息的），204（无内容的）和304（没有修改的）的响应都不能包括一个消息实体（message-body）。所有其他响应必须包含消息主体，即时它的长度可能为0.

信息长度（Message Length）

为了与HTTP/1.0应用程序兼容，包含HTTP/1.1消息主体的请求必须包括一个有效的内容长度（Content-Length）头域，除非服务器是HTTP/1.1遵循的。如果一个请求包括一个消息主体并且没有给出内容长度（Content-Length），那么服务器如果不能判断消息长度的话应该以400响应（错误的请求），或者以411响应（要求长度）如果它坚持想要收到一个有效内容长度（Content-length）



##消息头：
HTTP头域包括常用头域，请求头域，响应头域和实体头域。

###常用头域（general Header Fields）

有一些头域既适用于响应消息，但这些头域并不适合传输实体。这些头域只能应用于传输消息。

```http
general-header = Cache-Control ; Section 14.9| Connection ; Section 14.10
| Date ; Section 14.18
| Pragma ; Section 14.32
| Trailer ; Section 14.40
| Transfer-Encoding ; Section 14.41
| Upgrade ; Section 14.42
| Via ; Section 14.45
| Warning ; Section 14.46
```

常用头域名能被扩展，但这要和协议版本的变化相结合。然而，如果通信里的所有参与者都认同新的或实践性的头域是常用头域，那么它们可能就被赋于常用头域的语意。不被识别的头域会被作为实体头（entity-header）头域来看待 .

##**请求（Request）**

一个请求消息是从客户端到服务端的，在消息首行里包含方法，资源指示符，协议版本

```http
Request=Request--Line;
*((genearl-header;
|request-header;
|entity-header)CRLF);
CRLF
[message-body];
```

###请求行（Request-Line）

```http
Request-Line=Method SP Request-URL SP HTTP-Vesion CRLF
```

####方法

```HTTP
Mrthod = "OPTIONS"
|"GET"|"HEAD"|"POST"|"PUT"|"DELETE"|"TRACE"|"CONNET"|extension-method
```

资源所允许的方法有Allow头域指定。响应的返回码总是通知客户某个方法对当前资源是否被允许的，因为被允许的方法能被动态的改变。如果服务器能理解某方法但此方法对请求资源不被允许的，那么源服务器应该返回405状态码（方法不被允许）；如果源服务器不能识别或没有实现某个方法，那么服务器应该返回501状态码（没有实现）。方法GET和HEAD必须被所有一般的服务器支持。所有其他的方法是可选的。

#### 请求URL（Request-URI）

一个源服务器如果必须基于请求主机来区分资源，那么对HTTP/1.1请求，它必须遵循下面的规则去决定请求的资源：
1.如果Request-URI是绝对地址（absoluteURI）那么主机（host）是Request-URI的一部分，任何出现在请求里Host头域的值决定

2.假如Request-URI不是绝对地址，并且请求包括一个Host头域，则主机（Host）由该Host头域的值决定

3.假如由规则1或规则2定义的主机（host）对服务器来说是一个无效的主机（host），则应当以一个400（坏请求）错误消息返回。

缺失Host头域的HTTP/1.0请求的接收者可能会尝试去利用启发式的规则去决定被请求的真正资源。

#### 请求头域（RequestHeaderFields）

请求头域允许客户端传递请求的附加信息和客户端自己的附加信息给服务器。这些头域作为请求的修饰，这和程序语言方法调用的参数语义是等价的。

```http
请求头（request-header） = Accept
|Accept-Charset
|Accept-Encoding
|Authorization
|Expect
|Form
|Host
|if-Match
|if-Modified-Since
|if-None-Match
|if-Range
|if-Unmodified-Since
|if-Node-Match
|if-Range
|if-Unmodified-Since
|Max-Forwards
|Proxy-Authorization
|Range
|Referer
|TE
|User-Agent
```

不能识别的头域会被看做实体头域（entity-header）。

## 响应（Response）

接收和解析一个请求消息后，服务器发出一个HTTP响应信息

```http
response =Status-Line;
*((general-header)
|response-header
|entity-header)CRLF)
CRLF
[message-body]
```

### 状态行（Status-Line）

```http
Status-Line=HTTP-Version SP Status-Code SP Reason-Phrase CRLF
```

#### 状态码与原因短语

-1XX：报告的 -请求被接受到请继续处理

-2XX：成功 -被成功接收（received），理解（understood），接收（accepeted）的动作

-3xx 重发 -为了完成请求必须采取进一步的动作。

-4xx：客户端错误 -请求包括错的语法或不能被满足

-5xx：服务器出错 --服务器无法完成显然有效的请求。

```http
Status-Code=
"100" ; 10.1.1 节: 继续
|"101" ; 10.1.2 节: 转换协议
|"200" ; 10.2.1 节: OK
|"201" ; 10.2.2 节: 已创建
|"202" ; 10.2.3 节: 接受
|"203" ; 10.2.4 节: 非权威信息|"204" ; 10.2.5 节: 无内容
|"205" ; 10.2.6 节: 重置内容
|"206" ; 10.2.7 节: 部分内容
|"300" ; 10.3.1 节: 多个选择
|"301" ; 10.3.2 节: 永久移动
|"302" ; 10.3.3 节: 发现
|"303" ; 10.3.4 节: 见其它
|"304" ; 10.3.5 节: 没有被改变
|"305" ; 10.3.6 节: 使用代理
|"307" ; 10.3.8 节 临时重发
|"400" ; 10.4.1 节: 坏请求
|"401" ; 10.4.2 节: 未授权的
|"402" ; 10.4.3 节: 必要的支付
|"403" ; 10.4.4 节: 禁用
|"404" ; 10.4.5 节: 没有找到
|"405" ; 10.4.6 节: 方式不被允许
|"406" ; 10.4.7 节: 不接受的
|"407" ; 10.4.8 节: 需要代理验证
|"408" ; 10.4.9 节： 请求超时
|"409" ; 10.4.10 节； 冲突
|"410" ; 10.4.11 节： 不存在
|"411" ; 10.4.12 节： 长度必需
|"412" ； 10.4.13 节；先决条件失败
|"413" ; 10.4.14 节： 请求实体太大
|"414" ; 10.4.15 节； 请求 URI 太大
|"415" ; 10.4.16 节： 不被支持的媒体类型
|"416" ； 10.4.17 节： 请求的范围不满足
|"417" ; 10.4.18 节： 期望失败
|"500" ; 10.5.1 节: 服务器内部错误
|"501" ; 10.5.2 节: 不能实现
|"502" ; 10.5.3 节: 坏网关
|"503" ; 10.5.4 节: 服务不能获得
|"504" ; 10.5.5 节: 网关超时
|"505" ; 10.5.6 节: HTTP 版本不支持
|扩展码
```

HTTP 状态码是可扩展的。HTTP 应用程序不需要理解所有已注册状态码的含义，尽管那样的理
解是很希望的。但是，应用程序必须了解由第一位数字指定的状态码的类别，任何未被识别的
响应应被看作是那个类别的 x00 状态码，未被识别的响应不能被缓存除外。例如，如果客户端
收到一个未被识别的状态码 431，则可以安全的认为请求有错，并且它会对待此响应就像它接
收了一个状态码是 400 的响应。在这种情况下，用户代理（user agent）应当把响应的实体展
现给用户，因为实体有可能包括人类可读的信息，这些信息也许能解释非正常状态的原因。 



#### 响应头域（Response Header Fields）

响应头域允许服务器响应附加信息，这些信息不能放在状态行（Status-Line）里。这些头域给出有关服务器的信息以及请求URI（Request-URI）指定资源的更进一步访问信息

```HTTP
response-header=Accept-Ranges
|Age
|Etag
|Location
|Proxy-Autenticate
|Retry-After
|Server
|Vary
|WWW-Authenticate
```

不被识别的头域被看作实体头域



## 实体（Entity）

如果不被请求方法或响应状态码所限制，请求和响应消息都可以传输实体。 实体包括实体头域
（entity-header）与实体主体（entity-body），而有些响应只包括实体头域（entity-header）。
在本节中的发送者和接收者是否是客户端或服务器，这依赖于谁发送或谁接收此实体。 

###实体头域（Entity Header Fields） 

实体（entity-header）头域定义了关于实体主体的的元信息，或在无主体的情况下定义了请求
的资源的元信息。有些元信息是可选的；一些是必须的。 

```http
entity-header = Allow ;
| Content-Encoding ;
| Content-Language ; 
| Content-Length ; Section 
| Content-Location ; 
| Content-MD5 ; 
| Content-Range ;
| Content-Type ;
| Expires ; 
| Last-Modified ;
| extension-header

extension-header = message-header
```

扩展头机制允许在不改变协议的前提下定义额外的实体头域，但不保证这些域在接收端能够被
识别。未被识别的头域应当被接收者忽略，且必须被透明代理（transparent proxy）转发。 

###实体主体（Entity Body）

由 HTTP 请求或响应发送的实体主体（如果存在的话）的格式与编码方式应由实体的头域决定。
Entity-body= *OCTET
如 4。3 节所述，实体主体（entity-body）只有当消息主体存在时时才存在。实体主体（entitybody）从消息主体根据传输译码头域（Transfer-Encoding）解码得到，传输译码用于确保消息的安全和合适传输 

####类型（Type）

当消息包含实体主体（entity-body）时，主体的数据类型由实体头域的 Content-Type 和Content-Encoding 头域确定。这些头域定义了两层顺序的编码模型：
Entity-body：=Content-Encoding（ Content-Type（ data） ）Content-Type 指定了下层数据的媒体类型。Content-Encoding 可能被用来指定附加的应用于数据的内容编码，经常用于数据压缩的目的，内容编码是请求资源的属性。没有缺省的编码。
任一包含了实体主体的 HTTP/1.1 消息都应包括 Content-Type 头域以定义实体主体的媒体类型。如果只有媒体类型没有被 Content-Type 头域指定时，接收者可能会尝试猜测媒体类型，这通过观察实体主体的内容并且/或者通过观察 URI 指定资源的扩展名。如果媒体类型仍然不知道，接收者应该把类型看作”application/octec-stream”。 

#### 实体请求长度（Entity Length）

消息的实体主体长度指的是消息主体在被应用于传输编码（transfer-coding）之前的长度。

## 连接

### 持久连接

HTTP 持久连接有着诸多的优点：
--- 通过建立与关闭较少的 TCP 连接，不仅节省了路由器与主机（客户端，服务器，代理，网关，隧道或缓存）的 CPU 时间，还节省了主机用于 TCP 协议控制块（TCP protocol control blocks）的内存。
--- HTTP 请求和响应能在连接上进行管线请求方式。 管线请求方式能允许客户端执行多次请求而不用等待每一个请求的响应（译注：即客户端可以发送请求而不用等待以前的请求的响应到来后再发请求），并且此时只有一个 TCP 连接被使用，从而提高了效率，减少了时间。
--- 网络阻塞会被减少，这是由于减少了因 TCP 连接产生的包的数量并且也由于允许 TCP 有充分的时间去决定网络阻塞的状态。
--- 因为无须在创建 TCP 连接时的握手上耗费时间，而使后续请求的等待时间减少。
---HTTP 改进的越来越优雅，因为报告错误不需要关闭 tcp 连接的代价。将来的 HTTP 版本客户端可以乐观的尝试利用一个新特性，但是如果和老服务器通信时错误被报告，那么就要用旧的
语义进行重新尝试。
HTTP 实现应该实现持久连接 



持久连接提供了一种可以由客户端或服务器通知终止 TCP 连接的机制。利用 Connection 头域可以产生终止连接信号。一旦出现了终止连接的信号，客户端便不可再向此连接提出任何新请求 

#### 协商

除非请求里 Connection 头域中包含“close”连接标记（connection-token），HTTP/1.1 服务器总可以认为 HTTP/1.1 客户端想要维持持久连接（persistent connection）。如果服务器想在发出响应后立即关闭连接，它应当发送一个含“close”的 Connection 头域。一个 HTTP/1.1 客户端可能期望连接一直保持开着，但这必须是基于服务器响应里是否包含一个 Connection 头域并且此头域里是否包含“close”。如果客户端不想再继续维持连接来发送更多请求，那么它应发送一个值为“close”的 Connection 头域。如果客户端或服务器中的任一方在 Connection 头域里包含“close”，那么那个请求就成为这个连接的最后一个请求。

#### 管线（pilelining） 

支持持久连接（persistent conncetion）的客户端可以以管线的方式发送请求（即无须等待响应而发送多个请求）。服务器必须按接收请求的顺序发送响应。

假定以持久连接方式进行连接，并且假定在连接建立后进行管线方式请求的客户端应该准备去重新尝试连接如果首次管线请求方式尝试失败。如果客户端重新去尝试连接，那么，只有在客户端知道连接是持久连接之后，客户端才能进行管线发送请求。如果服务器在响应所有对应的请求之前关闭了连接，客户端必须准备去重新发送请求。
客户端不应该利用非等幂的方法或者非等幂的方法序列（见 9.1.2 节）进行管线方式的请求。否则一个过早的传输层连接的终止可能会导致不确定的结果。希望发送非等幂方法请求的客户端只有接收了上次它发出请求的响应后才能再次发送请求给服务器。 

**服务器过早关闭连接时客户端的行为**
如果 HTTP/1.1 客户端发送一条含有消息主体的请求消息，但不含值为“ 100-continue”的Expect 请求头域，并且如果客户端没有直接与 HTTP/1.1 源服务器相连，并且客户端在接收到服务器的状态响应之前看到了连接的关闭，那么客户端应该重试此请求。在重试时，客户端可以利用下面的算法来获得可靠的响应。
1． 向服务器发起一新连接。
2． 发送请求头域。
3． 初始化变量 R，使 R 的值为通往服务器的往返时间的估计值（比如基于建立连接的时间），
或在无法估计往返时间时设为一常数值 5 秒。
4． 计算 T=R*（2**N），N 为此前重试请求的次数。
5． 等待服务器出错响应，或是等待 T 秒（两者中时间较短的）。
6． 若没等到出错响应，T 秒后发送请求的消息主体。
7． 若客户端发现连接被提前关闭，转到第 1 步，直到请求被接受，接收到出错响应，或是用
户因不耐烦而终止了重试过程。
在任意点上，客户端如果接收到服务器的出错响应，客户端
--- 不应再继续发送请求， 并且
--- 应该关闭连接如果客户端没有完成发送请求消息。 