**3.0.1**

本项目通过自动配置和绑定到spring环境和其他spring编程模型习惯用法为OpenFeigh提供共针对spring boot应用的整合。
# 1. 声明式REST客户端：Feign
Feign是一个声明式web服务客户端。它使得编写web服务客户端更加容易。使用Feign创建一个接口并对其进行注解标注。它包含可插入式注解支持，包括Feign注解和JAX-RS注解。Feign同样支持可插入式编码器和解码器。Spring Cloud为其增加了对Spring MVC注解和与Spring Web默认的HttpMessageConverters的支持。在使用Feign时Spring Cloud整合了Eureka、Spring Cloud CircuitBreak以及Spring Cloud LoadBalancer提供了一个http协议的负载均衡客户端。
## 1.1. 如何包含Feign
使用org.springframework.cloud goupId和spring-cloud-stater-openfeign artifactId来在你的项目中包含Feign.参阅Spring Cloud的项目主页来获取使用当前的Spring Cloud发布版本配置你的构建系统的详情。

Spring Boot应用实例：
```java
@SpringBootApplication
@EnableFeignClients
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }

}
```
StoreClient.java(译者注：store没有特殊含义，并不是你必须使用store。这与你的具体项目有关，假如你正在开发一个天气的客户端，那你的名字就可能是WeatherClient.java)
```java
@FeignClient("stores")
public interface StoreClient {
    @RequestMapping(method = RequestMethod.GET, value = "/stores")
    List<Store> getStores();

    @RequestMapping(method = RequestMethod.GET, value = "/stores")
    Page<Store> getStores(Pageable pageable);

    @RequestMapping(method = RequestMethod.POST, value = "/stores/{storeId}", consumes = "application/json")
    Store update(@PathVariable("storeId") Long storeId, Store store);
}
```

在@Feign注解中的字符串值（上面的是“store”）代表了任意的客户端名字，他将用来创建一个Spring Cloud 负载均衡客户端。你也可以使用url属性指定一个URL(绝对路径值或者只有一个主机名)。该接口对应的Bean在应用上下文中的名字为该接口的全限定名称。你可以使用@FeignClient注解的qualifier属性来定义你自己的别名。

上面的负载均衡客户端将试图发现“store”服务的物理地址。如果你的应用是一个Eureka客户端，它将在Eureka服务的注册中心获取到。如果你不想使用Eureka，你可以在你的外部配置中使用SimpleDiscoverClient简单地配置一个服务列表。

Spring Cloud OpenFeign支持Spring Cloud LoadBalancer阻塞模型的所有功能。你一在项目文档中读取更多他们的信息。

## 1.2. 覆盖Feign的默认配置

Spring Cloud对Feign支持的一个核心盖面是命名客户端。每个Feign客户端都是集成组件的一部分，全体客户端合作工作按需连接远程客户端。应用程序开发者可以使用@FeignClient注解来给集成组件命名。Spring Cloud使用FeignClientsConfiguration按需为每个命名客户端创建一个ApplicationContext集成。其中包括一个feign.Decoder、一个feign.Encoder和一个feign.Contract。你可以使用@FeignClient注解的contextId属性覆盖集成组件的名字。

Spring Cloud允许你使用@FeignClient注解声明额外配置完全控制Feign客户端（在FeignClientsConfiguration）。
示例：
```java
@FeignClient(name = "stores", configuration = FooConfiguration.class)
public interface StoreClient {
    //..
}
```

在本示例中，客户端由存在于FeignClientsConfiguration和FooConfiguration中的组件组成（后面将会覆盖前面的FooConfiguration）。

*FooConfiguration不需要使用@Configuration注解标注。然而，如果这样做了，一定要注意在任何可能扫描到该配置的@ComponentScan中排除它。否则包含了该配置将会将feign.Decoder、feign.Encoder、feign.Contract等作为默认源，如果定义了的话。这可以通过将配置放置到一个独立、不重复的非@ComponentScan和@SpringBootApplication包中避免,或者明确的在@ComponentScan中排除。*

*使用@FeignClient注解的contextId属性除了更改ApplicationContext集成的属性之外，还会覆盖客户端的别名，该别名将会创建客户端的配置bean的名字的一部分。*

*以前，使用url属性不要求必须使用name属性。现在name属相被强制要求使用。*

