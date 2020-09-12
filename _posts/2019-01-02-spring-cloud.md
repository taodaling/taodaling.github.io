---
categories: framework
layout: post
---

- Table
{:toc}

# Eureka

Eureka是Spring Cloud中的服务注册发现中心。

## 单机部署

下面是Eureka项目的简单搭建过程。

创建一个新的项目，名字叫做eureka-sample。在maven文件中加入：

```xml
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
        </dependency>
```

之后在`application.yml`文件中写入必要的配置信息：

```yml
server:
  port: 8761
eureka:
  client:
    registerWithEureka: false #单机模式下不允许自己注册自己
    fetchRegistry: false #单机模式下不允许从注册中心拉注册信息

spring:
  application:
    name: eureka-sample
```

之后创建一个`EurekaServerApplication`文件作为Spring应用的入口。

```java
@SpringBootApplication
@EnableEurekaServer
public class EurekaServerApplication {
	public static void main(String[] args) {
		SpringApplication.run(EurekaServerApplication.class, args);
	}
}
```

启动项目后，通过浏览器访问`localhost:8761`。

# Gateway

Eureka是Spring Cloud中的网关。

## 单机部署

下面是Gateway的简单部署过程。

首先加入maven依赖：

```xml
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-gateway</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>
```

之后修改`application.yml`文件：

```yml
server:
  port: 10000

spring:
  application:
    name: gateway-sample
  cloud:
    gateway:
      routes:
      discovery:
        locator:
          enabled: true # 对于注册到注册中心的服务S，可以通过/S/X访问S的服务的路径/X
          lowerCaseServiceId: true # 这里允许我们以驼峰法来替代S

eureka:
  client:
    healthcheck:
      enabled: true
    service-url:
      defaultZone: http://peer1:8761/eureka/
```

之后创建一个入口类。

```java
@EnableDiscoveryClient
@SpringBootApplication
public class GatewayApplication {
    public static void main(String[] args) {
        SpringApplication.run(GatewayApplication.class, args);
    }
}
```

为了演示Gateway的用途，我们创建一个简单的Client项目。其maven配置为：

```xml
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>
```

之后配置文件`application.yml`内容为：

```yml
spring:
  application:
    name: gateway-client

server:
  port: 10001
  
eureka:
  client:
    healthcheck:
      enabled: true
    service-url:
      defaultZone: http://peer1:8761/eureka/
```

之后创建一个启动类：

```java
@SpringBootApplication
@EnableDiscoveryClient
public class CoreApplication {
    public static void main(String[] args) {
        SpringApplication.run(CoreApplication.class, args);
    }
}
```

之后我们还需要给gateway-client项目提供一个控制器。

```java
@RequestMapping("/hello")
@RestController
@RefreshScope
public class HelloController {
    @RequestMapping("/date")
    public String getDate() {
        return new SimpleDateFormat("yyyy-MM-dd HH:mm:ss").format(new Date());
    }
}
```

启动gateway-client后，可以通过路径`localhost:10001/hello/date`来获取当前的时间。但是由于我们使用了网关，因此可以使用`localhost:10000/gateway-client/hello/date`来实现相同功能。

# Config

Config是Spring Cloud中的分布式配置管理中心。

## 单机部署

先改maven文件。

```xml
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-config-server</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-bus-kafka</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-stream</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-stream-kafka</artifactId>
        </dependency>
```

上面引入了`kafka`，来实现配置的自动更新。同时还有`actuator`，用于提供一些和更新配置文件有关的接口。

之后我们修改配置文件`application.yml`。

```yml
eureka:
  client:
    healthcheck:
      enabled: true
    service-url:
      defaultZone: http://peer1:8761/eureka/

spring:
  application:
    name: config-sample
  profiles:
    active: native # 配置文件放在本地
  cloud:
    config:
      server:
        native:
          search-locations: /config # 本地配置文件所在的绝对路径
    stream:
      kafka:
        binder:
          zk-nodes: localhost:2181 #ZK
          brokers: localhost:9092 #Kafka
server:
  port: 8861

management:
  endpoints:
    web:
      exposure:
        include: "*" # 这里需要暴露一些端口
```

创建一个入口类。

