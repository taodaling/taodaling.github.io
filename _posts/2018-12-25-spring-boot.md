---
categories: framework
layout: post
---

- Table
{:toc}
# 快速开始

## 创建Maven项目

创建一个maven项目，并改写`pom.xml`文件

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>

	<groupId>com.example</groupId>
	<artifactId>myproject</artifactId>
	<version>0.0.1-SNAPSHOT</version>

	<!-- Inherit defaults from Spring Boot -->
	<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
		<version>2.1.1.RELEASE</version>
	</parent>

	<!-- Add typical dependencies for a web application -->
	<dependencies>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
		</dependency>
	</dependencies>

	<!-- Package as an executable jar -->
	<build>
		<plugins>
			<plugin>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-maven-plugin</artifactId>
			</plugin>
		</plugins>
	</build>

</project>
```

上面使用了`spring-boot-starter-parent`作为parent，但是由于maven是单继承的，因此你就无法再继承其他的pom，但是也可以不继承`spring-boot-starter-parent`，转而使用下面方式导入：

```xml
	<dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-starter-parent</artifactId>
                <version>2.1.1.RELEASE</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <build>
        <pluginManagement>
            <plugins>
                <plugin>
                    <groupId>org.springframework.boot</groupId>
                    <artifactId>spring-boot-maven-plugin</artifactId>
                    <version>2.1.1.RELEASE</version>
                    <executions>
                        <execution>
                            <goals>
                                <goal>repackage</goal>
                            </goals>
                        </execution>
                    </executions>
                </plugin>
            </plugins>
        </pluginManagement>
    </build>
```

## 第一个Controller

```java
import org.springframework.boot.*;
import org.springframework.boot.autoconfigure.*;
import org.springframework.web.bind.annotation.*;

@RestController
@EnableAutoConfiguration
public class Example {

	@RequestMapping("/")
	String home() {
		return "Hello World!";
	}

	public static void main(String[] args) throws Exception {
		SpringApplication.run(Example.class, args);
	}

}
```

启动后，打开浏览器，输入`localhost:8080`。

### 解释

`@RestController`是stereotype注解，它用于向用户提供代码功能的提示，并告诉Spring这个类扮演的角色。在这次情况下，我们的类是一个web控制器，所以Spring认为它能够处理到来的HTTP请求。

`@RequestMapping`注解提供路由信息，它告诉Spring所有路径为`/`的请求都会映射到`home`方法。而`@RestController`告诉Spring直接将结果字符串返回给调用者。`@RestController`和`@RequestMapping`注解都是`Spring MVC`提供的注解。

`@EnableAutoConfiguration`告诉Spring Boot去猜测我们希望怎样配置Spring，基于你追加的Jar包依赖。由于`spring-boot-starter-web`追加了`Tomcat`和`Spring MVC`，自动配置机制认为你在开发一个Web应用，并且根据此建立Spring。

自动配置被设计用来良好地处理starter，但是二者概念上并不是直接关联，除了starter外你可以自由选取要依赖的外部Jar包，Spring Boot会一如既往地尽最大努力自动配置你的应用。

`SpringApplication`启动我们的应用，启动Spring，而Spring开始自动配置Tomcat作为网站服务器。我们需要将`Example`传入作为参数，从而告诉`SpringApplication`谁是主要Spring组件。

## 可执行Jar

可执行Jar文件（有时候也称作“fat jars”）是包含你编译后的类文件以及所有依赖的Jar包的打包后文件。

Java没有提供一种标准的加载嵌套Jar文件的方式（Jar中包含其他Jar文件）。这样如果你希望分发一个自包含的应用可能会有些困难。

为了解决这个问题，许多开发者使用“uber”的Jar文件。一个uber jar将所有来自应用依赖的类文件打包到一个压缩包中，这将导致很难区分哪些类文件来自你的应用，并且如果依赖包中存中同文件名的类文件也将会被覆盖。

Spring Boot采用了一个不同的方式，允许你直接嵌套Jar文件。

要创建Jar文件，首先向pom.xml中添加下面插件：

```xml
<build>
	<plugins>
		<plugin>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-maven-plugin</artifactId>
		</plugin>
	</plugins>
