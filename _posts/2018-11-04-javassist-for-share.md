---
categories: share
layout: post
---

{:toc}

# 前置知识

- Java类加载流程

- Java类加载器

## JVM的类加载机制

java的类在javac指令编译后为class文件。JVM的类加载分为七个阶段：

1. 加载
2. 验证
3. 准备
4. 解析
5. 初始化
6. 使用
7. 卸载

仅讨论加载阶段：

1. 通过一个类的全限定名来获取定义此类的二进制字节流（并没有指明要从一个Class文件中获取，可以从其他渠道，譬如：网络、动态生成、数据库等）；
2. 将这个字节流所代表的静态存储结构转化为方法区的运行时数据结构；
3. 在内存中(对于HotSpot虚拟就而言就是方法区)生成一个代表这个类的java.lang.Class对象，作为方法区这个类的各种数据的访问入口；

## ClassLoader

Java中利用ClassLoader加载类。ClassLoader允许有父ClassLoader，因此呈树形。

但是类加载器也是java类，那么类加载器必然也是被另外一个类加载器所加载的，这样的话就导致了一个先有鸡还是现有蛋的问题。打破这一窘境的方式就是利用C++写的引导类加载器来加载Java的核心库，之后利用Java写的扩展类加载器来加载Java的扩展库，而系统类加载器则用于加载`CLASSPATH`上的类文件。这样我们的系统就得以起来了。（系统类加载器可以通过`ClassLoader.getSystemClassLoader()`取得)扩展类加载器和系统类加载器均由引导类加载器所加载，其parent关系为`引导类加载器`<--`扩展类加载器`<--`系统类加载器`。用户可以实现自己的类加器，从而实现自定义的类操作。

为了避免重复加载，每个类加载器都有自己的缓存，通过`findLoadedClass`方法可以从缓存中查找已加载过的类。同样为了避免重复加载，因此java采用Parent代理的方式来处理，简单来说就是当有请求要求自己加载某个类时，首先尝试让parent来加载，如果parent加载失败则再自己手动加载。当然这仅是规范而已，并不要求一定遵守。但是当调用ClassLoader中defineClass从字节数组中加载类对象时，JVM要求同一个类加载器对象不能重复多次定义同名类。

每个线程上均绑定了一个类加载器，称为线程上下文加载器。我们可以通过`Thread.getContextClassLoader`取得线程上下文加载器。我们也可以设置上下文加载器，通过`Thread.setContextClassLoader`即可。默认线程会继承父线程的上下文加载器，而初始线程的默认上下文加载器为系统类加载器。

# 字节码

Java源代码编译后得到的.class文件中包含的二进制内容即为字节码。

动态字节码即直接修改或创建字节码内容而跳过源代码编辑和编译的过程。

流行的动态字节码生成工具有`ASM`，`CGLIB`，`JAVASSIST`，其中CGLIB的底层是ASM。

`OpenJDK`，`JMH`使用了Asm。

`Spring`底层使用Cglib作为动态字节码工具。

`Hibernate`，`Dubbo`等使用了Javassist。



## 动态字节码与反射的区别

Java中动态字节码和反射都能实现动态代理，但是动态字节码的性能与一般java程序性能相同，但是反射性能会远差于一般java代码性能。

反射只能在方法被调用之前和之后做一定处理，但是动态字节码可以直接修改方法内的字节码（比如打桩实现代码覆盖率计算）。



# Javassist

## 什么是Javassist

> Javassist是一个开源的处理Java字节码的类库。是由东京工业大学的数学和计算机科学系的 Shigeru Chiba （千叶 滋）所创建的，它已加入了开放源代码JBoss 应用服务器项目。

