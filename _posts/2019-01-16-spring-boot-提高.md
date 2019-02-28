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

# 代理

## 异步调用

Spring提供了@Async和@EnableAsync注解，前者用于标注方法为异步调用，后者标注支持异步调用。

```java
@Service
@EnableAsync
public class AsyncService{
    @Async
    public void invoke(){
        try {
            Thread.sleep(3000);
        } catch (InterruptedException e) {
        }
    }
}
```

Spring允许你提供自己的线程池。默认情况下，Spring会按照下面顺序进行装配：

- 如果上下文存在唯一的org.springframework.core.task.TaskExecutor类型的bean，那么就使用它。
- 如果上下文存在名为taskExecutor的java.util.concurrent.Executor的bean，那么使用它。
- 使用org.springframework.core.task.SimpleAsyncTaskExecutor来处理异步方法调用。

如果上面的这些还无法满足你的需求，你可以提供一个实现了AsyncConfigurer接口的配置类。

```java
@Configuration
@EnableAsync
public class AppConfig implements AsyncConfigurer {
  
       @Override
       public Executor getAsyncExecutor() {
           ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
           executor.setCorePoolSize(7);
           executor.setMaxPoolSize(42);
           executor.setQueueCapacity(11);
           executor.setThreadNamePrefix("MyExecutor-");
           executor.initialize();
           return executor;
       }
  
       @Override
       public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
           return MyAsyncUncaughtExceptionHandler();
       }
}
```

## 事务

Spring通过@Transactional注解提供了事务的支持，使用方法非常简单。

```java
public class BizService{
    @Transactional
    public void doBizWork(){
        ...
    }
}
```

@Transactional也可以放在类上，这样每个方法都会包装在事务中。默认情况下，要回滚事务，只需要在方法中抛出方法上没有声明的异常类型。

## 定时任务

Spring提供了@Scheduled注解，用于标注组件的一个方法周期性执行。

```java
@Scheduled(fixDelay = 5000)
public void doSomething(){
    ...
}
```

## 用代理替换Bean

Spring允许你用自己生成的代理，替换已有的Bean。我们需要使用Spring提供的一个特殊接口BeanPostProcessor，Spring会在每个注册的Bean初始化的前后掉调用接口，并允许你用自己的代理替代这些Bean。

下面提供一个案例，案例将所有用@MyAnnotation注解的类型的组件，全部用代理替换，并且在它的内部方法调用时会输出调用的具体信息。

```java
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE})
public @interface MyAnnotation {
}

```

```java
@Component
public class MyBeanPostProcess implements BeanPostProcessor {
    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        System.out.println("postProcessBeforeInitialization for " + beanName);

        if (bean.getClass().getAnnotation(MyAnnotation.class) == null) {
            return bean;
        }
        return Proxy.newProxyInstance(bean.getClass().getClassLoader(), bean.getClass().getInterfaces(), new InvocationHandler() {
            @Override
            public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                Object result = method.invoke(bean, args);
                System.out.println(bean + "." + method + "(" + Arrays.toString(args) + ") = " + result);
                return result;
            }
        });
    }
}
```

```java
public interface MyService {
    public String greet();
}
```

```java
@MyAnnotation
@Component
public class MyServiceImpl implements MyService{
    public String greet() {
        return "Hello, world!";
    }
}
```

```java
@SpringBootApplication
public class Main {
    public static void main(String[] args) {
        ApplicationContext context = SpringApplication.run(Main.class, args);
        MyService myServiceImpl = context.getBean(MyService.class);
        System.out.println(myServiceImpl.greet());
    }
}
```

## 自动扫描接口并创建代理

大家可能都用过dubbo，mybatis等框架，它们和Spring结合的很好。下面我们我们也定义自己的一个注解@Answer，并定义一些加了该注解的接口。

```java
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.METHOD, ElementType.TYPE})
public @interface Answer {
    String value();
}
```

@Answer加在类上表示需要为这个接口创建代理Bean，而属性value表示默认值。@Answer加在接口方法上，表示这个方法的返回值被特殊指定为value（没有指定的则取接口注解上的默认值），比如：

```java
@Answer("default")
public interface MyService {
    @Answer("Hello")
    public String hello();

    public String world();
}
```

上面这个接口的实际含义应该是希望我们自动生成一个下面的实现类：

```java
@Component
public interface MyServiceImpl{
    public String hello(){return "hello";}
    public String world(){return "default";}
}
```

下面实现Spring的ImportBeanDefinitionRegistrar接口，提供一个实现类：

```java
public class MyImportBeanDefinitionRegistrar implements ImportBeanDefinitionRegistrar {
    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
        ClassPathBeanDefinitionScanner scanner = new ClassPathBeanDefinitionScanner(registry, false) {
            @Override
            protected boolean isCandidateComponent(AnnotatedBeanDefinition beanDefinition) {
                return beanDefinition.getMetadata().isInterface();
            }

            @Override
            protected void postProcessBeanDefinition(AbstractBeanDefinition definition, String beanName) {
                super.postProcessBeanDefinition(definition, beanName);

                try {
                    final Class<?> cls = Class.forName(definition.getBeanClassName());
                    definition.setInstanceSupplier(() -> {
                        return Proxy.newProxyInstance(cls.getClassLoader(),
                                new Class[]{cls}, new InvocationHandler() {
                                    @Override
                                    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                                        if (method.getAnnotation(Answer.class) != null) {
                                            return method.getAnnotation(Answer.class).value();
                                        } else {
                                            return cls.getAnnotation(Answer.class).value();
                                        }
                                    }
                                });
                    });
                } catch (ClassNotFoundException e) {
                    throw new RuntimeException(e);
                }
            }
        };
        scanner.addIncludeFilter((metadataReader, metadataReaderFactory) -> metadataReader.getAnnotationMetadata().hasAnnotation(Answer.class.getCanonicalName()));
        scanner.scan("com.daltao");
    }
}
```

最后就是我们的主类了：

```java
@SpringBootApplication
@Import(MyImportBeanDefinitionRegistrar.class)
public class Main {
    public static void main(String[] args) {
        ApplicationContext context = SpringApplication.run(Main.class, args);
        MyService myService = context.getBean(MyService.class);
        System.out.println(myService.hello());
        System.out.println(myService.world());
    }
}
```

# 参考

- [https://spring.io/guides/gs/async-method/](https://spring.io/guides/gs/async-method/)