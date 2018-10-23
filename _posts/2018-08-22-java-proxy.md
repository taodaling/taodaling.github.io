---
layout: post
categories: framework
---

代理模式实际上是基本等同于装饰器模式，其目的是避免用户对实体的直接访问，这样我们就可以在用户和被访问的实体之间创建一位代理，这样我们就能在用户访问实体时
做一些控制。

代理模式非常有用，比如我们希望能在用户访问时能记录日志，或者能校验用户是否拥有权限，或者不允许用户调用一些方法，或者我们调用方法时实际上会调用远程
服务器上的某个对象的方法(RMI)。

代理模式的玩法一般是:首先创建接口，而用户仅通过接口来使用被代理对象，从而解耦用户和被代理对象。之后我们让被代理对象和代理均实现接口，之后用代理对象
持有被代理对象并返回给用户。注意代理如果以接口持有被代理对象，那么还能解耦代理和被代理对象。

```java
class Mother{
  //访问Mary并翻翻她的日记
  public void visit(Mary mary){
    Diary diary = mary.provide();
    
    String page = diary.get(...);
  }
}

class Mary{
  Diary diary = new DairyImpl();
  //给妈妈自己的日记
  public Diary provide(){
    return new DiaryProxy(diary);
  }
}

interface Diary{
  String getPage(int index);
}

class DairyImpl implements Diary{
  public String getPage(int index){...}
}

class DiaryProxy implements Diary{
  Diary inner;
  public DiaryProxy(Diary inner){
    this.inner = inner;
  }
  
  public String getPage(int index){
    if(不希望妈妈看到页index){
      return "Nothing";
    }
    return inner.get(index);
  }
}
```

java中提供了多种代理的机制，下面仅讨论两种。

# JDK自带的Proxy

JDK中自带了Proxy，我们需要创建一个InvocationHandler，并让Proxy为我们创建实现若干特定接口的代理，之后代理的任意方法被调用时，都会转发到我们的
InvocationHandler进行处理。

```java
final Runnable task = new Runnable() {
  @Override
    public void run() {
    System.out.println("run...");
  }
};
InvocationHandler proxy = new InvocationHandler() {
  @Override
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    System.out.println("logging at " + new Date());
    System.out.println("invoke method " + method.getName() + " with args " + Arrays.toString(args));
    Object ret = method.invoke(task, args);
    System.out.println("return " + ret);
    return ret;
  }
};

Runnable proxyTask = (Runnable) Proxy.newProxyInstance(Thread.currentThread().getContextClassLoader(), new Class[]{Runnable.class}, proxy);

proxyTask.run();
```

# cglib的Proxy

JDK自带的代理方法有如下的问题：
1. 使用反射，导致性能下降。
2. 无法代理实现类。

而cglib则利用动态字节码(asm)技术，规避了上面提到的两个问题。

```java
Enhancer enhancer = new Enhancer();
enhancer.setSuperclass(Object.class);
Object object = new CGLibTest();
enhancer.setCallback(new MethodInterceptor() {
  @Override
  public Object intercept(Object obj, Method method, Object[] args, MethodProxy proxy) throws Throwable {
    System.out.println(MessageFormat.format("{0} is invoked with {1}", method, Arrays.toString(args)));
    Object result = proxy.invoke(object, args);
    System.out.println(MessageFormat.format("return {0}", result));
    return result;
  }
});
Object proxy = enhancer.create();
proxy.hashCode();
proxy.toString();
```

如果我们希望能创建多个代理对象，那么可以借助Factory接口。
```java
enhancer.setUseFactory(true);
Factory proxyFactory = (Factory) enhancer.create();
Object proxy = proxyFactory.newInstance(interceptor);
```