</build>
```

> `spring-boot-starter-parent`中定义了上述插件的具体信息，其将该插件目`repackage`绑定到了Maven生命周期`package`上。

保存pom.xml文件后，在命令行输入`mvn package`。这样target目录下会打包出两个文件，一个以`.jar`结尾，一个以`.jar.original`结尾。`.jar.original`结尾的是原始打包出的文件，而`.jar`结尾的是Spring Boot插件打包出的文件。

# 使用Spring

## Starters

Starters是一些以方便使用为目标提供的依赖描述符，你可以将它们包含到你的应用中。你可以一站式获得所有你需要的Spring以及关联的技术，而不必通过翻查样例代码并复制粘贴批量的依赖描述符。比如你想以Spring和JPA开始你的应用，仅需要包含`spring-boot-starter-data-jpa`依赖到你的项目中。

Starter中包含了大量的你所需要的依赖，允许你快速地，一致性地构建你的项目。

一个官方的starters遵循一个类似的命名模式；`spring-boot-starter-*`，其中`*`是一个应用的别名。这个名字结构意在在你需要查找starter的时候提供帮助。如果Maven继承在大量IDEs中，允许你直接通过名字搜索依赖。

第三方的starters不应该以`spring-boot`作为开头，因为它是保留给官方Spring Boot的作品。一个第三方包通常以项目名称作为开头，比如一个叫做`thirdpartyproject`的starter项目应该命名为`thirdpartyproject-spring-boot-starter`。

| Name                                          | Description                                                  | Pom                                                          |
| --------------------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| `spring-boot-starter`                         | Core starter, including auto-configuration support, logging and YAML | [Pom](https://github.com/spring-projects/spring-boot/tree/v2.1.1.RELEASE/spring-boot-project/spring-boot-starters/spring-boot-starter/pom.xml) |
| `spring-boot-starter-activemq`                | Starter for JMS messaging using Apache ActiveMQ              | [Pom](https://github.com/spring-projects/spring-boot/tree/v2.1.1.RELEASE/spring-boot-project/spring-boot-starters/spring-boot-starter-activemq/pom.xml) |
| `spring-boot-starter-amqp`                    | Starter for using Spring AMQP and Rabbit MQ                  | [Pom](https://github.com/spring-projects/spring-boot/tree/v2.1.1.RELEASE/spring-boot-project/spring-boot-starters/spring-boot-starter-amqp/pom.xml) |
| `spring-boot-starter-aop`                     | Starter for aspect-oriented programming with Spring AOP and AspectJ | [Pom](https://github.com/spring-projects/spring-boot/tree/v2.1.1.RELEASE/spring-boot-project/spring-boot-starters/spring-boot-starter-aop/pom.xml) |
| `spring-boot-starter-artemis`                 | Starter for JMS messaging using Apache Artemis               | [Pom](https://github.com/spring-projects/spring-boot/tree/v2.1.1.RELEASE/spring-boot-project/spring-boot-starters/spring-boot-starter-artemis/pom.xml) |
| `spring-boot-starter-batch`                   | Starter for using Spring Batch                               | [Pom](https://github.com/spring-projects/spring-boot/tree/v2.1.1.RELEASE/spring-boot-project/spring-boot-starters/spring-boot-starter-batch/pom.xml) |
| `spring-boot-starter-cache`                   | Starter for using Spring Framework’s caching support         | [Pom](https://github.com/spring-projects/spring-boot/tree/v2.1.1.RELEASE/spring-boot-project/spring-boot-starters/spring-boot-starter-cache/pom.xml) |
| `spring-boot-starter-cloud-connectors`        | Starter for using Spring Cloud Connectors which simplifies connecting to services in cloud platforms like Cloud Foundry and Heroku | [Pom](https://github.com/spring-projects/spring-boot/tree/v2.1.1.RELEASE/spring-boot-project/spring-boot-starters/spring-boot-starter-cloud-connectors/pom.xml) |
| `spring-boot-starter-data-cassandra`          | Starter for using Cassandra distributed database and Spring Data Cassandra | [Pom](https://github.com/spring-projects/spring-boot/tree/v2.1.1.RELEASE/spring-boot-project/spring-boot-starters/spring-boot-starter-data-cassandra/pom.xml) |
| `spring-boot-starter-data-cassandra-reactive` | Starter for using Cassandra distributed database and Spring Data Cassandra Reactive | [Pom](https://github.com/spring-projects/spring-boot/tree/v2.1.1.RELEASE/spring-boot-project/spring-boot-starters/spring-boot-starter-data-cassandra-reactive/pom.xml) |
| `spring-boot-starter-data-couchbase`          | Starter for using Couchbase document-oriented database and Spring Data Couchbase | [Pom](https://github.com/spring-projects/spring-boot/tree/v2.1.1.RELEASE/spring-boot-project/spring-boot-starters/spring-boot-starter-data-couchbase/pom.xml) |
| `spring-boot-starter-data-couchbase-reactive` | Starter for using Couchbase document-oriented database and Spring Data Couchbase Reactive | [Pom](https://github.com/spring-projects/spring-boot/tree/v2.1.1.RELEASE/spring-boot-project/spring-boot-starters/spring-boot-starter-data-couchbase-reactive/pom.xml) |
| `spring-boot-starter-data-elasticsearch`      | Starter for using Elasticsearch search and analytics engine and Spring Data Elasticsearch | [Pom](https://github.com/spring-projects/spring-boot/tree/v2.1.1.RELEASE/spring-boot-project/spring-boot-starters/spring-boot-starter-data-elasticsearch/pom.xml) |
| `spring-boot-starter-data-jdbc`               | Starter for using Spring Data JDBC                           | [Pom](https://github.com/spring-projects/spring-boot/tree/v2.1.1.RELEASE/spring-boot-project/spring-boot-starters/spring-boot-starter-data-jdbc/pom.xml) |
| `spring-boot-starter-data-jpa`                | Starter for using Spring Data JPA with Hibernate             | [Pom](https://github.com/spring-projects/spring-boot/tree/v2.1.1.RELEASE/spring-boot-project/spring-boot-starters/spring-boot-starter-data-jpa/pom.xml) |
| `spring-boot-starter-data-ldap`               | Starter for using Spring Data LDAP                           | [Pom](https://github.com/spring-projects/spring-boot/tree/v2.1.1.RELEASE/spring-boot-project/spring-boot-starters/spring-boot-starter-data-ldap/pom.xml) |
| `spring-boot-starter-data-mongodb`            | Starter for using MongoDB document-oriented database and Spring Data MongoDB | [Pom](https://github.com/spring-projects/spring-boot/tree/v2.1.1.RELEASE/spring-boot-project/spring-boot-starters/spring-boot-starter-data-mongodb/pom.xml) |
| `spring-boot-starter-data-mongodb-reactive`   | Starter for using MongoDB document-oriented database and Spring Data MongoDB Reactive | [Pom](https://github.com/spring-projects/spring-boot/tree/v2.1.1.RELEASE/spring-boot-project/spring-boot-starters/spring-boot-starter-data-mongodb-reactive/pom.xml) |
| `spring-boot-starter-data-neo4j`              | Starter for using Neo4j graph database and Spring Data Neo4j | [Pom](https://github.com/spring-projects/spring-boot/tree/v2.1.1.RELEASE/spring-boot-project/spring-boot-starters/spring-boot-starter-data-neo4j/pom.xml) |
| `spring-boot-starter-data-redis`              | Starter for using Redis key-value data store with Spring Data Redis and the Lettuce client | [Pom](https://github.com/spring-projects/spring-boot/tree/v2.1.1.RELEASE/spring-boot-project/spring-boot-starters/spring-boot-starter-data-redis/pom.xml) |
| `spring-boot-starter-data-redis-reactive`     | Starter for using Redis key-value data store with Spring Data Redis reactive and the Lettuce client | [Pom](https://github.com/spring-projects/spring-boot/tree/v2.1.1.RELEASE/spring-boot-project/spring-boot-starters/spring-boot-starter-data-redis-reactive/pom.xml) |
| `spring-boot-starter-data-rest`               | Starter for exposing Spring Data repositories over REST using Spring Data REST | [Pom](https://github.com/spring-projects/spring-boot/tree/v2.1.1.RELEASE/spring-boot-project/spring-boot-starters/spring-boot-starter-data-rest/pom.xml) |
| `spring-boot-starter-data-solr`               | Starter for using the Apache Solr search platform with Spring Data Solr | [Pom](https://github.com/spring-projects/spring-boot/tree/v2.1.1.RELEASE/spring-boot-project/spring-boot-starters/spring-boot-starter-data-solr/pom.xml) |
| `spring-boot-starter-freemarker`              | Starter for building MVC web applications using FreeMarker views | [Pom](https://github.com/spring-projects/spring-boot/tree/v2.1.1.RELEASE/spring-boot-project/spring-boot-starters/spring-boot-starter-freemarker/pom.xml) |
| `spring-boot-starter-groovy-templates`        | Starter for building MVC web applications using Groovy Templates views | [Pom](https://github.com/spring-projects/spring-boot/tree/v2.1.1.RELEASE/spring-boot-project/spring-boot-starters/spring-boot-starter-groovy-templates/pom.xml) |
| `spring-boot-starter-hateoas`                 | Starter for building hypermedia-based RESTful web application with Spring MVC and Spring HATEOAS | [Pom](https://github.com/spring-projects/spring-boot/tree/v2.1.1.RELEASE/spring-boot-project/spring-boot-starters/spring-boot-starter-hateoas/pom.xml) |
| `spring-boot-starter-integration`             | Starter for using Spring Integration                         | [Pom](https://github.com/spring-projects/spring-boot/tree/v2.1.1.RELEASE/spring-boot-project/spring-boot-starters/spring-boot-starter-integration/pom.xml) |
| `spring-boot-starter-jdbc`                    | Starter for using JDBC with the HikariCP connection pool     | [Pom](https://github.com/spring-projects/spring-boot/tree/v2.1.1.RELEASE/spring-boot-project/spring-boot-starters/spring-boot-starter-jdbc/pom.xml) |
| `spring-boot-starter-jersey`                  | Starter for building RESTful web applications using JAX-RS and Jersey. An alternative to [`spring-boot-starter-web`](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#spring-boot-starter-web) | [Pom](https://github.com/spring-projects/spring-boot/tree/v2.1.1.RELEASE/spring-boot-project/spring-boot-starters/spring-boot-starter-jersey/pom.xml) |
| `spring-boot-starter-jooq`                    | Starter for using jOOQ to access SQL databases. An alternative to [`spring-boot-starter-data-jpa`](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#spring-boot-starter-data-jpa) or [`spring-boot-starter-jdbc`](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#spring-boot-starter-jdbc) | [Pom](https://github.com/spring-projects/spring-boot/tree/v2.1.1.RELEASE/spring-boot-project/spring-boot-starters/spring-boot-starter-jooq/pom.xml) |
| `spring-boot-starter-json`                    | Starter for reading and writing json                         | [Pom](https://github.com/spring-projects/spring-boot/tree/v2.1.1.RELEASE/spring-boot-project/spring-boot-starters/spring-boot-starter-json/pom.xml) |
| `spring-boot-starter-jta-atomikos`            | Starter for JTA transactions using Atomikos                  | [Pom](https://github.com/spring-projects/spring-boot/tree/v2.1.1.RELEASE/spring-boot-project/spring-boot-starters/spring-boot-starter-jta-atomikos/pom.xml) |
| `spring-boot-starter-jta-bitronix`            | Starter for JTA transactions using Bitronix                  | [Pom](https://github.com/spring-projects/spring-boot/tree/v2.1.1.RELEASE/spring-boot-project/spring-boot-starters/spring-boot-starter-jta-bitronix/pom.xml) |
| `spring-boot-starter-mail`                    | Starter for using Java Mail and Spring Framework’s email sending support | [Pom](https://github.com/spring-projects/spring-boot/tree/v2.1.1.RELEASE/spring-boot-project/spring-boot-starters/spring-boot-starter-mail/pom.xml) |
| `spring-boot-starter-mustache`                | Starter for building web applications using Mustache views   | [Pom](https://github.com/spring-projects/spring-boot/tree/v2.1.1.RELEASE/spring-boot-project/spring-boot-starters/spring-boot-starter-mustache/pom.xml) |
| `spring-boot-starter-oauth2-client`           | Starter for using Spring Security’s OAuth2/OpenID Connect client features | [Pom](https://github.com/spring-projects/spring-boot/tree/v2.1.1.RELEASE/spring-boot-project/spring-boot-starters/spring-boot-starter-oauth2-client/pom.xml) |
| `spring-boot-starter-oauth2-resource-server`  | Starter for using Spring Security’s OAuth2 resource server features | [Pom](https://github.com/spring-projects/spring-boot/tree/v2.1.1.RELEASE/spring-boot-project/spring-boot-starters/spring-boot-starter-oauth2-resource-server/pom.xml) |
| `spring-boot-starter-quartz`                  | Starter for using the Quartz scheduler                       | [Pom](https://github.com/spring-projects/spring-boot/tree/v2.1.1.RELEASE/spring-boot-project/spring-boot-starters/spring-boot-starter-quartz/pom.xml) |
| `spring-boot-starter-security`                | Starter for using Spring Security                            | [Pom](https://github.com/spring-projects/spring-boot/tree/v2.1.1.RELEASE/spring-boot-project/spring-boot-starters/spring-boot-starter-security/pom.xml) |
| `spring-boot-starter-test`                    | Starter for testing Spring Boot applications with libraries including JUnit, Hamcrest and Mockito | [Pom](https://github.com/spring-projects/spring-boot/tree/v2.1.1.RELEASE/spring-boot-project/spring-boot-starters/spring-boot-starter-test/pom.xml) |
| `spring-boot-starter-thymeleaf`               | Starter for building MVC web applications using Thymeleaf views | [Pom](https://github.com/spring-projects/spring-boot/tree/v2.1.1.RELEASE/spring-boot-project/spring-boot-starters/spring-boot-starter-thymeleaf/pom.xml) |
| `spring-boot-starter-validation`              | Starter for using Java Bean Validation with Hibernate Validator | [Pom](https://github.com/spring-projects/spring-boot/tree/v2.1.1.RELEASE/spring-boot-project/spring-boot-starters/spring-boot-starter-validation/pom.xml) |
| `spring-boot-starter-web`                     | Starter for building web, including RESTful, applications using Spring MVC. Uses Tomcat as the default embedded container | [Pom](https://github.com/spring-projects/spring-boot/tree/v2.1.1.RELEASE/spring-boot-project/spring-boot-starters/spring-boot-starter-web/pom.xml) |
| `spring-boot-starter-web-services`            | Starter for using Spring Web Services                        | [Pom](https://github.com/spring-projects/spring-boot/tree/v2.1.1.RELEASE/spring-boot-project/spring-boot-starters/spring-boot-starter-web-services/pom.xml) |
| `spring-boot-starter-webflux`                 | Starter for building WebFlux applications using Spring Framework’s Reactive Web support | [Pom](https://github.com/spring-projects/spring-boot/tree/v2.1.1.RELEASE/spring-boot-project/spring-boot-starters/spring-boot-starter-webflux/pom.xml) |
| `spring-boot-starter-websocket`               | Starter for building WebSocket applications using Spring Framework’s WebSocket support | [Pom](https://github.com/spring-projects/spring-boot/tree/v2.1.1.RELEASE/spring-boot-project/spring-boot-starters/spring-boot-starter-websocket/pom.xml) |
| `spring-boot-starter-actuator`                | Starter for using Spring Boot’s Actuator which provides production ready features to help you monitor and manage your application | [Pom](https://github.com/spring-projects/spring-boot/tree/v2.1.1.RELEASE/spring-boot-project/spring-boot-starters/spring-boot-starter-actuator/pom.xml) |
| spring-boot-starter-jetty                     | Starter for using Jetty as the embedded servlet container. An alternative to [`spring-boot-starter-tomcat`](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#spring-boot-starter-tomcat) | [Pom](https://github.com/spring-projects/spring-boot/tree/v2.1.1.RELEASE/spring-boot-project/spring-boot-starters/spring-boot-starter-jetty/pom.xml) |
| spring-boot-starter-log4j2                    | Starter for using Log4j2 for logging. An alternative to [`spring-boot-starter-logging`](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#spring-boot-starter-logging) | [Pom](https://github.com/spring-projects/spring-boot/tree/v2.1.1.RELEASE/spring-boot-project/spring-boot-starters/spring-boot-starter-log4j2/pom.xml) |
| spring-boot-starter-logging                   | Starter for logging using Logback. Default logging starter   | [Pom](https://github.com/spring-projects/spring-boot/tree/v2.1.1.RELEASE/spring-boot-project/spring-boot-starters/spring-boot-starter-logging/pom.xml) |
| `spring-boot-starter-reactor-netty`           | Starter for using Reactor Netty as the embedded reactive HTTP server. | [Pom](https://github.com/spring-projects/spring-boot/tree/v2.1.1.RELEASE/spring-boot-project/spring-boot-starters/spring-boot-starter-reactor-netty/pom.xml) |
| spring-boot-starter-tomcat                    | Starter for using Tomcat as the embedded servlet container. Default servlet container starter used by [`spring-boot-starter-web`](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#spring-boot-starter-web) | [Pom](https://github.com/spring-projects/spring-boot/tree/v2.1.1.RELEASE/spring-boot-project/spring-boot-starters/spring-boot-starter-tomcat/pom.xml) |
| spring-boot-starter-undertow                  | Starter for using Undertow as the embedded servlet container. An alternative to [`spring-boot-starter-tomcat`](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#spring-boot-starter-tomcat) | [Pom](https://github.com/spring-projects/spring-boot/tree/v2.1.1.RELEASE/spring-boot-project/spring-boot-starters/spring-boot-starter-undertow/pom.xml) |

## 代码结构

### 默认包

一个不含包声明的类文件，它被视作处于默认包下。不鼓励并且应该避免使用默认包，因为它会在使用`@ComponentScan`, `@EntityScan`或`@SpringBootApplication`注解时造成特定的问题。

## 主要应用类的位置

我们通常推荐你将主要应用类放置在包的根目录下，在其它类的上级目录中。`@SpringBootApplication`注解通常放置在你的主类上，并且它会隐式定义一个“搜索的基包”。使用包的根目录还将允许模块扫描将仅在你的项目中进行。

> 如果你不希望使用`@SpringBootApplication`，`@EnableAutoConfiguration`和`@ComponentScan`注解也定义了相同的行为，因此你可以使用它们作为代替。

一个典型的包目录结构：

```
com
 +- example
     +- myapplication
         +- Application.java
         |
         +- customer
         |   +- Customer.java
         |   +- CustomerController.java
         |   +- CustomerService.java
         |   +- CustomerRepository.java
         |
         +- order
             +- Order.java
             +- OrderController.java
             +- OrderService.java
             +- OrderRepository.java