```java
@SpringBootApplication
@EnableConfigServer
@EnableDiscoveryClient
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

之后我们增加一个客户端来演示效果。创建一个`config-client`项目。

maven文件：

```xml
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-config</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-bus-kafka</artifactId>
        </dependency>
```

之后是配置文件，由于一些配置项是在Spring容器初始化前加载的，因此我们需要在`bootstrap.yml`文件中加入：

```yml
spring:
  application:
    name: config-client
  cloud:
    config:
      profile: dev #这里我们使用dev profile
      discovery:
        enabled: true
        service-id: config-sample # 配置服务的ID
    stream:
      kafka:
        binder:
          zk-nodes: localhost:2181 #Zk地址
          brokers: localhost:9092 #Kafka地址
    bus:
      refresh:
        enabled: true # 允许刷新
eureka:
  client:
    healthcheck:
      enabled: true
    service-url:
      defaultZone: http://peer1:8761/eureka/
```

之后的`application.yml`文件如下：

```yml
server:
  port: 10001

management:
  endpoints:
    web:
      exposure:
        include: "*"
```

照例我们加入一个入口类：

```java
@SpringBootApplication
@EnableDiscoveryClient
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

以及一个Controller。

```java
@RequestMapping("/hello")
@RestController
@RefreshScope
public class HelloController {
    @Value("${greet}")
    private String greet;

    @RequestMapping("/greet")
    public String greet() {
        return greet;
    }
}
```

注意`greet`属性我们还没有在任何地方配置。下面我们配置这个在`/config`目录下创建一个新文件，叫做`config-client-dev.yml`（格式为服务ID-profile.yml）。

内容如下：

```yml
greet: "Hello world"
```

之后我们先启动`config-sample`项目，之后启动`config-client`项目。访问路径`localhost:8861/config-client/dev`可以看到整个配置文件`config-client-dev.yml`。访问路径`localhost:10001/hello/greet`会输出`Hello world`。

接下来我们在不关闭两个服务的情况下修改`config-client-dev.yml`文件。修改后访问路径`localhost:8861/config-client/dev`会直接给出最新的信息，但是`localhost:10001/hello/greet`的结果不会改变，我们必须手工让配置中心通知其它项目。具体就是通过POST方式访问路径`localhost:8861/actuator/bus-refresh`，之后重新访问`localhost:10001/hello/greet`可以看到最新的更改。

可以发现Spring在kafka中自动创建了一个名字叫做`springCloudBus`的topic，里面有一些类型为`RefreshRemoteApplicationEvent`的消息。

# Sleth

## 单机部署

### Zipkin

去官网下载zipkin。

之后我们通过下面命令来启动Zipkin。我这边使用ES作为存储底层，用KAFKA接收消息。

```sh
$ java -jar zipkin.jar --STORAGE_TYPE=elasticsearch --ES_HOSTS=localhost:9200 --ES_HTTP_LOGGING=BASIC --KAFKA_BOOTSTRAP_SERVERS=127.0.0.1:9092
```

### 项目

创建一个名字叫做`sleth-client`的项目。

下面是maven文件：

```xml
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-bus-kafka</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-zipkin</artifactId>
        </dependency>
```

下面是`application.yml`文件：


```yml
server:
  port: 10001

spring:
  application:
    name: sleth-client
  cloud:
    stream:
      kafka:
        binder:
          zk-nodes: localhost:2181
          brokers: localhost:9092
  zipkin:
    sender:
      type: kafka #使用kafka作为发送消息的方式
  sleuth:
    sampler:
      probability: 1.0 # 采用频率，1表示100%

eureka:
  client:
    healthcheck:
      enabled: true
    service-url:
      defaultZone: http://peer1:8761/eureka/
```

之后创建一个简单的WEB应用：

```java
@SpringBootApplication
@EnableDiscoveryClient
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}

```

创建一个简单的控制器：

```java
@RequestMapping("/hello")
@RestController
@RefreshScope
public class HelloController {

    @RequestMapping("/date")
    public String getDate() {
        return new SimpleDateFormat("yyyy-MM-dd HH:mm:ss").format(new Date());
    }
}
```

之后先访问路径`localhost:10001/hello/date`，再去看看zipkin的页面是否出现了变化。