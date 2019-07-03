## WebClient

Spring WebFlux 包含了一个反应式的，非阻塞的`WebClient`来发送http请求。这个客户端具有功能齐全，流畅的API以及用于声明性组合的反应类型，请参阅 [Reactive Libraries](https://docs.spring.io/spring/docs/current/spring-framework-reference/web-reactive.html#webflux-reactive-libraries)。WebFlux客户端和服务器依赖相同的非阻塞编解码器来编码和解码请求和响应内容。

### 配置

创建一个WebClient最简单的方法是：

- `WebClient.create()`
- `WebClient.create(String baseUrl)`

上面的方法使用默认的配置Reactor Netty `HttpClient`，而且需要`io.projectreactor.netty:reactor-netty`在类路径上

你也可以使用`WebClient.builder()`来使用下面的配置：

- `uriBuilderFactory`:自定义UriBuilderFactory用作基本URL
- `defaultHeader`:每个request的默认头部
- `defaultCookie`:每个request的默认Cookies
- `defaultRequest`:定制每个请求的`Consumer`
- `filter`：过滤每个请求的过滤器
- `exchangeStrategies`:定制HTTP消息读取器和写入器
- `clientConnector`:Http client库设置

下面是一个设置HTTP 编解码器的例子:

```java
ExchangeStrategies strategies = ExchangeStrategies.builder()
            .codecs(configurer -> {
                // ...
            })
            .build();

    WebClient client = WebClient.builder()
            .exchangeStrategies(strategies)
            .build();
```

一旦build之后，生成WebClient对象就不能再改变了。我们可以通过克隆它并构建修改后的副本，而不会影响原始实例：

```java
    WebClient client1 = WebClient.builder()
            .filter(filterA).filter(filterB).build();

    WebClient client2 = client1.mutate()
            .filter(filterC).filter(filterD).build();

    // client1 has filterA, filterB

    // client2 has filterA, filterB, filterC, filterD
```

### retrieve()

方法retrieve()是获得response内容然后编码它最简单的方式。

eg:

```java
WebClient client = WebClient.create("https://example.org");

    Mono<Person> result = client.get()
            .uri("/persons/{id}", id).accept(MediaType.APPLICATION_JSON)
            .retrieve()
            .bodyToMono(Person.class);
```

你还可以获取从响应中解码的对象流，eg:

```java
Flux<Quote> result = client.get()
            .uri("/quotes").accept(MediaType.TEXT_EVENT_STREAM)
            .retrieve()
            .bodyToFlux(Quote.class);
```

默认情况下，4xx或5xx状态的responses返回的是一个WebClientResponseException或者它的子类，例如WebClientResponseException.BadRequest,WebClientResponseexception.NotFound...你也可以使用`onStatus`方法来指定异常，比如下面的例子：

```java
Mono<Person> result = client.get()
            .uri("/persons/{id}", id).accept(MediaType.APPLICATION_JSON)
            .retrieve()
            .onStatus(HttpStatus::is4xxServerError, response -> ...)
            .onStatus(HttpStatus::is5xxServerError, response -> ...)
            .bodyToMono(Person.class);
```

当使用了`onStatus`方法后，如果返回结果有内容，回调函数需要消费它。否则，返回内容会被自动丢弃来保证资源释放。

### exchange()

`exchange()`方法提供了比`retrieve()`更多的控制。下面的例子和`retrieve`等价，但也提供了对`ClientResponse`的访问：

```java
Mono<Person> result = client.get()
            .uri("/persons/{id}", id).accept(MediaType.APPLICATION_JSON)
            .exchange()
            .flatMap(response -> response.bodyToMono(Person.class));
```

在这个级别下，你甚至可以创建一个完整的ResponseEntity：

```java
 Mono<ResponseEntity<Person>> result = client.get()
            .uri("/persons/{id}", id).accept(MediaType.APPLICATION_JSON)
            .exchange()
            .flatMap(response -> response.toEntity(Person.class));
```

需要注意的是，不像`retrieve()`，使用`exchange()`的话，对4XX和5xx返回内容没有自动的错误信号。你需要自己检查返回的状态，然后决定怎么处理。

> 使用exchange（）时，必须始终使用ClientResponse的任何body或toEntity方法来确保释放资源并避免HTTP连接池的潜在问题。如果没有预期的响应内容，您可以使用bodyToMono（Void.class）。但是，如果响应确实包含内容，则连接将关闭，并且不会放回池中。

### Request Body

请求体可以从一个对象中解析得到，例如：

```java
 Mono<Person> personMono = ... ;

    Mono<Void> result = client.post()
            .uri("/persons/{id}", id)
            .contentType(MediaType.APPLICATION_JSON)
            .body(personMono, Person.class)
            .retrieve()
            .bodyToMono(Void.class);
```

你也可以从对象的流中解析得到，例如：

```jaVA
Flux<Person> personFlux = ... ;

    Mono<Void> result = client.post()
            .uri("/persons/{id}", id)
            .contentType(MediaType.APPLICATION_STREAM_JSON)
            .body(personFlux, Person.class)
            .retrieve()
            .bodyToMono(Void.class);
```

另外，也可以直接是一个对象的值，使用`syncBody`,eg:

```java
Person person = ... ;

    Mono<Void> result = client.post()
            .uri("/persons/{id}", id)
            .contentType(MediaType.APPLICATION_JSON)
            .syncBody(person)
            .retrieve()
            .bodyToMono(Void.class);
```

#### 表单数据

发送表单数据，你可以使用`MultiValueMap<String, String>`作为请求体。此时请求体的content会被`FormHttpMessageWriter`自动设置为`application/x-www-form-urlencoded`。下面是一个使用MultiValueMap的例子：

```java
  MultiValueMap<String, String> formData = ... ;

    Mono<Void> result = client.post()
            .uri("/path", id)
            .syncBody(formData)
            .retrieve()
            .bodyToMono(Void.class);
```

你也可以使用`BodyInserters`的形式来提交表单数据：

```java
    import static org.springframework.web.reactive.function.BodyInserters.*;

    Mono<Void> result = client.post()
            .uri("/path", id)
            .body(fromFormData("k1", "v1").with("k2", "v2"))
            .retrieve()
            .bodyToMono(Void.class);
```

#### 复杂数据

发送复杂数据，你需要使用`MultiValueMap<String, ?>`,值是一个Object对象，代表请求内容或者是一个`HttpEntity`对象，包含请求内容和headers信息。`MultipartBodyBuilder`提供了一个方便的方法来处理复杂请求，请看下面的例子：

```java
  MultipartBodyBuilder builder = new MultipartBodyBuilder();
    builder.part("fieldPart", "fieldValue");
    builder.part("filePart", new FileSystemResource("...logo.png"));
    builder.part("jsonPart", new Person("Jason"));

  MultiValueMap<String, HttpEntity<?>> parts = builder.build();
  Mono<Void> result = client.post()
            .uri("/path", id)
            .syncBody(parts)
            .retrieve()
            .bodyToMono(Void.class);
```

默认情况下，你不需要去指定每个内容的`Content-Type`,如果真的需要，你可以提供一个`MediaType`参数给`part()`方法

如果`MultiValueMap`包含了至少一个非String类型的值做为form-data的参数，您无需将`Content-Type`设置为`multipart / form-data`。使用`MultipartBodyBuilder`生成HttpEntity包装的时候已经考虑到了这个。

作为`MultipartBodyBuilder`的替代品，你也可以使用`BodyInserters`:

```java
import static org.springframework.web.reactive.function.BodyInserters.*;

    Mono<Void> result = client.post()
            .uri("/path", id)
            .body(fromMultipartData("fieldPart", "value").with("filePart", resource))
            .retrieve()
            .bodyToMono(Void.class);
```

### Client Filters

你可以注册一个client filter（ExchangeFilterFunction）来拦截和修改请求，eg:

```java
WebClient client = WebClient.builder()
        .filter((request, next) -> {

            ClientRequest filtered = ClientRequest.from(request)
                    .header("foo", "bar")
                    .build();

            return next.exchange(filtered);
        })
        .build();
```

filters可以用来关注横切面，例如authentication。下面的例子使用了一个basic authentication的filter：

```java
// static import of ExchangeFilterFunctions.basicAuthentication

WebClient client = WebClient.builder()
        .filter(basicAuthentication("user", "password"))
        .build();
```

 过滤器全局应用于每个请求。要更改特定请求的过滤器行为，您可以向ClientRequest添加请求属性，然后链中的所有过滤器都可以访问该属性，如以下示例所示：

```java
WebClient client = WebClient.builder()
        .filter((request, next) -> {
            Optional<Object> usr = request.attribute("myAttribute");
            // ...
        })
        .build();

client.get().uri("https://example.org/")
        .attribute("myAttribute", "...")
        .retrieve()
        .bodyToMono(Void.class);

    }
```

您还可以复制现有WebClient，插入新过滤器或删除已注册的过滤器。以下示例在索引0处插入基本身份验证筛选器：

```java
// static import of ExchangeFilterFunctions.basicAuthentication

WebClient client = webClient.mutate()
        .filters(filterList -> {
            filterList.add(0, basicAuthentication("user", "password"));
        })
        .build();
```

### 同步使用

WebClient可以通过在结尾处阻塞结果来以同步方式使用：

```java
Person person = client.get().uri("/person/{id}", i).retrieve()
    .bodyToMono(Person.class)
    .block();

List<Person> persons = client.get().uri("/persons").retrieve()
    .bodyToFlux(Person.class)
    .collectList()
    .block();
```

但是，如果需要进行多次调用，则可以更有效地避免单独阻止每个响应，而是等待组合结果：

```java
Mono<Person> personMono = client.get().uri("/person/{id}", personId)
        .retrieve().bodyToMono(Person.class);

Mono<List<Hobby>> hobbiesMono = client.get().uri("/person/{id}/hobbies", personId)
        .retrieve().bodyToFlux(Hobby.class).collectList();

Map<String, Object> data = Mono.zip(personMono, hobbiesMono, (person, hobbies) -> {
            Map<String, String> map = new LinkedHashMap<>();
            map.put("person", personName);
            map.put("hobbies", hobbies);
            return map;
        })
        .block();
```

> 

