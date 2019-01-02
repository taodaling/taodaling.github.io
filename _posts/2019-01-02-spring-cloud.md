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

