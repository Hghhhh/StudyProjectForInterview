##安全方法与等幂方法

###安全方法
GET 和 HEAD 方法只是执行没有影响的动作那就是获取资源。这些方法应该被考虑是“安全”的。可以让用户代理调用的其它方法，如：POST，PUT，DELETE，这些方法是特殊的，用户代理应该知道这些方法可能会执行不安全的动作 。

### 等幂方法

方法可以有等幂的性质因为（除了出错或终止问题）N>0 个相同请求的副作用同单个请求的副作用的效果是一样（译注：等幂就是值不变性，相同的请求得到相同的响应结果，不会出现相同的请求出现不同的响应结果）。方法 GET，HEAD，PUT，DELETE 都有这种性质。同样，方法 OPTIONS 和 TRACE 不应该有副作用，因此具有内在的等幂性。然而，有可能有多个请求的请求序列是不等幂的，即使在那样的序列中所有方法都是等幂的。（如果整个序列整体的执行的结果总是相同的，并且此结果不会因为序列的整体，部分的再次执行而改变，那么此序列是等幂的。）例如，一个序列是非等幂的如果它的结果依赖于一个值，此值在以后相同的序列里会改变。根据定义，一个序列如果没有副作用，那么此序列是等幂的（假设在资源集上没有并行的操作）。 

###OPTIONS（选项）

OPTIONS 方法表明请求想得到请求/响应链上关于此请求里的 URI（Request-URI）指定资源的通信选项信息。此方法允许客户端去判定请求资源的选项和或需求，或者服务器的能力，而不需要利用一个资源动作（译注：使用POST，PUT，DELETE 方法）或一个资源获取（译注：用 GET 方法）方法。 

此方法的响应是不能缓存的。 

如果 OPTIONS 请求消息里包括一个实体主体（当请求消息里出现 Content-Length 或者Transfer-Encoding 头域时），那么媒体类型必须通过 Content-Type 头域指明。虽然此规范没有定义如何使用此实体主体，将来的 HTTP 扩展可能会利用 OPTIONS 请求的消息主体去得到服务器得更多信息。一个服务器如果不支持 OPTION 请求的消息主体，它会遗弃此请求消息主体。
如果请求 URI 是一个星号（"*"）,，OPTIONS 请求将会应用于服务器的所有资源而不是特定资源。因为服务器的通信选项通常依赖于资源，所以”*”请求只能在“ping”或者“no-op”方法时才有用；它干不了任何事情除了允许客户端测试服务器的能力。例如：它能被用来测试代理是否遵循 HTTP/1.1。
如果请求 URI 不是一个星号（"*"）,，OPTIONS 请求只能应用于请求 URI 指定资源的选项。
200 响应应该包含任何指明选项性质的头域，这些选项性质由服务器实现并且只适合那个请求的资源（例如，Allow 头域），但也可能包一些扩展的在此规范里没有定义的头域。如果有响应
主体的话也应该包含一些通信选项的信息。这个响应主体的格式并没有在此规范里定义，但是可能会在以后的 HTTP 里定义。内容协商可能被用于选择合适的响应格式。如果没有响应主体包
含，响应就应该包含一个值为“0”的 Content-Length 头域。
Max-Forwards 请求头域可能会被用于针对请求链中特定的代理。当代理接收到一个 OPTIONS请求，且此请求的 URI 为 absoluteURI，并且此请求是可以被转发的，那么代理必须要检测Max-Forwards 头域。如果 Max-Forwards 头域的值为“0”，那么此代理不能转发此消息；而是代理应该以它自己的通信选项响应。如果 Max-Forwards 头域是比 0 大的整数值，那么代理必须递减此值当它转发此请求时。如果没有 Max-Forwards 头域出现在请求里，那么代理转发此请求时不能包含 Max-Forwards 头域。 

```HTTP
curl -X OPTIONS www.exmaple.com -i

HTTP/1.1 200 
Allow: GET,HEAD,OPTIONS
Content-Type: text/html;charset=utf-8
Content-Length: 0
Date: Mon, 25 Mar 2019 04:00:16 GMT
```



### GET

GET 方法意思是获取被请求 URI（Request-URI）指定的信息（以实体的格式）。还有get方法的uri在rfc文档中没有限制大小，可能浏览器或服务器会限制uri的大小，如果长度太长会返回414（长度太长）

get方法可以有请求体吗？
RFC文档里面没有明确说明get不能有请求体，只说了不满足方法语义的请求体会被忽略，所以get的请求体一般是被忽略的,如果你硬要用get里面的请求体，其实你是违反了RFC2616中的规定的