name和url属相支持位置标识符。

```java
@FeignClient(name = "${feign.name}", url = "${feign.url}")
public interface StoreClient {
    //..
}
```

Spring Cloud OpenFeign默认为feign提供如下bean（beanType beanName: ClassName）：
* Decoder feignDecoder: ResponseEntityDecoder(包装这一个Spring Decoder)
* Encoder feignEncoder: SpringEncoder
* Logger feignLogger: Self4jLogger
* Contract feignContract: SpringMvcContract
* Feign.Builder feignBuilder: FeignCircuitBreaker.Builder
* Client feignClient: 如果Spring Cloud LoadBalancer在classpath中，FeignBlockingLoadBalancerClient被使用。如果没有东西在classpath中，默认的feign client被使用。

*spring-cloud-stater-openfeign支持spring-cloud-stater-loadbalancer。然而，作为一个可选的依赖，你如果想要使用它就必须确定它已将被加入到你的项目中。*

可以通过设置feign.okhttp.enabled或者feign.httpclient.enabled为true来使用OkHttpClient或者ApacheHttpClient，分别在classpath中添加他们的依赖。你可以通过提供一个org.apache.http.impl.client.CloseableHttpClient（在使用ApacheHttpClient时）或者okhttp3.OkHttpClient bean自定义被使用的客户端。

Spring Cloud OpenFeign 默认不为feign提供如下所示的bean，但是仍然从应用上下文中查找这些类型的bean来创建feign客户端：

* Logger.Level
* Retryer
* ErroeDecoder
* Request.Options
* Collection< RequestInterceptor >
* SetterFactory
* QueryMapEncoder
  一个Retryer.NEVER_RETRY bean默认被创建为Retryer类型，这将会禁止重试操作。注意这个重试操作与Feign默认的不同，它将会自动重试IOExceptions，将他们作为暂时网络连接异常对待并且将任何RetryableException从一个ErrorDecoder抛出。

创建一个对应类型的bean并将它放到一个@FeignClient配置中使得你可以覆盖对应的bean。
示例：
```java
@Configuration
public class FooConfiguration {
    @Bean
    public Contract feignContract() {
        return new feign.Contract.Default();
    }

    @Bean
    public BasicAuthRequestInterceptor basicAuthRequestInterceptor() {
        return new BasicAuthRequestInterceptor("user", "password");
    }
}
```

此例使用feign.Contract.Default代替SpringMvcContract并向RequestInterceptor集合中添加了一个RequestInterceptor。

@FeignClient也可以使用属性配置文件进行配置。

application.yml
```yaml
feign:
  client:
    config:
      feignName:
        connectTimeout: 5000
        readTimeout: 5000
        loggerLevel: full
        errorDecoder: com.example.SimpleErrorDecoder
        retryer: com.example.SimpleRetryer
       defaultQueryParameters:
          query: queryValue
        defaultRequestHeaders:
          header: headerValue
        requestInterceptors:
          - com.example.FooRequestInterceptor
          - com.example.BarRequestInterceptor
        decode404: false
        encoder: com.example.SimpleEncoder
        decoder: com.example.SimpleDecoder
        contract: com.example.SimpleContract
```

在@EnableFeignClients中使用与上面相似的语法在defaultConfiguration属性中指定默认配置。不同之处在于此配置将会运用到所有的feign客户端中。

如果你更喜欢使用属性配置文件配置所有的@FeignClient，你可以将default作为feign名作为配置属性。

你可以使用feign.client.config.feignName.defaultQueryParameters和feign.client.config.feignName.defaultRequestHeaders来指定查询参数和头部，这些参数和头部将随名称为fileName的客户端的请求一起发送。

application.yml
```yaml
feign:
  client:
    config:
      default:
        connectTimeout: 5000
        readTimeout: 5000
        loggerLevel: basic
```

如果你即创建了@Configuration bean又创建了配置属性，配置属性将优先使用。配置属性将覆盖@Configuration的值。但是如果你想要改变@Configuration的优先级，你可以将feign.client.default-to-properties的值改为false。

如果你想使用相同的名称或url创建多个feign客户端，这些客户端可能会指向相同的服务，但是却使用不同的自定义配置。我们必须使用@FeignClient的contextId属性来避免这些配置bean的名字冲突。

