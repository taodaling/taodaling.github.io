---
categories: framework
layout: post
---

- Table
{:toc}
# 服务治理：Spring Cloud Eureka

Spring Cloud Eureka是Spring Cloud Netflix微服务套件中的一部分，它基于Netflix Eureka做了二次封装，主要负责完成微服务架构中的服务治理功能。

`服务治理`用于实现各个微服务实例的自动化注册和发现。对于服务调用者和服务提供方，一开始可能会通过静态配置的方式在调用者的配置文件中填写服务提供方的ip地址等信息。但是服务提供方可能会发生水平扩展，即提供方的数量会增加，这样的话我们不得不修改调用方的配置文件并重启调用方。随着业务发展，越来越多的微服务出现，静态配置变得越来越难以使用。

- 服务注册：在服务治理框架中，一般会有一个注册中心，每个服务向注册中心登记自己提供的服务以及自己的地址。
- 服务发现：调用方首先向服务注册中心查询服务提供者信息，之后按照它们注册时填写的地址对服务提供者进行访问。

Eureka使用Netflix Eureka来实现服务注册与发现，它同时包含了服务端和客户端的组件，且二者均使用Java编写，所以Eureka是Java上实现的分布式服务治理框架。同时Eureka服务端提供了完备的RESTful API，因此不同平台不同语言编写的服务调用者和提供方都能使用服务端，只不过可能需要实现自己的客户端。

## 实例

### 搭建eureka服务器

我们首先搭建eureka服务器。创建eureka-server项目，在pom文件中加入下面依赖：

```xml
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
</dependency>
```

具体的依赖版本在`spring-cloud-dependencies`中指定。

接下来修改application.yml文件：

```yaml
eureka:
  instance:
    hostname: localhost
  client:
    register-with-eureka: false # 不注册自己
    fetch-registry: false # 不查询注册信息
    serviceUrl:
      defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/

server:
  port: 8760

```

之后创建入口类：

```java
@SpringBootApplication
@EnableEurekaServer
public class Application {
    public static void main(String[] args) {
        new SpringApplicationBuilder(Application.class)
                .web(WebApplicationType.SERVLET)
                .run(args);
    }
}
```

启动Application后，访问http://localhost:8760即可看到eureka的信息面板。

### 服务提供者

创建eureka-provider项目。

首先你要包含下面依赖：

```xml
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

接着修改application.yml文件：

```yaml
eureka:
  client:
    serviceUrl:
      defaultZone: http://localhost:8761/eureka/

spring:
  application:
    name: hello-service

server:
  port: 8770
```

创建入口类：

```java
@SpringBootApplication
@EnableEurekaClient
@RestController
public class Application {
    @Value("${server.port}")
    private Integer port;

    public static void main(String[] args) {
        new SpringApplicationBuilder(Application.class)
                .web(WebApplicationType.SERVLET)
                .run(args);
    }

    @RequestMapping(value = "/hello", method = RequestMethod.GET)
    public String index() {
        return "Hello World, I'm port " + port;
    }
}

```

启动入口类后，之后刷新eureka的信息面板就能看到注册的服务信息。

### 高可用注册中心

单机一旦发生故障，将不能向外提供服务。而对于注册中心，我们必须保证它的高可用性。

在Eureka的设计中，所有节点都同时是服务提供方和消费方，服务注册中心也不例外。Eureka服务器的高可用实际上是将自己作为服务向其它注册中心注册自己，这样可以形成一组的服务注册中心，实现服务清单的互相同步，达到高可用性的目的。

下面我们建立包含两个实例的Eureka服务器集群：

创建一个新的项目eureka-server-ha。

修改pom文件，增加下面依赖：

```xml
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
</dependency>
```

此外还要增加`spring-boot-maven-plugin`的依赖。

之后增加两个配置文件：

application-peer1.yml

```yaml
eureka:
  instance:
    hostname: localhost
  client:
    serviceUrl:
      defaultZone: http://localhost:8761/eureka/