```

## 配置类

Spring Boot偏好基于Java的配置。虽然它也允许你使用带有XML资源的`SpringApplication`，我们通常推荐你的主要源文件作为一个单独的`@Configuration`类，通常定义了`main`方法的类是一个主要`@Configuration`的很好的候选人。

### 导入额外的配置类

你不需要将你的配置全部放入一个类中，`@Import`注解可以用于导入额外的配置类。取而代之地，你可以使用`@ComponentScan`去自动拾取Spring组件，包括`@Configuration`类。

### 使用XML配置文件

如果你必须使用XML形式的配置文件，我们依旧推荐你以`@Configuration`类作为开始，之后你就可以使用`@ImportResource`注解去加载XML配置文件。

## 自动配置

Spring Boot试图基于你依赖的Jar文件自动配置你的Spring应用。比如如果在你的类路径中存在`HSQLDB`，并且你还尚未手动配置任何数据库连接Bean，那么Spring Boot将为你自动配置一个内存中数据库。

你需要先通过增加`@EnableAutoConfiguration`或者`@SpringBootApplication`注解到你的一个`@Configuration`才能开启自动配置功能。

> 你应该仅放置最多一个`@SpringBootApplication`或`@EnableAutoConfiguration`注解。

### 逐渐替换自动配置

自动配置是非侵入性的，任何时候，你都可以着手用自己的配置来替换自动配置的某一部分。比如如果你加入自己的`DataSource`Bean，那么默认的嵌入数据库的支持将会自动离场。

如果你希望目前哪些自动化配置被应用了，以及原因，你可以用`--debug` 启动你的应用，这样做会开启一个核心logger的debug级别，之后会将条件判断汇报到控制台。

### 禁用特定的自动配置类

如果你发现一些你不想要的自动配置类被应用了，你可以使用`@EnableAutoConfiguration`的排斥属性来禁用它们，比如：

```java
import org.springframework.boot.autoconfigure.*;
import org.springframework.boot.autoconfigure.jdbc.*;
import org.springframework.context.annotation.*;