Javassist的[Github地址](https://github.com/jboss-javassist/javassist)



## 为什么选择Javassist

简单！！！

Javassist内置了一个微型的编译器程序，可以直接将源代码编译为字节码。



# Javassist案例

## 增加toString

下面展示如何为类生成正确的toString()函数。

```java
public interface ValueHolder{
    public void setValue(int value);
    public int getValue();
}
```

```java
public class ValueHolderImpl implements ValueHolder{
    private int value;
    public void setValue(int value){this.value = value;}
    public int getValue(){return this.value;}
}
```

假设我们已经将ValueHolder编译为字节码了，现在我们利用Javassist为ValueHolder增加合理的toString方法。

```java
public class AddToStringExample {
    public static void main(String[] args) throws NotFoundException, CannotCompileException, IllegalAccessException, InstantiationException {
        ValueHolder normalValueHolder = new ValueHolderImpl();
        normalValueHolder.setValue(1);
        System.out.println(normalValueHolder);

        CtClass ctClass = ClassPool.getDefault().get(ValueHolderImpl.class.getCanonicalName());
        ctClass.addMethod(CtMethod.make(
                "    public String toString() {\n" +
                "        return \"\" + getValue();\n" +
                "    }", ctClass));

        ClassLoader newClassLoader = new PatternClassLoader(Pattern.compile("cn\\.dalt\\.test\\.javassist\\.ValueHolderImpl"));
        Class<?> cls = ctClass.toClass(newClassLoader, null);
        ValueHolder valueHolder = (ValueHolder) cls.newInstance();
        valueHolder.setValue(1);
        System.out.println(valueHolder.toString());
    }
}

```

## 增加计数器

假设我们需要统计ValueHolder的set调用次数。

```java
public class AddCountableInterface {
    public static void main(String[] args) throws NotFoundException, CannotCompileException, IllegalAccessException, InstantiationException {
        CtClass ctClass = ClassPool.getDefault().get(ValueHolderImpl.class.getCanonicalName());
        ctClass.addField(CtField.make("private int cnt;", ctClass));
        ctClass.getMethod("setValue", "(I)V").insertBefore("{cnt++;}");
        ctClass.addMethod(CtMethod.make("public int getSetValueInvokeTime(){return cnt;}", ctClass));
        ctClass.addInterface(ClassPool.getDefault().get(Countable.class.getCanonicalName()));

        ClassLoader newClassLoader = new PatternClassLoader(Pattern.compile("cn\\.dalt\\.test\\.javassist\\.ValueHolderImpl"));
        Class<?> cls = ctClass.toClass(newClassLoader, null);

        Countable instance = (Countable) cls.newInstance();
        instance.setValue(1);
        instance.setValue(2);
        instance.setValue(1);
        System.out.println(instance.getSetValueInvokeTime());
    }
}

```

## 返回结果非负

我们希望当getValue返回值为负数时，返回0。

```java
public class AtLeast0 {
    public static void main(String[] args) throws NotFoundException, CannotCompileException, IllegalAccessException, InstantiationException {
        CtClass ctClass = ClassPool.getDefault().get(ValueHolderImpl.class.getCanonicalName());
        ctClass.getMethod("getValue", "()I").insertAfter("{ $_ = $_ < 0 ? 0 : $_; }");

        ClassLoader newClassLoader = new PatternClassLoader(Pattern.compile("cn\\.dalt\\.test\\.javassist\\.ValueHolderImpl"));
        Class<?> cls = ctClass.toClass(newClassLoader, null);
        ValueHolder instance = (ValueHolder) cls.newInstance();
        instance.setValue(1);
        System.out.println(instance.getValue());
        instance.setValue(-1);
        System.out.println(instance.getValue());
    }
}
```

## 增加Supplier接口

显然每次都利用反射方式(newInstance)创建对象非常慢，因此我们为实例增加supplier接口。

```java
public class AsSupplier {
    public static void main(String[] args) throws NotFoundException, CannotCompileException, IllegalAccessException, InstantiationException {
        CtClass ctClass = ClassPool.getDefault().get(ValueHolderImpl.class.getCanonicalName());
        ctClass.setName(ctClass.getName() + "$AsSupplier");
        ctClass.setSuperclass(ClassPool.getDefault().get(ValueHolderImpl.class.getCanonicalName()));

        ctClass.addMethod(CtMethod.make("public Object get(){return new " + ctClass.getName() + "();}", ctClass));
        ctClass.addInterface(ClassPool.getDefault().get(Supplier.class.getCanonicalName()));

        Class<?> cls = ctClass.toClass();
        ValueHolderImpl instance = (ValueHolderImpl) cls.newInstance();
        Supplier<ValueHolderImpl> supplier = (Supplier<ValueHolderImpl>) instance;
        System.out.println(supplier.get());
    }
}

```



# 附录

## Type descriptors

Internal names are used only for types that are constrained to be class or interface types, in all other situations, such as field types, Java types are represented in compiled classes with *type descriptors*.The descriptors of the primitive types are single characters, and the descriptor of a class type is the internal name of this class, preceded by L and followed by a semicolon.

| Java type    | Type descriptoR      |
| ------------ | -------------------- |
| boolean      | Z                    |
| char         | C                    |
| byte         | B                    |
| short        | S                    |
| int          | I                    |
| float        | F                    |
| long         | J                    |
| double       | D                    |
| Object       | Ljava/lang/Object;   |
| int[]        | [I                   |
| Object\[]\[] | [[Ljava/lang/Object; |

## Method descriptors

A method descriptor is a list of type descriptors that describe the parameter types and the return type of a method, in a single string.A method descriptor starts with a left parenthesis, followed by the type descriptors of each formal parameter, followed by a right parenthesis, followed by the type descripor of the return type, or V if the method returns void(A method descriptor doesn't contain the name of method or the argument name).

| Method declaration in source file | Method descriptor      |
| --------------------------------- | ---------------------- |
| void m(int i, float f)            | (IF)V                  |
| int m(int i, String s)            | (ILjava/lang/String;)I |
| int[] m(int[] i)                  | ([I)[I                 |