server:
  port: 8760
```

application-peer2.yml

```yaml
eureka:
  instance:
    hostname: localhost
  client:
    serviceUrl:
      defaultZone: http://localhost:8760/eureka/

server:
  port: 8761
```

在新建入口类：

```java
@SpringBootApplication
@EnableEurekaServer
public class Application {
    public static void main(String[] args) {
        new SpringApplicationBuilder(Application.class)
                .web(WebApplicationType.SERVLET)
                .run(args);
    }
}
```

之后通过`mvn package`命令和`spring-boot-maven-plugin`打包获得eureka-server-ha.jar文件。

之后利用启动两次应用，两次使用的profile分别为peer1和peer2：

```sh
java -jar eureka-server-ha.jar --spring.profiles.active=peer1
java -jar eureka-server-ha.jar --spring.profiles.active=peer2
```

打开浏览器，查看http://localhost:8760和http://localhost:8761，会发现两个注册中心相互注册了自己。

如果服务提供者要向高可用Eureka服务注册自己，只需要修改

```properties
eureka.client.serviceUrl.defaultZone=http://localhost:8760/eureka/,http://localhost:8761/eureka/
```

### 服务消费者

我们创建服务消费者项目eureka-consumer。

为pom增加下面依赖：

```xml
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-netflix-ribbon</artifactId>
</dependency>
```

之后编辑application.yml文件：

```yaml
eureka:
  client:
    serviceUrl:
      defaultZone: http://localhost:8761/eureka/,http://localhost:8760/eureka/

spring:
  application:
    name: ribbon-consumer

server:
  port: 8780
```

之后创建入口类：

```java
@SpringBootApplication
@RestController
@EnableDiscoveryClient
public class Application {
    @Configuration
    public static class Config {
        @Bean
        @LoadBalanced
        RestTemplate restTemplate() {
            return new RestTemplate();
        }
    }

    @Resource
    RestTemplate restTemplate;

    @RequestMapping(value = "consume", method = RequestMethod.GET)
    public String consume() {
        return restTemplate.getForEntity("http://HELLO-SERVICE/hello", String.class).getBody();
    }