@Configuration
@EnableAutoConfiguration(exclude={DataSourceAutoConfiguration.class})
public class MyConfiguration {
}
```

如果类并不处于类路径中，你也可以使用注解的`excludeName`属性并且指定全限定名称。最终你也可以使用`sprng.autoconfigure.exclude`属性来指定哪些类应该被禁用。

> 你既可以在注解中也可以通过属性来定义排除信息。

## Spring的Bean和依赖注入

你可以自由使用任何标准Spring框架技术来定义你的Bean和它们的注入关系。简单来说，我们经常发现使用`@ComponentScan`和`@Autowired`非常有效。

如果你按照上面建议的方式组织你的代码，你可以加入`@ComponentScan`而无需任何参数。所有你的应用组件（`@Component`,`@Service`,`@Repository`,`@Controller`等)会被自动地作为Spring中的Bean进行注册。

下面的样例展示了如何为一个`@Service`提供构造器注入：

```java
package com.example.service;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

@Service
public class DatabaseAccountService implements AccountService {

	private final RiskAssessor riskAssessor;

	@Autowired
	public DatabaseAccountService(RiskAssessor riskAssessor) {
		this.riskAssessor = riskAssessor;
	}

	// ...

}
```

如果Bean的构造器仅需一个参数，你可以省略`@Autowired`。

## 使用@SpringBootApplication注解

大量Spring Boot的开发者都希望他们的应用使用自动配置，模块扫描，并且在他们的“application class”上定义额外的配置。一个单独的`@SpringBootApplication`注解可以同时激活这三种特性：

- `@EnableAutoConfiguration`: 激活Spring Boot的自动配置机制
- `@ComponentScan`:激活"application class"所在的包内的模块扫描
- `@Configuration`:允许在上下文中注册额外的Bean或者导入额外的配置类

`@SpringBootApplication`注解等价于使用`@Configuration`,`@EnableAutoConfiguration`和`@ComponentScane`的默认参数版本，就像下面例子所展示的那样：

```java
package com.example.myapplication;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication // same as @Configuration @EnableAutoConfiguration @ComponentScan
public class Application {

	public static void main(String[] args) {
		SpringApplication.run(Application.class, args);
	}

}
```

> `@SpringBootApplication`也提供了别名属性用于自定义`@EnableAutoConfiguration`和`@ComponentScan`。

## 生产环境部署

可执行的Jar文件可以用于生产环境部署。

对于额外的生产环境需要的特性，比如健康状况，统计，和REST或JMX终端的metric数据，可以考虑使用`spring-boot-actuator`。

# Spring特性

## 实例化SpringApplication

除了使用`SpringApplication.run`启动Spring外，还可以通过创建SpringApplication对象的方式启动Spring，这样可以允许你增加一些额外的配置：

```java
public static void main(String[] args) {
	SpringApplication app = new SpringApplication(MySpringConfiguration.class);
	app.setBannerMode(Banner.Mode.OFF);
	app.run(args);
}
```

```java
new SpringApplicationBuilder()
		.sources(Parent.class)
		.child(Application.class)
		.bannerMode(Banner.Mode.OFF)
		.run(args);
```

## 应用事件和监听器

除了通常的Spring框架事件，比如`ContextRefreshEvent`，一个`SpringApplication`还会发送一些额外的应用事件。

> 有些事件会在ApplicationContext创建之前触发，所以你不能将你的监听器作为`@Bean`进行注册。又可以通过`SpringApplication.addListeners`的方式进行注册。如果你希望无论应用以何种方式创建，监听器都能自动注册，你可以通过增加一个`META-INF/spring.factories`文件到项目中，并且通过`org.springframework.context.ApplicationListener`关键字进行设置，比如说: `org.springframework.context.ApplicationListener=com.example.project.MyListener`

应用事件按照下面顺序发送：

1. 一个`ApplicationStartingEvent`，在完成了注册监听器和initializers，其它操作之前发送。
2. 一个`ApplicationEnvironmentPreparedEvent`，在确定`Environment`后，上下文创建之前发送。
3. 一个`ApplicationPreparedEvent`，bean的定义被加载后，refresh开始前被发送。
4. 一个`ApplicationStartedEvent`，上下文刷新后，所有应用和命令行runner被调用前被发送。
5. 一个`ApplicationReadyEvent`，所有应用和命令行runner被调用后，应用已经准备好接受服务请求时发送。
6. 一个`ApplicationFailedEvent`，在启动时报错后发送。

## Web环境

`SpringApplication`试图代替你创建`ApplicationContext`的正确类型，而决定`WebApplicationType`的算法相当简单：

- 如果Spring MVC出现了，那么就使用`AnnotationConfigServletWebServerApplicationContext `。
- 如果Spring MVC没有出现但是Spring WebFlux出现了，使用`AnnotationConfigReactiveWebServerApplicationContext `。
- 否则，使用`AnnotationConfigApplicationContext` 。

你也可以通过调用`setApplicationContextClass(...)`来完全掌控`ApplicationContext`类型。

## 访问应用参数

如果你需要访问通过`SpringApplication.run(...)`传入的应用参数，你可以注入一个`org.springframework.boot.ApplicationArguments`bean。

## 使用应用和命令行Runner

如果你需要在`SpringApplication`启动之后执行一些特定的代码，你可以实现`ApplicationRunner`或`CommandLineRunner`接口，两者都有相同的运行方式，并且都提供了`run`方法，他们都将在`SpringApplication.run`完成之前被调用。

`CommandLineRunner`接口允许我们访问应用字符串数组形式的参数，然而`ApplicationRunner`使用了之前讨论过的`ApplicationArguments`接口。

```java
import org.springframework.boot.*;
import org.springframework.stereotype.*;

@Component
public class MyBean implements CommandLineRunner {

	public void run(String... args) {
		// Do something...
	}

}
```

如果多个Runner需要以特定的顺序进行执行，你可以额外实现`org.springframework.core.Ordered`接口或者使用 `org.springframework.core.annotation.Order` 注解。

## 应用退出

每一个`SpringApplication`都注册了一个JVM退出的钩子，以保证`ApplicationContext`能在退出时优雅地关闭。所有标准Spring回调（比如`DisposableBean`接口或`@PreDestroy`注解）都能生效。

额外的，beans可以实现 `org.springframework.boot.ExitCodeGenerator` 接口，如果他们希望返回一个特定的exit code当`SpringApplication.exit()`被调用。该exit code可以被传递给`System.exit()`，作为JVM的退出码，如下面例子所示：

```java
@SpringBootApplication
public class ExitCodeApplication {

	@Bean
	public ExitCodeGenerator exitCodeGenerator() {
		return () -> 42;
	}

	public static void main(String[] args) {
		System.exit(SpringApplication
				.exit(SpringApplication.run(ExitCodeApplication.class, args)));
	}
}
```

## 外部配置

Spring Boot允许你将配置放在外部，这样你就可以在不同环境下使用相同的代码。你可以使用配置文件，YAML文件，环境变量和命令行参数来设置外部参数。属性可以通过`@Value`注解直接注入到你的Bean中，或者通过Spring的`Environment`抽象或者通过`ConfigurationProperties`。

Spring Boot使用了一个非常特别的`PropertySource`顺序来决定不同配置文件中的属性覆盖关系。属性将按照下面的顺序来读取：

1. Devtools配置的全局属性（~/.spring-boot-devtools.properties）
2. 在你的测试类上的`@TestPropertySource`注解
3. 在你的测试类上的`@SpringBootTest#properties`注解属性。
4. 命令行参数
5. 来自`SPRING_APPLICATION_JSON`(嵌套在环境变量或是系统属性中的内联JSON)
6. `ServletConfig`初始化参数
7. `ServletContext`初始化参数
8. 来自 `java:comp/env`的JNDI属性
9. Java系统属性（`System.getProperties()`）
10. 操作系统环境变量
11. 一个仅拥有形如`random.*`属性的`RandomValuePropertySource `。
12. Jar包外Profile指定的应用配置文件（`application-{profile}.properties`和YAML变量）
13. Jar包内的Profile指定的应用配置文件（`application-{profile}.properties`和YAML变量）
14. Jar包外应用配置文件（`application.propeties`和YAML变量）
15. Jar包内应用配置文件（`application.propeties`和YAML变量）
16. 注解在`@Configuration`类上的`@PropetySource`
17. 默认属性（由`SpringApplication.setDefaultProperties`指定)