```java
@FeignClient(contextId = "fooClient", name = "stores", configuration = FooConfiguration.class)
public interface FooClient {
    //..
}
```
```java
@FeignClient(contextId = "barClient", name = "stores", configuration = BarConfiguration.class)
public interface BarClient {
    //..
}
```

还可以将FeignClient配置为不从父上下文继承bean。可以通过重写FeignClientConfigurer bean中的inheritParentConfiguration（）返回false来实现这一点：

```java
@Configuration
public class CustomConfiguration{

@Bean
public FeignClientConfigurer feignClientConfigurer() {
            return new FeignClientConfigurer() {

                @Override
                public boolean inheritParentConfiguration() {
                    return false;
                }
            };

        }
}
```

*默认情况下，外部客户机不编码斜杠/字符。你可以通过将feign.client.decodeSlash设置为false来改变这一行为。*

## 1.3. 超时处理
我们既可以设置默认的超时配置，也可以针对某一个命名客户端做超时配置。OpenFeign通过两个超时参数进行工作：

* contectTimeout 防止由于服务器的长处理时间而阻塞调用。
* readTime 应用于连接建立时间，在返回响应花费时间过长时被触发。

*一旦服务没有运行或者收到了一个连接失败的包。通讯将以错误消息或者回退结束。如果connectTimeout被设置的过晚，这些可能会在连接超时之前发生。执行查询和接收这样一个数据包花费的时间 导致了很大一部分时间。它可能会根据涉及DNS查找的远程主机进行更改。*

## 1.4. 手动创建Feign客户端
在一些情况下，不使用上面的方法自定义你的Feign客户端是很有必要的。在这时候，你可以使用Feign Builder API来创建Feign客户端。下面是一个使用相同的接口但使用两个独立的请求拦截器创建两个Feign客户端的示例。

```java
@Import(FeignClientsConfiguration.class)
class FooController {

    private FooClient fooClient;

    private FooClient adminClient;

        @Autowired
    public FooController(Decoder decoder, Encoder encoder, Client client, Contract contract) {
        this.fooClient = Feign.builder().client(client)
                .encoder(encoder)
                .decoder(decoder)
                .contract(contract)
                .requestInterceptor(new BasicAuthRequestInterceptor("user", "user"))
                .target(FooClient.class, "https://PROD-SVC");

        this.adminClient = Feign.builder().client(client)
                .encoder(encoder)
                .decoder(decoder)
                .contract(contract)
                .requestInterceptor(new BasicAuthRequestInterceptor("admin", "admin"))
                .target(FooClient.class, "https://PROD-SVC");
    }
}
```

*在上面的示例中，FeignClientConfiguration.class是Spring Cloud OpenFeign提供的默认配置。*

*PROD-SVC是客户端将要发送请求的服务端名称。*

*Feign Contract对象定义了哪种注解和值在接口中是合法的。自动注入的Contract bean提供了对Spring MVC注解的支持，不支持Feign原生注解。*

你也可以调用Builder的inheritParentContext(false)方法配置Feign客户端，使其不从父上下文中继承bean。

## 1.5. Feign支持Spring Cloud CircuitBreaker

如果Spring Cloud CricuitBreaker在classpath中，并且feign.cricuitbreaker.enabled=true,Feign将会使用一个断路器包裹所有方法。

通过使用@Scope("prototype")注解创建一个Feign.Builderber bean可以在所有的客户端中禁用Spring Cloud CricuitBreaker支持：

```java
@Configuration
public class FooConfiguration {
        @Bean
    @Scope("prototype")
    public Feign.Builder feignBuilder() {
        return Feign.builder();
    }
}
```

断路器的命名遵循 \<feignClientName>_\<calledMethod>模式。当使用名字为foo的Feign客户端调用接口的bar方法时，断路器名字将会是foo_bar。

## 1.6 Feign 与Spring Cloud CricutBreaker回退

Spring CricuitBreaker支持回退的概念：当链路开路或者发生错误时一个默认的代码路径将会被执行。通过设置@FeignClient的fallback属性值为实现了回退的类的类名使Feign客户端支持回退。你仍然需要将该实现声明为一个Spring bean。
```java
@FeignClient(name = "test", url = "http://localhost:${server.port}/", fallback = Fallback.class)
protected interface TestClient {

    @RequestMapping(method = RequestMethod.GET, value = "/hello")
    Hello getHello();

    @RequestMapping(method = RequestMethod.GET, value = "/hellonotfound")
    String getException();

}

@Component
static class Fallback implements TestClient {

    @Override
    public Hello getHello() {
        throw new NoFallbackAvailableException("Boom!", new RuntimeException());
    }

    @Override
    public String getException() {
        return "Fixed response";
    }

}
```

