---
categories: framework
layout: post
---

- Table
{:toc}
# 依赖管理

## 继承式依赖管理

springboot的依赖管理非常简单，只需要让你的pom文件继承`spring-boot-starter-parent`即可。

比如：

```xml
<project>
	<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
		<version>2.1.2.RELEASE</version>
	</parent>
</<project>
```

这样你在使用spring所管理的依赖时就不能手动指定版本，因为这些都已经在`spring-boot-starter-parent`的pom中预先定义好了。

将依赖管理交给springboot，最大的好处就是不会在发生版本的冲突问题，也可以减少你在整合不同依赖时解决依赖冲突所花的时间。

## 导入式依赖管理

由于Maven的pom文件仅支持单继承，因此一旦你通过继承的方式引入springboot的依赖管理，那么你就不能再继续继承其它的pom文件了，这对于大多数的应用来说都是足够的，但是缺乏灵活性。因此springboot允许你以另外一种方式引入依赖。

```xml
<project>
	<dependencyManagement>
        <dependencies>
			<dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-starter-parent</artifactId>
                <version>${spring-boot.version}</version>
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
                    <version>${spring-boot.version}</version>
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
</<project>
```

# 配置文件

## 文件格式

spring的配置文件支持properties和yaml两种格式，但是比较推荐yaml的方式，因为它能避免一些重复前缀并且更加清晰。

properties的样例：

```properties
spring.datasource.url=jdbc:h2:mem:taodb
spring.datasource.driverClassName=org.h2.Driver
spring.datasource.username=sa
spring.datasource.password=
```

对应的yaml文件：

```yaml
spring:
	datasource:
		url: jdbc:h2:mem:taodb
		driverClassName: org.h2.Driver
		username: sa
		password: 
```

## 配置文件名称

配置文件的名称由SpringBoot中的`spring.config.name`属性决定，它的默认值是`application`。

```sh
java -jar myproject.jar --spring.config.name=myproject
```

## 配置文件位置

SpringBoot应用会从名为`application.yaml`的文件中加载属性。而读取的位置顺序如下（序号越小优先级越高，多个文件中配置相同属性时取优先级最高文件中的属性）：

1. 当前目录下的/config子目录。
2. 当前目录
3. 类路径中的/config目录下
4. 类路径的根目录下

在SpringBoot中这个位置也是可以指定的，你可以使用`spring.config.location`指定具体的位置，也可以指定多个位置（通过多个逗号分隔）。

```sh
java -jar myproject.jar --spring.config.location=classpath:/default.properties,classpath:/override.properties
```

## profile配置文件

在不同的profile下，springboot应用除了会加载`application.yaml`配置文件外，还会自动加载profile对应配置文件，其约定为`application-{profile}.yaml`文件，而后者的加载优先级高于前者。

这样你就能为不同的环境提供不同的配置文件，比如，提供下面两个配置文件

- application-dev.yaml
- application-prod.yaml

前者用于开发环境使用，里面配置开发环境使用的服务地址，而后者则配置生产环境使用的服务地址。这样在切换到生产环境时，只需要使用下面命令启动应用：

```sh
java -jar myproject.jar --spring.profiles.active=prod
```

除了上面在多个配置文件中配置属性的方式外，你还可以在一个yaml文件中指定属性在特定的profile下才会启用，比如下面：

```yaml
---
my.property: fromyamlfile
---
spring.profiles: prod
spring.profiles.include:
  - proddb
  - prodmq
```

在名为prod的profile启用时，会加载`application-proddb.yaml`和`application-prodmq.yaml`中的属性。

## 配置文件中使用占位符

在配置文件中你可以使用占位符来引用其它属性，比如：

```yaml
app:
	name: MyApp
	description: ${app.name} is a Spring Boot Application
```

# 属性

## 属性来源

SpringBoot的属性来源有许多，下面按照优先级从高到低列出：

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

## 属性注入

要将SpringBoot属性注入到Bean中，你可以`@Value`注解：

```java
import org.springframework.stereotype.*
import org.springframework.beans.factory.annotation.*

@Component
public class MyBean {

    @Value("${spring.profile.active}")
    private String profile;

    // ...

}
```

## 配置类

除了像属性注入一样一个个注入属性，你还可以直接将属性映射到某给类的字段上。比如在类路径下提供下面配置文件`myserver.yml`：

```yaml
myServer:
	id: xx01
	version: 1.0.0
	port: 8080
```

之后提供下面配置类（注意要提供setter）：

```java
@Component
@ConfigurationProperties(prefix = "myserver")
@PropertySource("classpath:myserver.yml")
public class MyServerProperties{
    private String id;
    private String version;
    private int port;
    public void setId(String id){
        this.id = id;
    }
    public void setVersion(String version){
        this.version = version;
    }
    public void setPort(int port){
        this.port = port;
    }
}
```

之后使用这些属性去创建我们的MyServer：

```java
@Configuration
public class BeanConfig{
    @Bean
    public MyServer myServer(@Autowired MyServerProperties properties){
        return MyServer
            .newBuilder()
            .id(properties.getId())
            .version(properties.getVersion())
            .port(properties.getPort())
            .build();
    }
}
```