    public static void main(String[] args) {
        new SpringApplicationBuilder(Application.class)
                .web(WebApplicationType.SERVLET)
                .run(args);
    }
}
```

启动入口类，浏览地址http://localhost:8780/consume。

之后使用不同端口（通过spring参数--server.port=xx）启动多个提供者后，多次访问地址http://localhost:8780/consume，会发现端口信息会改变，这意味着负载均衡发挥了作用。

## 服务提供者

### 服务注册

服务提供者在启动的时候会通过发送REST请求将自己注册到Eureka服务器上，同时附带上服务自身的一些元数据。Eureka服务器将这些元数据存储在一个二级表中，表的一级关键字为服务名，二级关键字为实例名称。

要启用或禁用服务注册，可以修改`eureka.client.register-with-eureka`属性值，默认为true。

### 服务同步

不同的服务提供者可能会注册到不同的服务注册中心上，即他们的信息分别被多个服务中心维护。而服务注册中心之间相互注册为服务，当服务提供者发送注册请求到一个服务注册中心时，该服务注册中心会转发请求给集群中相连的其它注册中心，从而实现注册中心之间的服务同步。

### 服务续约

在服务注册完成后，服务提供者和注册中心会通过心跳检测来判断对方是否可用。同时心跳检测还可以避免服务实例被注册中心的剔除任务（evit task）从服务注册列表中移除，因此心跳还称之为续约（renew）。

续约有两个重要的可配置属性

```properties
eureka.instance.lease-renewal-interval-in-seconds=30 #心跳检测间隔
eureka.instance.lease-expiration-duration-in-seconds=90 #超时时间
```

## 服务消费者

### 获取服务

服务消费者启动后会发送一个REST请求给服务注册中心，来获取服务注册中心中注册的服务清单。为了性能考虑，客户端会缓存得到的清单，该清单30秒后过期。

```properties
eureka.client.fetch-registry=true #是否从服务器获取注册服务清单
eureka.client.registry-fetch-interval-seconds=30 #清单过期时间
```

### 服务调用

服务消费者在获取服务清单后，通过服务名可以获得具体的实例名称和该实例的元数据。之后客户端可以根据自己的需要决定调用哪个服务实例，在Ribbon中默认采用轮询策略。

对于每个服务实例，Eureka还提供了Region和Zone的概念，一个Region中可以包含多个Zone。每个服务实例都需要被注册到一个Zone中。在进行服务调用时，优先访问处于同一个Zone中的服务实例，否则才选择其它Zone中的服务。

### 服务下线

在系统的生命周期中必然会存在服务需要重启或关闭的情况。在服务实例关闭期间，我们不会希望客户端继续访问它。所以在正常关闭服务时，它会发送一个服务实例下线的REST请求给注册中心。注册中心接收到请求后，将该服务实例的状态设置为下线（DOWN），并将下线事件向其它集群中的服务中心广播。

## 服务注册中心

### 剔除任务

有些时候，服务不会正常下线，原因可能是宕机或网络故障。为了能从服务清单将这些无效服务实例剔除，Eureka会自动启动一个定时任务，默认每60秒会扫描一次服务清单，并删除其中超时未发送心跳的服务实例。

### 自我保护

在本地调试Eureka时，经常会在服务注册中心的信息面板中看到类似下面的红色警告：`EMERGENCY!...`

该警告实际触发了Eureka的自我保护机制。服务实例和注册中心之间通过心跳来相互检测，注册中心还会统计一个服务实例心跳失败的比例在15分钟内是否低于85%，如果低于，注册中心会将这些实例的注册信息保护起来，让这些实例不会过期。如果在保护期内实例出现问题，那么客户端很可能会在调用这些服务实例时出现调用失败的情况，此时客户端必须要有容错机制，比如重试或者断路器等。关闭保护机制可以帮助注册中心剔除这些无效的服务实例。

```properties
eureka.server.enable-self-preservation=false #关闭保护机制
```

# 客户端负载均衡：Spring Cloud Ribbon

Spring Cloud Ribbon是一个HTTP、TCP客户端负载均衡工具，它基于Netflix Ribbon实现。通过Spring Cloud的封装，让我们轻松将面向服务的REST模板请求转换为负载均衡的服务调用。

## 客户端负载均衡

负载均衡在系统架构中非常重要，因为负载均衡是对系统高可用、分散压力的重要手段。我们通常说的负载均衡都是指服务端负载均衡，分为硬件负载均衡和软件负载均衡。硬件负载均衡主要在服务器节点之间安装专门处理负载均衡设备，软件负载均衡则在服务器上安装一些具有均衡负载功能或模块的软件来完成请求分发工作，比如Nginx。无论是哪种负载均衡方式，其架构均下图所示：

![](https://raw.githubusercontent.com/taodaling/simple-blog/master/assets/images/2019-1-2-spring-cloud/load-balance.png)

## RestTemplate

RestTemplate会使用Ribbon的自动化配置，同时通过配置@LoadBalanced还能开启负载均衡。

## 配置

Ribbon依赖多个接口以及它们的策略实现：

- IClientConfig：Ribbon的客户端配置。
- IRule：Ribbon的负载均衡策略，默认使用ZonAvoidanceRule实现。
- IPing：Ribbon的实例检查策略，默认使用NoOpPing实现。
- ServerList：服务实例清单列表，默认使用ConfigurationBasedServerList实现。
- ServerListFilter：服务实例清单过滤策略，默认使用ZonPreferenceServerListFilter实现。
- ILoadBalancer：负载均衡策略，默认使用ZoneAwareLoadBalancer实现。

Ribbon的参数配置有两种方式，全局配置以及客户端配置。

全局配置的格式为`ribbon.<key>=<value>`，而客户端配置为`<client>.ribbon.<key>=<value>`。

要配置Ribbon使用的服务器，我们需要配置`ribbon.listOfServers`属性，比如：

```properties
ribbon.listOfServers=localhost:8001,localhost:8002
```



## 整合Eureka

在Spring Cloud应用中同时引入Spring Cloud Ribbon和Spring Cloud Eureka，将会触发Eureka对Ribbon的自动化配置。这时Ribbon的策略组件将会被替换：

- ServerList：替换为DiscoveryEnabledNIWSServerList的实例，该实例会从Eureka服务器拉取服务清单。
- IPing：替换为NIWSDiscoveryPing的实例，该实例将实例检查的任务交由注册中心。

# 服务容错保护：Spring Cloud Hystrix

在微服务结构中，服务被拆分为很多单元，单元之间通过注册中心联系起来。由于服务之间的调用是通过远程调用的方式实现的，这样一旦下游服务由于某些原因出现了阻塞和延迟，会同样影响依赖他们的上游服务。如果此时上游服务的请求继续增加，最终就会形成服任务积压以致于自身瘫痪。而这在微服务中非常常见，一个单元出现故障，依赖关系会引起故障的蔓延，最终导致系统瘫痪。为了解决这样的问题，断路器应运而生。

断路器模式源于Martin Fowler的Circuit Breaker一文。而实际中断路器作为一种电路装置，用于在电器短路时，自动切断故障电路，防止过载。而在分布式架构中，断路器的作用也很类似，当某个服务单元故障时，断路器会通过故障监控向调用方及时地返回一个超时报错，而不是长时间等待。这样线程就不会因为调用故障服务而长时间被占用，避免了故障的蔓延。

Spring Cloud Hystrix实现了断路器、线程隔离等服务保护功能。它是基于Netflix的开源框架Hystrix实现的，Hystrix拥有服务降级、服务熔断、线程和信号隔离、请求缓存、请求合并以及服务监控等功能。

## 实例

启动注册中心，并启动两个服务提供者和一个客户端实现了负载均衡的服务消费者。之后关闭一个服务提供者，不断调用服务消费者会发现有网络连接报错。

修改服务消费者代码。

首先修改pom文件，增加下面依赖：

```xml
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
</dependency>
```

之后修改我们的入口类：

```java
@SpringBootApplication
@EnableDiscoveryClient
@EnableCircuitBreaker
@RestController
public class Application {
    @Configuration
    public static class Config {
        @Bean
        @LoadBalanced
        RestTemplate restTemplate() {
            return new RestTemplate();
        }
    }

