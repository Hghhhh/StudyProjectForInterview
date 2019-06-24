WebClient�Ǵ�Spring WebFlux 5.0�汾��ʼ�ṩ��һ���������Ļ�����Ӧʽ��̵Ľ���Http����Ŀͻ��˹��ߡ�������Ӧʽ��̵Ļ���Reactor�ġ�WebClient���ṩ�˱�׼Http����ʽ��Ӧ��get��post��put��delete�ȷ�������������������Ӧ������
������Ӧʽ��ѹ���ķ�ʽִ��HTTP����WebClientĬ��ʹ�� Reactor Netty ��ΪHTTP����������ȻҲ����ͨ��ClientHttpConnector�޸�������HTTP��������

ʹ�� Reactor ���з�Ӧʽ���
https://www.ibm.com/developerworks/cn/java/j-cn-with-reactor-response-encode/index.html

ʹ�� Spring 5 �� WebFlux ������Ӧʽ Web Ӧ��
https://www.ibm.com/developerworks/cn/java/spring5-webflux-reactive/index.html

 ʹ�� WebClient ���� REST API�����ӣ�

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

spring 5 webclientʹ��ָ��
https://juejin.im/post/5a62f17cf265da3e51333205

restTemplate��webclient��spring boot��
https://www.cnblogs.com/NeverCtrl-C/p/9508313.html


spring webclient����
https://docs.spring.io/spring/docs/current/spring-framework-reference/web-reactive.html#webflux-client

Spring WebClient��RestTemplate���ܶԱȡ�����ӦʽSpring�ĵ�������
https://blog.csdn.net/get_set/article/details/79506373

webClient-as-caller
webClient-as-caller����WebFlux���������˿ں�8094������˵��ֱ�ӿ�Controller��

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