提供一个完整的例子，假设你开发了一个`@Component`使用了`name`属性：

```java
import org.springframework.stereotype.*
import org.springframework.beans.factory.annotation.*

@Component
public class MyBean {

    @Value("${name}")
    private String name;

    // ...

}
```

在你的Jar包内你可以提供`application.properties`，并为`name`属性提供一个默认值。当在新环境运行时，你可以在Jar包外提供一个`application.properties`，并在其中覆盖`name`属性的定义。对于一次性的测试，你可以用额外的命令行参数指定`name`属性值（比如`java -jar app.jar --name="Spring"`)。

> `SPRING_APPLICATION_JSON`属性可以作为环境变量的一部分在命令行中提供，比如：
>
> `SPRING_APPLICATION_JSON='{"foo":{"bar":"spam"}}' java -jar myapp.jar`
>
> 在上面这个例子中，你最终会从Spring`Environment`中得到`foo.bar=spam`。你也可以在系统变量中提供JSON作为`spring.application.json`：
>
> `java -Dspring.application.json='{"foo":{"bar":"spam"}}' -jar myapp.jar`
>
> 或者命令行参数：
>
> `java -jar myapp.jar --spring.applicaiton.json='{"foo":"bar"}'`
>
> 或者作为JNDI变量`java:comp/env/spring.application.json`。

### 配置随机值

`RandomValuePropertySource`在注入随机值时相当有用（比如说在测试用例中）。它可以产生整数，长整形，uuid，或者字符串，就像下面样例中所显示的：

```properties
my.secret=${random.value}
my.number=${random.int}
my.bignumber=${random.long}
my.uuid=${random.uuid}
my.number.less.than.ten=${random.int(10)}
my.number.in.range=${random.int[1024,65536]}
```

`random.int*`的语法为`OPEN value (,max) CLOSE`，其中`OPEN,CLOSE`可以是任意字符，`value,max`是整数，如果提供了可选的`max`，那么`value`表示最小值（包含），`max`是最大值（不包含）。

### 命令行属性

默认情况下，`SpringApplication`会将所有命令行可选参数（即以`--`开头的参数，比如`--server.port=9000`）转换为属性，并且加入到Spring的`Environment`中。

如果你不希望将命令行参数加入到`Environment`中，你也完全可以通过使用`SpringApplication.setAddCommandLineProperties(false)`来禁用他们。

### 应用属性文件

`SpringApplication`从`application.properties`文件中加载属性，并且加入到Spring的`Environment`中。加载位置如下：

1. 当前目录下的`/config`子目录
2. 当前目录
3. 类路径中的`/config`包
4. 类路径的根目录

序号越小的位置中的配置文件中的属性优先级越高。你也可以使用YAML文件替换properties文件。

通过`spring.config.name`环境属性，你可以指定加载的配置文件名。使用`spring.config.location`环境属性，你可以直接指定文件的显式路径（以逗号分隔的路径列表）。

```sh
java -jar myproject.jar --spring.config.name=myproject
```

```sh
java -jar myproject.jar --spring.config.location=classpath:/default.properties,classpath:/override.properties
```

> `spring.config.name`和`spring.config.location`必须作为一个环境属性定义（作为操作系统的环境变量，或是系统属性，或是命令行参数）。

如果`spring.config.location`指定的是目录，那么应该以`/`作为结尾，这时候会使用`spring.config.name`作为实际的文件名。

### Profile指定的属性

除了`application.properties`文件外，profile指定的配置文件也能通过下面的约定进行定义：`application-{profile}.properties`.`Environment`有一系列的默认profile（默认情况下，profile为`default`）。

Profile指定的配置文件的加载位置与`application.properties`是相同的。

### 属性中的占位符

在配置文件中的值，会通过已存在于`Environment`中的属性进行过滤，所以你可以利用占位符引用之前的值：

```properties
app.name=MyApp
app.description=${app.name} is a Spring Boot application
```

## 开发Web应用

Spring Boot非常适合Web应用开发。你可以使用自包含的HTTP服务器，利用内嵌的Tomcat，Jetty，Undertow，或者Netty。大多数Web应用使用`spring-boot-starter-web`模块来构建并快速运行。你也可以选择使用`spring-boot-starter-webflux`模块构建响应式Web应用。

### Spring Web MVC 框架

Spring Web MVC框架（通常简称为Spring MVC）是一个富“model view controller”网站框架。Spring MVC允许你创建特殊的`@Controller`或`@RestController`Bean来处理到来的HTTP请求。在你控制器中的方法通过使用`@RequestMapping`注解来映射到HTTP。

下面的代码展示了一个提供JSON结果的典型`@RestController`:

```java
@RestController
@RequestMapping(value="/users")
public class MyRestController {

	@RequestMapping(value="/{user}", method=RequestMethod.GET)
	public User getUser(@PathVariable Long user) {
		// ...
	}

	@RequestMapping(value="/{user}/customers", method=RequestMethod.GET)
	List<Customer> getUserCustomers(@PathVariable Long user) {
		// ...
	}

	@RequestMapping(value="/{user}", method=RequestMethod.DELETE)
	public User deleteUser(@PathVariable Long user) {
		// ...
	}

}
```

