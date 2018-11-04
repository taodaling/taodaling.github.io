---
layout: post
categories: java
---

Java中的类是通过类加载器（继承自ClassLoader）加载的。每个类加载器都有一个parent，形成了树型结构。

但是类加载器也是java类，那么类加载器必然也是被另外一个类加载器所加载的，这样的话就导致了一个先有鸡还是现有蛋的问题。打破这一窘境的方式就是利用C++写的引导类加载器来加载Java的核心库，之后利用Java写的扩展类加载器来加载Java的扩展库，而系统类加载器则用于加载`CLASSPATH`上的类文件。这样我们的系统就得以起来了。（系统类加载器可以通过`ClassLoader.getSystemClassLoader()`取得)扩展类加载器和系统类加载器均由引导类加载器所加载，其parent关系为`引导类加载器`<--`扩展类加载器`<--`系统类加载器`。用户可以实现自己的类加器，从而实现自定义的类操作。

为了避免重复加载，每个类加载器都有自己的缓存，通过`findLoadedClass`方法可以从缓存中查找已加载过的类。同样为了避免重复加载，因此java采用Parent代理的方式来处理，简单来说就是当有请求要求自己加载某个类时，首先尝试让parent来加载，如果parent加载失败则再自己手动加载。当然这仅是规范而已，并不要求一定遵守。但是当调用ClassLoader中defineClass从字节数组中加载类对象时，JVM要求同一个类加载器对象不能重复多次定义同名类。

每个线程上均绑定了一个类加载器，称为线程上下文加载器。我们可以通过`Thread.getContextClassLoader`取得线程上下文加载器。我们也可以设置上下文加载器，通过`Thread.setContextClassLoader`即可。默认线程会继承父线程的上下文加载器，而初始线程的默认上下文加载器为系统类加载器。

每个class对象都记录了加载它的类加载器，通过`Class.getClassLoader`取得。当类加载进行到解析步骤时，引用的外部类将导致外部类的加载，而用于加载这些
外部类的就是加载当前类的类加载器，这是无法改变的。那么该如何使用自定义类加载器加载类呢？java中提供了`Class.forName`方法。其有两个版本：

- forName(String name)
- forName(String name, boolean initialize, ClassLoader loader)

两者都指定类的全限定名来加载特定类。前者使用调用者类的类加载器作为类加载器，并要求立即执行类的初始化过程。而后者则允许我们选择是否立即执行初始化过程以及使用自定义类加载器来控制这个过程。

有了这个机制我们就可以使用自定义的类加载器来做非常多有趣的事情。比如我们实现一个类加载器，但是并不返回实际的类，而是借助asm框架进行代理包装，
这样我们就能在调用任意方法的时候记录方法执行时间和次数从而实现了一个性能测试框架。或者我们可以在方法的每一行代码后都打桩，这样我们就可以记录哪些代码被执行过，哪些没有，这就实现了代码覆盖率检测。下面我示范另外一个比较好玩的玩法，就是借助类加载器进行代码热替换（Hot swap）：

```java
public class SimpleHotSwap implements Runnable {

    @Override
    public void run() {
        System.out.println(0);
    }

    public static class HotSwapClassLoader extends ClassLoader {
        public HotSwapClassLoader(ClassLoader parent) {
            super(parent);
        }

        @Override
        protected Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException {
            if (!name.startsWith("cn.dalt.test.")) {
                return super.loadClass(name, resolve);
            }

            Class c = findLoadedClass(name);
            if (c == null) {
                c = findClass(name);
            }

            if (resolve) {
                resolveClass(c);
            }
            return c;
        }

        @Override
        protected Class<?> findClass(String name) throws ClassNotFoundException {
            String path = name.replace('.', '/') + ".class";
            ByteArrayOutputStream bos = new ByteArrayOutputStream();
            try (InputStream is = getResourceAsStream(path)) {
                if (is == null) {
                    throw new ClassNotFoundException();
                }

                byte[] buf = new byte[4096];
                int num;
                while ((num = is.read(buf)) != -1) {
                    bos.write(buf, 0, num);
                }
            } catch (IOException e) {
                throw new RuntimeException(e);
            }

            byte[] data = bos.toByteArray();
            return defineClass(name, data, 0, data.length);
        }
    }

    public static void main(String[] args) throws Exception {
        do {
            Class cls = Class.forName(SimpleHotSwap.class.getCanonicalName(), true, new HotSwapClassLoader(ClassLoader.getSystemClassLoader()));
            Runnable runnable = (Runnable) cls.newInstance();
            runnable.run();
        } while (System.in.read() != 'q');
    }
}
```
你可以尝试更改`System.out.println(0);`中的数字并重新编译代码后按下回车键，即可看到新类被加载并打印出了新设的数字。