如果你想获取触发回退的原因，你可以使用@FeignClient注解的fallbackFactory属性。
```java
@FeignClient(name = "testClientWithFactory", url = "http://localhost:${server.port}/",
        fallbackFactory = TestFallbackFactory.class)
protected interface TestClientWithFactory {

    @RequestMapping(method = RequestMethod.GET, value = "/hello")
    Hello getHello();

    @RequestMapping(method = RequestMethod.GET, value = "/hellonotfound")
    String getException();

}

@Component
static class TestFallbackFactory implements FallbackFactory<FallbackWithFactory> {

    @Override
    public FallbackWithFactory create(Throwable cause) {
        return new FallbackWithFactory();
    }

}

static class FallbackWithFactory implements TestClientWithFactory {

    @Override
    public Hello getHello() {
        throw new NoFallbackAvailableException("Boom!", new RuntimeException());
    }

    @Override
    public String getException() {
        return "Fixed response";
    }

}
```

## 1.7. Feign 和 @Parimary

当Feign使用Spring Cloud CricuitBreaker回退时，在一个ApplicationContext中会有多个相同类型的bean。由于没有一个确定的bean或没有被标记为primary，@Wutowired注解将不能使用。为了能够正常工作，Spring Cloud OpenFeign将所有头的Feign标记为@Primary，因此，spring将会知道那个bean将被用来注入。在一些情况下，这可能不是那么称心如意。可以通过将@FeignClient注解的primary属性设置为false来关闭这一行为。

```java
@FeignClient(name = "hello", primary = false)
public interface HelloClient {
    // methods here
}
```

## 1.8. Feign的继承支持
Feign通过单继承支持样板api。这允许将常见操作分组到方便的基本接口中。

**UserService.java**
```java
public interface UserService {

    @RequestMapping(method = RequestMethod.GET, value ="/users/{id}")
    User getUser(@PathVariable("id") long id);
}
```
**UserResource.java**
```java
@RestController
public class UserResource implements UserService {

}
```
**UserClient.java**
```java
package project.user;

@FeignClient("users")
public interface UserClient extends UserService {

}
```

*通常不建议在服务器和客户机之间共享接口。它引入了紧耦合，并且实际上不适用于springmvc的当前形式（方法参数映射不是继承的）。*

## 1.9. Feign 请求/响应 压缩
您可以考虑为您的外部请求启用请求或响应GZIP压缩。您可以通过启用以下属性之一来执行此操作：
```java
feign.compression.request.enabled=true
feign.compression.response.enabled=true
```
Feign 请求/响应 压缩提供的设置与您可能为web服务器设置的设置类似：
```java
feign.compression.request.enabled=true
feign.compression.request.mime-types=text/xml,application/xml,application/json
feign.compression.request.min-request-size=2048
```
这些属性允许您选择压缩媒体类型和最小请求阈值长度。

对于除OkHttpClient之外的http客户端，可以启用默认的gzip解码器来解码UTF-8编码的gzip响应：
```java
feign.compression.response.enabled=true
feign.compression.response.useGzipDecoder=true
```

## 1.10. Feign日志
为每个创建的外部客户机创建一个记录器。默认情况下，记录器的名称是用于创建外部客户机的接口的完整类名。外部日志记录只响应DEBUG级别。
application.yml
```yaml
logging.level.project.user.UserClient: DEBUG
```
你可以为每个客户端配置Logger.level对象，告诉Feign要记录多少。选择包括：

* NONE，不记录（默认）
* BASIC，只记录请求方法和URL以及响应状态代码和执行时间。
* HEADERS，记录基本信息以及请求和响应头。
* FULL，记录请求和响应的头、正文和元数据。

例如，下面将设置Logger至FULL：
```java
@Configuration
public class FooConfiguration {
    @Bean
    Logger.Level feignLoggerLevel() {
        return Logger.Level.FULL;
    }
}
```