Spring MVC是Spring框架的核心的一部分，可以在[reference documentation](https://docs.spring.io/spring/docs/5.1.3.RELEASE/spring-framework-reference/web.html#mvc)中获得更细节的信息。这里有数个覆盖Spring MVC的向导：[spring.io/guides](https://spring.io/guides)。

### Spring MVC自动配置

Spring Boot为Spring MVC提供自动配置，自动配置追加下面的特性：

- 包含`ContentNegotiatingViewResolver`和`BeanNameViewResolver`的Bean
- 支持响应静态资源，包括对WebJars的支持
- 自动注册`Converter`，`GenericConverter`和`Formatter`的Bean
- 支持`HttpMessageConverters`
- 自动注册`MessageCodesResolver`
- 支持静态`index.html`
- 支持自定义`Favicon`
- 自动使用`ConfigurableWebBindingInitializer`Bean

如果你想要保留Spring Boot MVC特性，并且你想增加额外的MVC配置（拦截器，格式化，视图控制器以及其它特性），你可以增加自己的`WebMvcConfigurer`类型的`@Configuration`类，但是不提供`@EnableWebMvc`。

如果你希望完全掌控Spring MVC，你可以增加你自己的`@Configuration`注解，并且提供`@EnableWebMvc`。

### HttpMessageConverters

Spring MVC使用`HttpMessageConverter`接口来转换HTTP请求和响应。比如对象可以自动转换为JSON或XML。默认，字符串以`UTF-8`进行编码。

如果你需要自定义Converter，可以定义自己的`HttpMessageConverters`Bean：

```java
import org.springframework.boot.autoconfigure.web.HttpMessageConverters;
import org.springframework.context.annotation.*;
import org.springframework.http.converter.*;

@Configuration
public class MyConfiguration {

	@Bean
	public HttpMessageConverters customConverters() {
		HttpMessageConverter<?> additional = ...
		HttpMessageConverter<?> another = ...
		return new HttpMessageConverters(additional, another);
	}

}
```

### 自定义JSON序列化和反序列化

如果你是用Jackson来序列化和反序列化JSON数据，你可能想要撰写自己的`JsonSerializer`和`JsonDeserializer`。自定义序列化通过通过Jackson利用模块来进行注册，但是Spring Boot提供了另外一个作为替代的`@JsonComponent`注解，你可以利用它很容易地将序列化注册为Spring Bean。

你可以在`JsonSerializer`或`JsonDeserializer`实现上直接使用`@JsonComponent`注解。你也可以将`@JsonComponent`使用在包含实现了序列化和反序列化子类的外部类上，比如下面：

```java
import java.io.*;
import com.fasterxml.jackson.core.*;
import com.fasterxml.jackson.databind.*;
import org.springframework.boot.jackson.*;

@JsonComponent
public class Example {

	public static class Serializer extends JsonSerializer<SomeObject> {
		// ...
	}
平日嗯
	public static class Deserializer extends JsonDeserializer<SomeObject> {
		// ...
	}

}
```

在`ApplicationContext`中的所有`JsonComponent`都自动向Jackson进行注册。因为`@Component`是`@JsonComponent`的元注解，因此通用的模块扫描规则也会被应用。

### 静态内容

默认情况下，Spring Boot提供来自类路径上`/static`(或`/public`或`/resources`或`/META-INF/resources`)的静态内容，或者来自`ServletContext`根目录上的内容。

默认情况下，资源都映射到`/**`，但是你可以通过`spring.mvc.static-path-pattern`修改它。比如，重定向所有的资源到`/resources/**`：

```properties
spring.mvc.static-path-pattern=/resources/**
```

你也可以通过`spring.resources.static-locations`自定义本地静态资源位置。

### 模板引擎

像REST风格的web服务一样，你也可以使用Spring MVC来提供动态HTML内容服务。Spring MVC提供了多种模板技术，包括Thymeleaf,FreeMarker和JSP，以及还有其它一系列的模板引擎。

Spring Boot为下面的模板引擎提供了自动化配置支持：

- FreeMarker
- Groovy
- Thymeleaf
- Mustache

如果你是用了上述的模板引擎以及默认配置，你的模板将自动从`/src/main/resources/templates`中读取。

### 错误处理

默认情况下，Spring Boot提供了一个`/error`映射处理所有的异常，并且它也作为全局错误页面注册在servlet容器。对于机器客户带你，它会产生包含了错误、HTTP状态码和异常信息的JSON响应。对于浏览器客户端，将会以HTML格式展示类似的内容。要替换这些默认行为，你需要实现`ErrorController`并且注册一个该类型的bean，或者增加一个`ErrorAtributes`类型的bean以使用现有的机制并替换部分内容。

> `BasicErrorController`可以作为自定义`ErrorController`的基类，这在你仅需要为新的内容类型增加处理器的时候特别有用。要这样做的话，你仅需继承`BasicErrorController`，并为其增加一个公共方法，并打上`@RequestMapping`注解，为注解提供一个`produces`属性，并且为你的类型创建一个bean。

你也可以定义一个打上`@ControllerAdvide`注解的类，来自定义某个控制器或是特定异常类型返回的JSON内容。

```java
@ControllerAdvice(basePackageClasses = AcmeController.class)
public class AcmeControllerAdvice extends ResponseEntityExceptionHandler {

	@ExceptionHandler(YourException.class)
	@ResponseBody
	ResponseEntity<?> handleControllerException(HttpServletRequest request, Throwable ex) {
		HttpStatus status = getStatus(request);
		return new ResponseEntity<>(new CustomErrorType(status.value(), ex.getMessage()), status);
	}

	private HttpStatus getStatus(HttpServletRequest request) {
		Integer statusCode = (Integer) request.getAttribute("javax.servlet.error.status_code");
		if (statusCode == null) {
			return HttpStatus.INTERNAL_SERVER_ERROR;
		}
		return HttpStatus.valueOf(statusCode);
	}

}
```

### 自定义错误页面

如果你希望为给定的状态码展示自定义的错误页面，你可以向`/error`文件夹增加一个文件，错误页面可以是静态的HTML文件或者通过模板引擎构建，文件的名字应该为确定的状态码或是一系列掩码。

比如，要匹配404错误码到静态HTML文件，你的文件夹结构应该如下：

```
src/
 +- main/
     +- java/
     |   + <source code>
     +- resources/
         +- public/
             +- error/
             |   +- 404.html
             +- <other public assets>
```

如果要提供更复杂的匹配，你可以提供一个实现了`ErrorViewResolver`的Bean，像下面的例子中所展示的：

```java
public class MyErrorViewResolver implements ErrorViewResolver {

	@Override
	public ModelAndView resolveErrorView(HttpServletRequest request,
			HttpStatus status, Map<String, Object> model) {
		// Use the request or status to optionally return a ModelAndView
		return ...
	}

}
```

## SQL数据库

Spring框架为SQL数据库提供了扩展支持，从直接利用`JdbcTemplate`的JDBC访问到完整的`Object Relational Mapping`技术（比如说Hibernate）。

### 配置DataSource

Java的`javax.sql.DataSource`接口提供了一个与数据库连接协作的标准接口。传统情况下一个DataSource使用一个`URL`以及一些认证信息来建立数据库连接。

### 嵌入式数据库支持

通常情况下基于内存数据库开发一个应用是很较方便的，显然，内存数据库不能提供持久化存储。你需要在应用启动的时候装填数据，并且在应用停止的时候丢弃数据。

Spring Boot自动配置像H2，HSQL和Derby等嵌入式数据库。你不需要提供连接URL，你只需要包含你希望使用的嵌入式数据库的依赖。

> 如果你在测试时使用这个特性，你会发现整个测试流程仅使用了一个相同的数据库，而不管有多多少个应用上下文。如果你希望每个上下文都有一个独立的嵌入式数据库，你应该设置下面属性：`spring.datasource.generate-unique-name`为`true`。

一个典型的POM依赖如下：

```xml
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
<dependency>
	<groupId>org.hsqldb</groupId>
	<artifactId>hsqldb</artifactId>
	<scope>runtime</scope>
</dependency>
```

> 如果你要求自动化配置嵌入式数据库，你需要依赖`spring-jdbc`，在这个例子中，它是通过`spring-boot-starter-data-jpa`传递引用到的。

> 如果你必须要为嵌入式数据库配置URL，那必须要确保禁用数据库自动关闭。如果你使用H2，你应该使用`DB_CLOSE_ON_EXIT=FALSE`，如果使用HSQLDB，你应该确保不使用`shutdown=true`。禁用数据库的自动关闭，允许Spring Boot控制数据库关闭的时机。

### 连接生产数据库

生产数据库连接也可以通过使用池化`DataSource`被自动配置。Spring Boot使用下面的算法来选择数据库连接的实现：

1. 我们由于性能和并发偏好`HikariCP`。如果`HikariCP`可用，我们总是选择它。
2. 否则，如果Tomcat池化`DataSource`可用，我们使用它。
3. 否则，如果`Common DBCP2`可用，我们使用它。

如果你使用了 `spring-boot-starter-jdbc`或`spring-boot-starter-data-jpa` ，那么你会自动获得对`HikariCP`的依赖。

> 你可以完全绕过上述的算法，通过设置`spring.datasource.type`指定连接池的类型。这在当你使用Tomcat容器的时候特别有用，因为`tomcat-jdbc`总时默认被提供。

> 额外的数据库连接总是可以手动配置，如果你定义了自己的`DataSource`的bean，那么就不会触发对数据库连接的自动配置。

DataSource是通过外部配置属性`spring.datasource.*`配置的。比如说你可能会在配置文件中声明下面的段落：

```properties
spring.datasource.url=jdbc:mysql://localhost/test
spring.datasource.username=dbuser
spring.datasource.password=dbpass
spring.datasource.driver-class-name=com.mysql.jdbc.Driver
```

> 你至少也应该要配置URL属性，否则，Spring Boot会尝试自动配置并使用嵌入式数据库。

> 你通常不需要指定`driver-class-name`，由于Spring Boot能从URL中推导出大部分数据库类型。

## 使用JdbcTemplate

Spring的`JdbcTemplate`和`NamedParameterJdbcTemplate`类型是自动配置的，你可以直接将它们注入到你自己的Bean中，就如下面代码所示：

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.stereotype.Component;

@Component
public class MyBean {

	private final JdbcTemplate jdbcTemplate;

	@Autowired
	public MyBean(JdbcTemplate jdbcTemplate) {
		this.jdbcTemplate = jdbcTemplate;
	}

	// ...

}
```

你也可以通过使用`spring.jdbc.template.*`属性来自定义一些Template的属性，就像下面所示:

```properties
spring.jdbc.template.max-rows=500
```

> `NamedParamterJdbcTemplate`重用了`JdbcTemplate`实例，如果定义了多个`JdbcTemplate`，并且没有确定主候选人，那么就不会自动配置`NamedParameterJdbcTemplate`

### JPA和Spring Data JPA

Java持久化API是允许你将对象映射到关系型数据库的标准技术。`spring-boot-starter-data-jpa`的POM中提供了快速开始的内容。它主要提供了下面的关键依赖：

- Hibernate：最流行的JPA实现
- Spring Data JPA：使得实现基于JPA的仓库变得容易
- Spring ORMs：为Spring框架提供核心ORM支持

#### 实体类

传统地，JPA实体类被指定在`persistence.xml`文件中。使用了Spring Boot之后，这个文件就不是必须的了，取而代之的则是实体类扫描。默认情况下，所有你的主要配置类（打了`@EnableAutoConfiguration`或`@SpringBootApplication`的类）下的包都会被扫描。

任何被标记了`@Entity`,`@Embeddable`或`@MappedSuperClass`都会被视作实体类。一个典型的实体类如下：

```java
package com.example.myapp.domain;

import java.io.Serializable;
import javax.persistence.*;

@Entity
public class City implements Serializable {

	@Id
	@GeneratedValue
	private Long id;

	@Column(nullable = false)
	private String name;

	@Column(nullable = false)
	private String state;

	// ... additional members, often include @OneToMany mappings

	protected City() {
		// no-args constructor required by JPA spec
		// this one is protected since it shouldn't be used directly
	}

	public City(String name, String state) {
		this.name = name;
		this.state = state;
	}

	public String getName() {
		return this.name;
	}

	public String getState() {
		return this.state;
	}

	// ... etc

}
```

> 你可以通过设置`@EntityScan`注解自定义实体类扫描路径。

#### Spring Data JPA仓库

Spring Data JPA仓库是你定义的用于访问数据的接口。JPA查询是从你的方法名称中自动给创建的。比如，一个`CirtyRepository`接口可以定义一个`findAllByState(String date)`方法来查找同一个州中的所有城市。

对于更加复杂的查找，你可以用Spring Data的`Query`注解注解在你的方法上。

Spring Data仓库通常继承自`Repository`或`CrudRepository`接口。如果你是用了自动配置，所有定义在主配置类同级或下级包中的仓库都会被扫描。

下面例子展示了一个典型的Spring Data仓库接口的定义：

```java
package com.example.myapp.domain;

import org.springframework.data.domain.*;
import org.springframework.data.repository.*;

public interface CityRepository extends Repository<City, Long> {

	Page<City> findAll(Pageable pageable);

	City findByNameAndStateAllIgnoringCase(String name, String state);

}
```

Spring Data JPA仓库支持三种不同的启动模式：default，deferred，和lazy。要以deferred或lazy模式启动，设置`spring.data.jpa.repositories.bootstrap-mode`为`deferred`或`lazy`。如果以deferred或lazy模式启动，自动配置的`EntityManagerFactoryBuilder`会使用上下文的异步任务执行器，作为启动执行器。

#### 创建和删除JPA数据库

默认，JPA的数据库在你使用嵌入式数据库的时候会被自动创建。你可以通过`spring.jpa.*`显式配置JPA。比如，要创建表或删除表，你可以像`application.properties`中加入下面行:

```properties
spring.jpa.hibernate.ddl-auto=create-drop
```

### Spring Data JDBC

Spring Data包含对JDBC的仓库支持，并且会为`CrudRepository`的方法自动生成SQL。对于更加高级的查询，提供了`@Query`注解。

当必要的依赖出现在类路径中，Spring Boot会自动配置Spring Data的JDBC仓库。这些依赖都可以通过加入`spring-boot-starter-data-jdbc`一次性加入到你的项目中。如果必要，你可以通过加入`@EnableJdbcRepositories`注解或`JdbcConfiguration`子类来掌控Spring Data JDBC的配置。

## 日志

Spring Boot使用[Commons Logging](https://commons.apache.org/logging)作为内部日志，但是将底层的日志实现开放。为Java Util Logging, Log4J2和Logback提供了默认配置。在每一种情况，日志被预先配置为使用控制台输出，而文件输出则是可选的。

默认情况下，如果你使用了Starter，Logback将作为日志实现被使用。

## 测试

Spring Boot提供数个工具和注解帮助你完成测试。测试通过两个模块提供：`spring-boot-test`和`spring-boot-test-autoconfigure`。

大部分开发者使用`spring-boot-starter-test`Starter，其中回到如Spring Boot测试模块，JUnit，AssertJ，Hamcrest，和数个其它有用的库。

### 测试范围依赖

`spring-boot-starter-test`Starter（处于test范围）包含下面提供的包:

- JUnit：Java应用单元测试的实际标准
- Spring Test & Spring Boot Test：为Spring Boot应用提供工具和集成测试支持
- AssertJ：一个流畅的断言库
- Hamcrest：匹配器对象的库
- Mockito：一个Java模拟框架
- JSONassert：对JSON的断言库
- JsonPath：对JSON的XPath库

我们通常会发现这些公共库在写测试用例的时候非常有用。如果这些库不匹配你的需求，你可以追加自己的依赖。

### 测试Spring应用

依赖注入的主要好处就是它使得你的代码更加容易进行单元测试。你可以在不涉及Spring的情况下使用`new`操作符创建对象。你也可以使用模拟的对象而非真实的依赖。

一个Spring Boot应用是一个Spring的ApplicationContext。

> 外部属性，日志以及其它Spring Boot特性会在上下文安装当且仅当你使用`SpringApplication`创建上下文。

Spring Boot提供了`@SpringBootTest`注解，当你需要Spring Boot的特性时作为标准的spring-test的`@ContextConfiguration`注解的替代物。这个注解通过SpringApplication创建测试时需要的ApplicationContext。除了`@SpringBootTest`外还提供了数个其它的注解。

> 如果你是用JUnit4，不要忘了要加上`@RunWith(SpringRunner.class)`到你的测试用例中，否则这个注解将被忽略。如果你使用JUnit5，如果`@SpringBootTest`已经出现，那就没有必要提供等价的`@ExtendWith(SpringExtension)`。

默认情况下，`SpringBootTest`不会启动服务器，你可以使用`SpringBootTest`的`webEnvironment`属性来进一步定义你的测试用例如何执行。

- MOCK（默认）：加载一个Web`ApplicationContext`并提供一个模拟的Web环境。内嵌的服务器在不会被启动。如果在你的类路径中Web环境不可用，那这个模式会透明回退到创建一个常规的非Web `ApplicationContext`。
- RANDOM_PORT：加载一个`WebServerApplicationContext`并且提供一个真实的Web环境。内嵌服务器将被启动并监听一个随机端口。
- DEFINED_PORT：加载一个`WebServerApplicationContext`并且提供一个真实的Web环境。内嵌服务器将被启动并监听一个定义好的端口（来自你的配置文件或取默认值8080）。
- NONE：利用`SpringApplication`加载一个`ApplicationContext`但是不提供任何Web环境。

> 如果你的测试用例是`@Transactional`，默认所有方法执行完后都会回滚事务。然而，当继而使用`RANDOM_PORT`或`DEFINED_PORT`提供了一个真实的Web环境时，HTTP客户端和服务器在运行于不同的线程中，因此，也对应不同的事务。在服务器的创建的事务都不会被回退。

> 如果你为管理服务器分配一个不同的端口，那么`@SpringBootTest`带有`webEnvironment=WebEnvironment.RANDOM_PORT`时也会为管理服务器分配一个随机端口。

### 利用模拟环境进行测试

默认，`@SpringBootTest`不会启动服务器。如果你希望测试模拟环境的网站终端，你可以额外配置`MockMvc`：

```java
import org.junit.Test;
import org.junit.runner.RunWith;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.junit4.SpringRunner;
import org.springframework.test.web.servlet.MockMvc;

import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.content;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;

@RunWith(SpringRunner.class)
@SpringBootTest
@AutoConfigureMockMvc
public class MockMvcExampleTests {

	@Autowired
	private MockMvc mvc;

	@Test
	public void exampleTest() throws Exception {
		this.mvc.perform(get("/")).andExpect(status().isOk())
				.andExpect(content().string("Hello World"));
	}

}
```

> 如果你仅关注Web层，而不打算启动一个完全的`ApplicationContext`，考虑使用`@WebMvcTest`。

作为替换，你可以配置一个`WebTestClient`，就像下面所示：

```java
import org.junit.Test;
import org.junit.runner.RunWith;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.reactive.AutoConfigureWebTestClient;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.junit4.SpringRunner;
import org.springframework.test.web.reactive.server.WebTestClient;

@RunWith(SpringRunner.class)
@SpringBootTest
@AutoConfigureWebTestClient
public class MockWebTestClientExampleTests {

	@Autowired
	private WebTestClient webClient;

	@Test
	public void exampleTest() {
		this.webClient.get().uri("/").exchange().expectStatus().isOk()
				.expectBody(String.class).isEqualTo("Hello World");
	}

}
```

### 测试运行的服务器

如果你需要启动一个完整的服务器，我们推荐你使用随机端口，如果你们使用 `@SpringBootTest(webEnvironment=WebEnvironment.RANDOM_PORT)`，每次执行测试时都会选择一个随机端口。

`@LocalServerPort`注解可以被作为注入实际使用的端口注入到你的测试用例中。测试中需要对服务器调用REST风格请求，可以通过注入`WebTestClient`实现，它会决定执行的服务器的相对路径，并提供了一组特殊的APi用于校验响应。

```java
import org.junit.Test;
import org.junit.runner.RunWith;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.context.SpringBootTest.WebEnvironment;
import org.springframework.test.context.junit4.SpringRunner;
import org.springframework.test.web.reactive.server.WebTestClient;

@RunWith(SpringRunner.class)
@SpringBootTest(webEnvironment = WebEnvironment.RANDOM_PORT)
public class RandomPortWebTestClientExampleTests {

	@Autowired
	private WebTestClient webClient;

	@Test
	public void exampleTest() {
		this.webClient.get().uri("/").exchange().expectStatus().isOk()
				.expectBody(String.class).isEqualTo("Hello World");
	}

}
```

上例需要类路径包含`spring-webflux`，如果你不能加入webflux，Spring Boot也提供了`TestRestTemplate`工具包：

```java
import org.junit.Test;
import org.junit.runner.RunWith;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.context.SpringBootTest.WebEnvironment;
import org.springframework.boot.test.web.client.TestRestTemplate;
import org.springframework.test.context.junit4.SpringRunner;

import static org.assertj.core.api.Assertions.assertThat;

@RunWith(SpringRunner.class)
@SpringBootTest(webEnvironment = WebEnvironment.RANDOM_PORT)
public class RandomPortTestRestTemplateExampleTests {

	@Autowired
	private TestRestTemplate restTemplate;

	@Test
	public void exampleTest() {
		String body = this.restTemplate.getForObject("/", String.class);
		assertThat(body).isEqualTo("Hello World");
	}

}
```

### 模拟和监控Bean

当执行测试用例，有时候必须模拟应用上下文中的模块。比如，你可能有一些在开发时不可用的远程服务。Mocking在模拟一些很难在真实环境触发的失败时非常有用。

Spring Boot包含了`@MockBean`注解，用于在你的`ApplicationContext`中定义Mockito的模拟对象。你可以使用这个注解新建Bean或替换已存在的Bean定义。这个注解可以直接在测试用例类上使用，可以在测试用例的字段上是由，或者使用在`@Configuration`类或字段上。当使用在字段上时，被创建的的模拟对象也会被注入。在每个测试方法执行完后，模拟的Bean都会被重置。

> 如果你的测试使用了至少一个Spring Boot的测试注册（比如`@SpringBootTest`），这个特性就会被自动启用。要在一个不同的筹划中使用这些特性，必须显式增加监听器，就如下面例子所示：
>
> `@TestExecutionListeners(MockitoTestExecutionListener.class)`

下面的样例使用模拟对象替换一个现存的`RemoteService`Bean：

```java
import org.junit.*;
import org.junit.runner.*;
import org.springframework.beans.factory.annotation.*;
import org.springframework.boot.test.context.*;
import org.springframework.boot.test.mock.mockito.*;
import org.springframework.test.context.junit4.*;

import static org.assertj.core.api.Assertions.*;
import static org.mockito.BDDMockito.*;

@RunWith(SpringRunner.class)
@SpringBootTest
public class MyTests {

	@MockBean
	private RemoteService remoteService;

	@Autowired
	private Reverser reverser;

	@Test
	public void exampleTest() {
		// RemoteService has been injected into the reverser bean
		given(this.remoteService.someCall()).willReturn("mock");
		String reverse = reverser.reverseSomeCall();
		assertThat(reverse).isEqualTo("kcom");
	}

}
```

另外，你可以使用`@SpyBean`取包装任何现存的Bean。

> 由于Spring的测试框架会为共享相同配置的测试用例，在测试用例之间缓存应用上下文，而`@MockBean`或`@SpyBean`会影响到缓存使用的key，很可能会增加上下文的数量。

> 如果你在打了`@Cacheable`方法中使用了`@SpyBean`来监控一个Bean，方法使用名字来引用参数，你必须要使用`-parameter`参数编译应用。

### 自动配置测试

Spring Boot的自动测试系统做的很好，但是对于测试来说内容可能太多了。仅加载部分的配置来测试应用的切片往往很有用。比如，你可能希望测试Spring MVC控制器正确映射URL，并且你不希望涉及数据库的调用，或者你仅需要发测试JPA实体类，对web层不感兴趣。

`spring-boot-test-autoconfigure`模块包含了数个注解，用于自动化配置这些切片。它们中的每一个都以类似的方式工作，提供了类似`@...Test`注解，加载`ApplicationContext`和一个或更多`@AutoConfigure...`注解，用于自定义自动配置设置。

### 自动配置JPA测试

你可以使用`@DataJpaTest`注解，测试JPA应用。默认，它配置了内存中嵌入数据库，扫描`@Entity`类，并且配置Spring Data JPA仓库。常规`@Component`的Bean不会加载到`ApplicationContext`中。

默认，Data JPA的测试用例时事务性的，并在方法结束后回滚事务。你可以关闭一个方法或整个类中的事务：

```java
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest;
import org.springframework.test.context.junit4.SpringRunner;
import org.springframework.transaction.annotation.Propagation;
import org.springframework.transaction.annotation.Transactional;

@RunWith(SpringRunner.class)
@DataJpaTest
@Transactional(propagation = Propagation.NOT_SUPPORTED)
public class ExampleNonTransactionalTests {

}
```

Data JPA测试用例可以自动注入`TestEntityManager`的Bean，它是标准JPA`EntityManager`的专用于测试的替代物。如果你希望在`@DataJpaTest`实例外使用`TestEntityManger`，你也可以使用`@AutoConfigureTestEntityManager`注解。一个`JdbcTemplate`也是可用的，下面的样例展示了如何使用`@DataJpaTest`：

```java
import org.junit.*;
import org.junit.runner.*;
import org.springframework.boot.test.autoconfigure.orm.jpa.*;

import static org.assertj.core.api.Assertions.*;

@RunWith(SpringRunner.class)
@DataJpaTest
public class ExampleRepositoryTests {

	@Autowired
	private TestEntityManager entityManager;

	@Autowired
	private UserRepository repository;

	@Test
	public void testExample() throws Exception {
		this.entityManager.persist(new User("sboot", "1234"));
		User user = this.repository.findByUsername("sboot");
		assertThat(user.getUsername()).isEqualTo("sboot");
		assertThat(user.getVin()).isEqualTo("1234");
	}

}
```

由于它们很快且不需要额外的安装，内存中的嵌入式数据库通常在测试时工作得很好。如果你更想针对真实的数据库执行测试用例，你可以使用`@AutoConfigureTestDatabase`注解，就如下面样例所示：

```java
@RunWith(SpringRunner.class)
@DataJpaTest
@AutoConfigureTestDatabase(replace=Replace.NONE)
public class ExampleRepositoryTests {

	// ...

}
```

