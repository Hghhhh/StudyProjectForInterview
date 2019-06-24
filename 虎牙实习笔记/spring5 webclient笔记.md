WebClient是从Spring WebFlux 5.0版本开始提供的一个非阻塞的基于响应式编程的进行Http请求的客户端工具。它的响应式编程的基于Reactor的。WebClient中提供了标准Http请求方式对应的get、post、put、delete等方法，可以用来发起相应的请求。
它以响应式被压流的方式执行HTTP请求；WebClient默认使用 Reactor Netty 作为HTTP连接器，当然也可以通过ClientHttpConnector修改其它的HTTP连接器。

使用 Reactor 进行反应式编程
https://www.ibm.com/developerworks/cn/java/j-cn-with-reactor-response-encode/index.html

使用 Spring 5 的 WebFlux 开发反应式 Web 应用
https://www.ibm.com/developerworks/cn/java/spring5-webflux-reactive/index.html

 使用 WebClient 访问 REST API的例子：

public class RESTClient {
    public static void main(final String[] args) {
        final User user = new User();
        user.setName("Test");
        user.setEmail("test@example.org");
        final WebClient client = WebClient.create("http://localhost:8080/user");
        final Monol<User> createdUser = client.post()
                .uri("")
                .accept(MediaType.APPLICATION_JSON)
                .body(Mono.just(user), User.class)
                .exchange()
                .flatMap(response -> response.bodyToMono(User.class));
        System.out.println(createdUser.block());
    }
}

spring 5 webclient使用指南
https://juejin.im/post/5a62f17cf265da3e51333205

restTemplate与webclient（spring boot）
https://www.cnblogs.com/NeverCtrl-C/p/9508313.html


spring webclient官网
https://docs.spring.io/spring/docs/current/spring-framework-reference/web-reactive.html#webflux-client

Spring WebClient与RestTemplate性能对比——响应式Spring的道法术器
https://blog.csdn.net/get_set/article/details/79506373

webClient-as-caller
webClient-as-caller基于WebFlux的依赖，端口号8094，不多说，直接看Controller：

    @RestController
    public class HelloController {
        private final String TARGET_HOST = "http://localhost:8092";
        private WebClient webClient;

        public HelloController() {
            this.webClient = WebClient.builder().baseUrl(TARGET_HOST).build();
        }

        @GetMapping("/hello/{latency}")
        public Mono<String> hello(@PathVariable int latency) {
            return webClient
                    .get().uri("/hello/" + latency)
                    .exchange()
                    .flatMap(clientResponse -> clientResponse.bodyToMono(String.class));
        }