## 1.11. Feign支持@QueryMap注解
OpenFeign的@QueryMap注解提供POJO被用于GET参数映射的支持。不幸的是，默认的OpenFeign的@QueryMap注解由于缺少value属性与Spring是不相容的。

Spring Cloud OpenFeign提供了一个等价的@SpringQueryMap注解，此注解将被用于注释POJO或Map参数作为查询参数映射。

例如，Params类定义了param1和param2参数：
```java
// Params.java
public class Params {
    private String param1;
    private String param2;

    // [Getters and setters omitted for brevity]
}
```
下面的feign客户端通过使用@SpringQueryMap注解使用Params类：
```java
@FeignClient("demo")
public interface DemoTemplate {

    @GetMapping(path = "/demo")
    String demoEndpoint(@SpringQueryMap Params params);
}
```

如果你想在生成的查询参数之上获取更多控制，你可以实现自定义的QueryMapEncoder bean。

## 1.12. HATEOAS 支持
Spring 提供了一些创建符合HATEOAS原则、Spring Hateoas和Spring Data REST的REST表示的API。

如果你的项目使用了org.springframework.boot:spring-boot-starter-hateoas或者org.framework.boot:spring-boot-starter-data-rest启动器，Feign HATEOAS将被默认启动。

当HATEOAS支持被启用后，Feign客户端被允许序列化和反序列化HATEOAS表述模型：EntityModel、CollectionModel和PagedModel。
```java
@FeignClient("demo")
public interface DemoTemplate {

    @GetMapping(path = "/stores")
    CollectionModel<Store> getStores();
}
```

## 1.13. Spring @MatriVariable 支持
Spring Cloud OpenFeign提供对Spring@MatriVariable注解的支持。

如果一个方法参数被以map的形式传递，@MatriVariable路径段是通过加入map中以等号连接的键值对被创建的。

如果传递了另一个对象，则@MatrixVariable注释中提供的名称（如果已定义）或带注释的变量名称将使用=，与提供的方法参数联接。

**重点**
尽管在服务端，Spring不要求用户使用与matrix变量名同名的路径段占位符，因为这可能使得在客户端比较模糊，但Spring Cloud OpenFeign要求你在路径段占位符命名与@MatrixVariable的name属性值（如果定义了的话）或者声明的变量名相匹配。

例如：
```java
@GetMapping("/objects/links/{matrixVars}")
Map<String, List<String>> getObjects(@MatrixVariable Map<String, List<String>> matrixVars);
```
注意变量名和路径段占位符都叫matrixVars。
```java
@FeignClient("demo")
public interface DemoTemplate {

    @GetMapping(path = "/stores")
    CollectionModel<Store> getStores();
}
```

## 1.14 Feign CollectionFormat支持
我们通过@CollectionFormat注解提供对feign.CollectionFormat的支持。你可以通过向他传递一个要求的feign.CollectionFormat作为注解的值使用它注解一个Feign客户端方法。

在下面的例子中，CSV格式被用来代替默认的EXPLODED来处理方法。
```java
@FeignClient(name = "demo")
    protected interface PageableFeignClient {

        @CollectionFormat(feign.CollectionFormat.CSV)
        @GetMapping(path = "/page")
        ResponseEntity performRequest(Pageable page);

    }
```
*在发送Pageable作为查询参数时设置CSV格式，以便正确编码。*

## 1.15. Reactive支持
由于OpenFeign项目目前不支持被动客户机，比如springwebclient，springcloud也不支持对外开放。我们一旦它在核心项目中可用，就会在这里添加对它的支持。

在此之前，我们建议使用feign reactive来支持springwebclient。

### 1.15.1. 早期初始化错误
根据您使用feign户机的方式，启动应用程序时可能会看到初始化错误。要解决此问题，可以在自动连接客户端时使用ObjectProvider。
```java
@Autowired
ObjectProvider<TestFeginClient> testFeginClient;
```

## 1.16. Spring Data 支持
您可以考虑启用Jackson模块来提供支持org.springframework.data.domain.Page以及org.springframework.data.domain.Sort解码。
```java
feign.autoconfiguration.jackson.enabled=true
```
# 2. 配置属性
要查看所有与Sleuth相关的配置属性的列表，请查看[附录页](https://docs.spring.io/spring-cloud-openfeign/docs/3.0.1/reference/html/appendix.html)。