>Yes. In other words, any HTTP request message is allowed to contain a message body, and thus must parse messages with that in mind. Server semantics for GET, however, are restricted such that a body, if any, has no semantic meaning to the request. The requirements on parsing are separate from the requirements on method semantics.

> So, yes, you can send a body with GET, and no, it is never useful to do so.
>
> This is part of the layered design of HTTP/1.1 that will become clear again once the spec is partitioned (work in progress).
>
> ....Roy

Yes, you can send a request body with GET but it should not have any meaning. If you give it meaning by parsing it on the server and *changing your response based on its contents*, then you are ignoring this recommendation in [the HTTP/1.1 spec, section 4.3](https://tools.ietf.org/html/rfc2616#section-4.3):

> [...] if the request method does not include defined semantics for an entity-body, then the message-body [SHOULD](https://www.ietf.org/rfc/rfc2119.txt) be ignored when handling the request.

And the description of the GET method in [the HTTP/1.1 spec, section 9.3](https://tools.ietf.org/html/rfc2616#section-9.3):

> The GET method means retrieve whatever information ([...]) is identified by the Request-URI.

which states that the request-body is not part of the identification of the resource in a GET request, only the request URI.

**Update** The RFC2616 referenced as "HTTP/1.1 spec" is now obsolete. In 2014 it was replaced by RFCs 7230-7237. Quote "the message-body SHOULD be ignored when handling the request" has been deleted. It's now just "Request message framing is independent of method semantics, even if the method doesn't define any use for a message body" The 2nd quote "The GET method means retrieve whatever information ... is identified by the Request-URI" was deleted. - From a comment

### HEAD

HEAD 方法和 GET 方法一致，除了服务器不能在响应里返回消息主体。HEAD 请求响应里
HTTP 头域里的元信息（译注：元信息就是头域信息）应该和 GET 请求响应里的元信息一致。
此方法被用来获取请求实体的元信息而不需要传输实体主体（entity-body）。此方法经常被用
来测试超文本链接的有效性，可访问性，和最近的改变。.
HEAD 请求的响应是可缓存的，因为响应里的信息可能被缓存用于更新以前那个资源对应缓存
的实体.。如果出现一个新的域值指明缓存的实体和当前源服务器上的实体有所不同（可能因为
Content-Length，Content-MD5，ETag 或 Last-Modified 值的改变），那么缓存（cache）必
须认为缓存项是过时的（stale） 

###POST

POST 方法被用于请求源服务器接受请求中的实体作为请求资源的一个新的从属物。 POST 被
设计涵盖下面的功能。
--已存在的资源的注释；
--发布消息给一个布告板，新闻组，邮件列表，或者相似的文章组。
--提供一个数据块，如提交一个表单给一个数据处理过程。
--通过追加操作来扩展数据库。
POST 方法的实际功能是由服务器决定的，并且经常依赖于请求 URI（Request-URI）。POST
提交的实体是请求 URI 的从属物，就好像一个文件从属于一个目录，一篇新闻文章从属于一个
新闻组，或者一条记录从属于一个数据库。
POST 方法执行的动作可能不会对请求 URI 所指的资源起作用。在这种情况下，200（成功）或
者 204（没有内容）将是适合的响应状态，这依赖于响应是否包含一个描述结果的实体。
如果资源被源服务器创建，响应应该是 201（Created）并且包含一个实体，此实体描述了请
求的状态。并且引用了这个新资源和一个 Location 头域（见 14.30 节）。
POST 方法的响应是不可缓存的。除非响应里有合适的 Cache-Control 或者 Expires 头域。然而，
303（见其他）响应能被用户代理利用去获得可缓存的响应 



利用 HTTP 协议的服务作者不应该利用基于窗体 GET 提交敏感数据，因为这个能引起数据在
请求 URI 里被编码。许多已存在的服务，代理，和用户代理将在对第三方可见的地方记录请求
URI。服务器能利用基于窗体 POST 提交来取代基于窗体 GET 提交。 



###PUT

PUT 方法请求服务器去把请求里的实体存储在请求 URI（Request-URI）标识下。如果请求
URI（Request-URI）指定的的资源已经在源服务器上存在，那么此请求里的实体应该被当作
是源服务器关于此 URI 所指定资源实体的最新修改版本。如果请求 URI（Request-URI）指定
的资源不存在，并且此 URI 被用户代理定义为一个新资源，那么源服务器就应该根据请求里的
实体创建一个此 URI 所标识下的资源。如果一个新的资源被创建了，源服务器必须能向用户代
理（user agent） 发送 201（已创建）响应。如果已存在的资源被改变了，那么源服务器应该
发送 200（Ok）或者 204（无内容）响应。如果资源不能根据请求 URI 创建或者改变，一个合
适的错误响应应该给出以反应问题的性质。实体的接收者不能忽略任何它不理解和不能实现的
Content-*（如：Content-Range）头域，并且必须返回 501（没有被实现）响应。
如果请求穿过一个缓存（cache），并且此请求 URI（Request-URI）指示了一个或多个当前
缓存的实体，那么这些实体应该被看作是旧的。PUT 方法的响应是不可缓存的 。

**put和post的区别是？**
区别在uri，put知道uri具体代表的资源，而post不知道创建资源的具体位置。A POST method should also be used **if you do not know the specific URL** of where your newly created resource should reside. 

```http
PUT /forums/<new_thread> HTTP/2.0
Host: https://yourwebsite.com/
//简而言之，put方法用于在客户机已知的特定URL处创建或覆盖资源。

POST /forums HTTP/2.0
Host: https://yourwebsite.com/
//什么的post可能接收到下面的返回：
HTTP/2.0 201 Created
Location: /forums/<new_thread>
//The POST method should be used to create a subordinate (or child) of the resource identified by the Request-URI. In the example above, the Request-URI would be /forums and the subordinate or child would be <new_thread> as defined by the origin.
```

**什么时候用put，什么时候用post？**

put是幂等的，我们执行多次返回的结果是一样的，而post不是。

当您知道要创建或覆盖的对象的URL时，应该使用Put方法。或者，如果您只知道要在其中创建内容的类别或子部分的URL，请使用post方法。



### DELETE

DELETE 方法请求源服务器删除请求 URI 指定的资源。此方法可能会在源服务器上被人为的干
涉（或通过其他方法）。客户端不能保证此操作能被执行，即使源服务器返回成功状态码。然而，
服务器不应该指明成功除非它打算删除资源或把此资源移到一个不可访问的位置。
如果响应里包含描述成功的实体，响应应该是 200（OK）；如果 DELETE 动作还没有执行，
应该以 202（已接受）响应；如果 DELETE 请求方法已经执行但响应不包含实体，那么应该以
204（无内容）响应。
如果请求穿过缓存，并且请求 URI（Request-URI）指定了一个或多个缓存当前实体，那么这
些缓存项应该被认为是旧的。DELETE 方法的响应是不能被缓存的。 



###TRACE

TRACE方法被用于激发一个远程的，应用层的请求消息回路（译注：TRACE方法让客户端测
试到服务器的网络通路，回路的意思如发送一个请求返回一个响应，这就是一个请求响应回
路）。最后的接收者也许是源服务器，也许是接收到包含Max-Forwards头域值为0请求的代理
或网关。**TRACE请求不能包含一个实体**。
TRACE方法允许客户端去了解数据被请求链的另一端接收的情况，并且利用那些数据信息去
测试或诊断。Via头域值（见14.45）有特殊的用途，因为它可以作为请求链的跟踪信息。利用
Max-Forwards头域允许客户端限制请求链的长度，这是非常有用的，因为可以利用此去测试代
理链在无限循环里转发消息。
**如果请求是有效的，响应应该在实体主体里包含整个请求消息，并且响应应该包含一个**
**Content-Type头域值为”message/http”的头域。**此方法的响应不能被缓存 

### CONNET

在 HTTP 协议中，**CONNECT** 方法可以开启一个客户端与所请求资源之间的双向沟通的通道。它可以用来创建隧道（tunnel）。例如，**CONNECT** 可以用来访问采用了 [SSL](https://developer.mozilla.org/en-US/docs/Glossary/SSL) ([HTTPS](https://developer.mozilla.org/en-US/docs/Glossary/HTTPS))  协议的站点。客户端先和代理服务器建立连接，然后发送connet方法给代理，要求代理和目的主机建立连接，之后客户端和服务器进行通信，代理服务器就像透明一样，只是接收、转发TCP stream。

有一台可以访问外网的服务器时，可以用这个方法来翻墙

###PATCH

HTTP Put方法只允许完全替换文档。此建议添加新的HTTP方法patch，用于对资源进行部分修改。

不同于  `PUT `方法，而与 `POST `方法类似，`PATCH`  方法是非幂等的，这就意味着连续多个的相同请求会产生不同的效果。

要判断一台服务器是否支持  `PATCH `方法，那么就看它是否将其添加到了响应首部 [`Allow`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Allow) 或者 [`Access-Control-Allow-Methods`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Access-Control-Allow-Methods) （在跨域访问的场合，CORS）的方法列表中 。