    @Resource
    RestTemplate restTemplate;

    @RequestMapping(value = "consume", method = RequestMethod.GET)
    @HystrixCommand(fallbackMethod = "helloFallback")
    public String consume() {
        return restTemplate.getForEntity("http://HELLO-SERVICE/hello", String.class).getBody();
    }

    public String helloFallback() {
        return "error";
    }

    public static void main(String[] args) {
        new SpringApplicationBuilder(Application.class)
                .web(WebApplicationType.SERVLET)
                .run(args);
    }
}
```

重复上面的实验，就会发现这次报错得到的不再是一个连接异常，而是`error`。

除了连接断开外，我们可以改造服务提供端，让它每次提供服务时睡眠0~4秒，由于Hystrix的默认超时时间为2秒，因此我们有一半的几率可以观察到服务超时的现象。

## 服务降级

上上面实例中我们已经见过了通过`@HystrixCommand`实现服务降级的方式。除了注解外，Hystrix也提供了一个抽象类HystrixCommand。通过继承它，我们能对应的实现更高灵活度的服务降级逻辑。HystrixCommand实现了命令模式，观察者模式，我们可以很轻松地在其上进行扩展。

fallback是Hystrix命令执行失败时使用的后备方法，用于实现服务的降级处理逻辑。

在HystrixCommand中的run方法中抛出异常时，除了HystrixBadRequestException外，其它异常都会被视作触发服务降级的条件。除此之外，你可以通过HystrixCommand注解的ignoreExceptions属性配置自己希望不触发服务降级的异常类型。

在通过继承HystrixCommand的方式下，我们可以通过getExcutionException()获得服务降级的原因，而在使用注解的方式下，我们可以为服务降级方法增加Throwable类型的参数，并得到异常信息。

```java
public String fallback(Throwable e){
    return e.toString();
}
```

## 请求缓存

随着系统用户的增长，在分布式环境下，通常压力来自于对远程服务的调用，这样会造成无法避免的性能损失。同时HTTP相对于其他通信协议没有任何优势，所以很容易成为性能瓶颈。

Hystrix提供了命令缓存的功能，我们可以通过开启缓存来优化性能，缓存的生命周期仅为一次HTTP请求。

要开启缓存非常简单，只需要在实现HystrixCommand时重载getCacheKey方法并返回非空值即可。

要清理缓存，只需要调用HystrixRequestCache.clear()方法即可。

我们也可以通过注解的方式开启缓存，只需要为打了`@HystrixCommand`的方法增加`@CacheResult`即可指定请求返回的结果应该被缓存，而`@CacheRemove`则用于无效化缓存，`@CacheKey`则显式地指定缓存使用的key。

## 请求合并

微服务通过远程请求来相互调用，而远程调用会带来通信消耗和占用连接数。而实际上往往通信发送的内容非常的少，时间消耗在了网络传输上。

Hystrix提供了HystrixCollapser来实现请求的合并，以减少通信消耗和线程占用数目。HystrixCollapser放置在HystrixCommand之前，作为合并请求器，将处于很短时间窗口内的对同一请求的调用打包并批量发送。当然服务提供方也需要提供相应的批量实现接口。

要开启请求合并，有两种方式，一种是继承HystrixCollapser，一种是使用`@HystrixCollapser`注解。

## Hystrix仪表盘

Hystrix Dashboard是Hystrix的仪表盘组件，用于监控Hystrix的各项指标。

要开启仪表盘功能，你需要向你的hystrix项目增加依赖包:

```xml
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

