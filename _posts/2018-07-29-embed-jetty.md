---
layout: post
categories: framework
---

*本文仅仅是Jetty部分文档的直接翻译和一些个人理解，原文地址为[Jetty Document](http://www.eclipse.org/jetty/documentation/9.4.11.v20180605/embedding-jetty.html)*

* 目录
{:toc}

# 1.简介
> Jetty是一个纯粹的基于Java的网页服务器和Java Servlet容器。尽管网页服务器通常用来为人们呈现文档，但是Jetty通常在较大的软件框架中用于计算机与计算机之间的通信。Jetty作为Eclipse基金会的一部分，是一个自由和开源项目。该网页服务器被用在Apache ActiveMQ、Alfresco、Apache Geronimo、Apache Maven、Apache Spark、Google App Engine、Eclipse、FUSE、Twitter's Streaming API、Zimbra等产品上。Jetty也是Lift、Eucalyptus、Red5、Hadoop、I2P等开源项目的服务器。 Jetty支持最新的Java Servlet API（带JSP的支持），支持SPDY和WebSocket协议。

Jetty的口号是"不要将你的项目部署在Jetty中，而是将Jetty部署在你的项目中"。Jetty嵌入式的方法可以为你现有的应用提供HTTP访问接口，从而将你的项目转换为一个服务器。

Jetty嵌入式开发的一般步骤是：
1. 创建一个Server示例
2. 配置Connectors
3. 追加Handlers
4. 启动Server

# 2.示例

## 2.1.创建一个Server

```java
public class SimplestServer
{
    public static void main( String[] args ) throws Exception
    {
        Server server = new Server(8080);
        server.start();
        server.dumpStdErr();
        server.join();
    }
}
```

之后你可以通过http://127.0.0.1:8080来访问Web应用。这个应用并不会返回什么有意义的内容，我们必须添加自定义的Handler来正确处理客户端请求。

## 2.2.追加Handler

```java
public class HelloHandler extends AbstractHandler {

    final String greeting;
    final String body;

    public HelloHandler(String greeting, String body) {
        this.greeting = greeting;
        this.body = body;
    }

    @Override
    public void handle(String target, Request baseRequest, HttpServletRequest request, HttpServletResponse response) throws IOException, ServletException {
        response.setContentType("text/html; charset=utf-8");
        response.setStatus(HttpServletResponse.SC_OK);

        response.getWriter().printf("<h1>%s</h1>%s", greeting, body);

        baseRequest.setHandled(true);
    }

    public static void main(String[] args) throws Exception {
        Server server = new Server(8080);
        server.setHandler(new HelloHandler("Good afternoon", "Nice to meet you!"));
        server.start();
        server.dumpStdErr();
        server.join();
    }
}
```

## 2.3.处理器包装器

Jetty内部提供了相当有用的处理器包装器，下面简单介绍：
 - HandlerCollection，代表处理器集合，它会调用所有包含在内的处理器轮流处理请求。
 - HandlerList，代表有序处理器列表，它会依次调用处理器，直到某个处理器抛出异常或显式声明处理完成。
 - ContextHandlerCollection，代表上下文处理器集合，其中每个处理器对应一个URL，一个请求将用所有匹配URL中最长的URL对应的处理器进行处理。
 
 下面展示一下HandlerList的简单用法：
 ```java
 public class FileServer {
    public static void main(String[] args) throws Exception {
        Server server = new Server(8080);

        ResourceHandler handler = new ResourceHandler();
        handler.setDirectoriesListed(true);
        handler.setWelcomeFiles(new String[]{"index.html"});
        handler.setResourceBase(".");

        server.setHandler(new HandlerList(handler, new DefaultHandler()));
        
        server.dumpStdErr();
        server.start();
        server.join();
    }
}
```
我们建立了一个文件浏览器，你访问浏览器后页面将列出所有目录下的文件。但是假如你显式输入不合法的路径，那么`ResourceHandler`无法处理，
那么`DefaultHandler`将接手处理并返回404。

## 2.4.增加Connector

前面我们在创建`Server`的时候直接提供了端口号，服务器将默认监听该端口，并接受连接。但是如果我们需要自定义连接的方式，可以使用显示指定Connector。
Connector负责为Server创建连接。

```java
public class OneConnector {
    public static void main(String[] args) throws Exception {
        Server server = new Server();

        //HTTP connector
        ServerConnector connector = new ServerConnector(server);
        connector.setHost("0.0.0.0");
        connector.setPort(8080);
        connector.setIdleTimeout(60 * 1000);

        server.addConnector(connector);

        server.start();
        server.dumpStdErr();
        server.join();
    }
}
```
## 2.5.嵌入式Servlet

Servlet是Java中非常常见的处理Http请求的方式，如果你希望使用Servlet的方式来处理请求，Jetty已经为你提供了`ServletHandler`。
```java
public class MinimalServlet extends HttpServlet {
    public static void main(String[] args) throws Exception {
        Server server = new Server(8080);

        ServletHandler servlet = new ServletHandler();
        servlet.addServletWithMapping(new ServletHolder(new MinimalServlet()),
                "/*");

        server.setHandler(servlet);

        server.dumpStdErr();
        server.start();
        server.join();
    }

    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        resp.setStatus(HttpServletResponse.SC_OK);
        resp.setContentType("text/html; charset=utf-8");
        resp.getWriter().printf("<h1>Hello world</h1>");
    }
}
```

- `ContextHandler`在Jetty中代表一个带上下文路径的处理器，仅当Http请求地址的前缀与上下文URL匹配，该处理器才会被执行。
- `ServletContextHandler`是ContextHandler的特殊化，其在ContextHandler的基础上增加了对`session`的支持。
- `WebAppContext`是对ServletContextHandler的扩展，允许你通过配置web.xml的方式配置Servlet。换句话说你可以直接用它驱动Web应用。