并增加下面两个Bean：

```java
@Bean
public HystrixMetricsStreamServlet hystrixMetricsStreamServlet() {
	return new HystrixMetricsStreamServlet();
}

@Bean
public ServletRegistrationBean registrationOfHystrixMetricsStreamServlet(HystrixMetricsStreamServlet hystrixMetricsStreamServlet) {
	ServletRegistrationBean result = new ServletRegistrationBean();
	result.setServlet(hystrixMetricsStreamServlet);
	result.setEnabled(true);
	result.addUrlMappings("/hystrix.stream");
	return result;
}
```

这时候你访问`http//hystrix-project-host/hystrix.stream`时页面会不断地刷新JSON数据。

之后创建一个hystrix-dashboard项目，向pom加入依赖：

```xml
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
</dependency>
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

之后创建入口类:

```java
@SpringCloudApplication
@EnableHystrixDashboard
public class Application {
    public static void main(String[] args) {
        new SpringApplicationBuilder(Application.class)
                .web(WebApplicationType.SERVLET)
                .run(args);
    }
}
```

之后访问`http://hystrix-dashboard-project-host/hystrix`，就可以看到hystrix的仪表盘界面。

![](https://raw.githubusercontent.com/taodaling/simple-blog/master/assets/images/2019-1-2-spring-cloud/hystrix-dashboard.png)

在最上面的文本框中填写`http//hystrix-project-host/hystrix.stream`即可看到该应用的hystrix图表。

![](https://raw.githubusercontent.com/taodaling/simple-blog/master/assets/images/2019-1-2-spring-cloud/hystrix-dashboard2.png)

## Turbine集群监控

