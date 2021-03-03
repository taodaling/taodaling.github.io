---
categories: techonology
layout: post
---

- Table
{:toc}

# 源码

Java中的SPI机制是服务定义一个接口`com.daltao.X`，之后服务提供方提供一个实现类，包含在某个jar包下。对应的jar包的`META-INF/services/`下需要放入一个文件`com.daltao.X`，其中每一行写入一个实现类。这样服务的使用方就可以通过这种方式查找到服务的实现类并加载。

`ServiceLoader`是Java中用于实现SPI机制的一个组件，其用于遍历某个接口对应的所有的配置文件中的配置项。比如java中的`Charset`类就使用了`ServiceLoader`。

```java
public abstract class Charset
    implements Comparable<Charset>
    public static SortedMap<String,Charset> availableCharsets() {
        return AccessController.doPrivileged(
            new PrivilegedAction<SortedMap<String,Charset>>() {
                public SortedMap<String,Charset> run() {
                    TreeMap<String,Charset> m =
                        new TreeMap<String,Charset>(
                            ASCIICaseInsensitiveComparator.CASE_INSENSITIVE_ORDER);
                    put(standardProvider.charsets(), m);
                    CharsetProvider ecp = ExtendedProviderHolder.extendedProvider;
                    if (ecp != null)
                        put(ecp.charsets(), m);
                    for (Iterator<CharsetProvider> i = providers(); i.hasNext();) {
                        CharsetProvider cp = i.next();
                        put(cp.charsets(), m);
                    }
                    return Collections.unmodifiableSortedMap(m);
                }
            });
    }

    private static Iterator<CharsetProvider> providers() {
        return new Iterator<CharsetProvider>() {

                ClassLoader cl = ClassLoader.getSystemClassLoader();
                //利用系统ClassLoader去查找所有的CharsetProvider的服务提供方
                ServiceLoader<CharsetProvider> sl =
                    ServiceLoader.load(CharsetProvider.class, cl);
                Iterator<CharsetProvider> i = sl.iterator();

                CharsetProvider next = null;

                private boolean getNext() {
                    while (next == null) {
                        try {
                            if (!i.hasNext())
                                return false;
                            next = i.next();
                        } catch (ServiceConfigurationError sce) {
                            if (sce.getCause() instanceof SecurityException) {
                                // Ignore security exceptions
                                continue;
                            }
                            throw sce;
                        }
                    }
                    return true;
                }

                public boolean hasNext() {
                    return getNext();
                }

                public CharsetProvider next() {
                    if (!getNext())
                        throw new NoSuchElementException();
                    CharsetProvider n = next;
                    next = null;
                    return n;
                }

                public void remove() {
                    throw new UnsupportedOperationException();
                }

            };
    }
}
```

`Charset`类实际上也是通过`ServiceLoader`来查找所有的服务提供方的。下面就讲讲`ServiceLoader`的实现。

```java
    public static <S> ServiceLoader<S> load(Class<S> service,
                                            ClassLoader loader)
    {
        return new ServiceLoader<>(service, loader);
    }

    public static <S> ServiceLoader<S> load(Class<S> service) {
        ClassLoader cl = Thread.currentThread().getContextClassLoader();
        return ServiceLoader.load(service, cl);
    }
```

这两个都是比较常用的工厂方法，实际上都转发给了`ServiceLoader`的构造器。接下来看看构造器的内容。

```java
public final class ServiceLoader<S>
    implements Iterable<S>
{

    private static final String PREFIX = "META-INF/services/";

    // The class or interface representing the service being loaded
    private final Class<S> service;

    // The class loader used to locate, load, and instantiate providers
    private final ClassLoader loader;

    // The access control context taken when the ServiceLoader is created
    private final AccessControlContext acc;

    //可以理解provider是一个全局的排重缓存
    // Cached providers, in instantiation order
    private LinkedHashMap<String,S> providers = new LinkedHashMap<>();

    //惰性的迭代器
    // The current lazy-lookup iterator
    private LazyIterator lookupIterator;

    private ServiceLoader(Class<S> svc, ClassLoader cl) {
        service = Objects.requireNonNull(svc, "Service interface cannot be null");
        loader = (cl == null) ? ClassLoader.getSystemClassLoader() : cl;
        acc = (System.getSecurityManager() != null) ? AccessController.getContext() : null;
        //这里会做准备操作
        reload();
    }

    public void reload() {
        providers.clear();
        lookupIterator = new LazyIterator(service, loader);
    }

    //ServiceLoader的主要用处，返回一个迭代器，可以遍历所有的服务提供者
    //这里优先找providers缓存中的内容，如果没有才去调用内部的迭代器，找下一个文件
    public Iterator<S> iterator() {
        return new Iterator<S>() {
            
            Iterator<Map.Entry<String,S>> knownProviders
                = providers.entrySet().iterator();

            public boolean hasNext() {
                if (knownProviders.hasNext())
                    return true;
                return lookupIterator.hasNext();
            }

            public S next() {
                //如果遍历完了上一次的配置文件，就使用下一个配置文件
                if (knownProviders.hasNext())
                    return knownProviders.next().getValue();
                return lookupIterator.next();
            }

            public void remove() {
                throw new UnsupportedOperationException();
            }

        };
    }
}
```

很显然其中的看头在于`LazyIterator`的实现类。`LazyIterator`是`ServiceLoader`的一个内部类，其用于遍历所有的配置文件。

```java
    // Private inner class implementing fully-lazy provider lookup
    //
    private class LazyIterator
        implements Iterator<S>
    {

        Class<S> service;
        ClassLoader loader;
        Enumeration<URL> configs = null;
        Iterator<String> pending = null;
        String nextName = null;

        private LazyIterator(Class<S> service, ClassLoader loader) {
            this.service = service;
            this.loader = loader;
        }

        //判断是否还有下一个配置文件
        private boolean hasNextService() {
            if (nextName != null) {
                return true;
            }
            if (configs == null) {
                try {
                    //这里名称是拼接出来的
                    //PREFIX的定义为：private static final String PREFIX = "META-INF/services/";
                    String fullName = PREFIX + service.getName();
                    if (loader == null)
                        configs = ClassLoader.getSystemResources(fullName);
                    else
                        configs = loader.getResources(fullName);
                } catch (IOException x) {
                    fail(service, "Error locating configuration files", x);
                }
            }
            while ((pending == null) || !pending.hasNext()) {
                if (!configs.hasMoreElements()) {
                    return false;
                }
                pending = parse(service, configs.nextElement());
            }
            nextName = pending.next();
            return true;
        }

        private S nextService() {
            if (!hasNextService())
                throw new NoSuchElementException();
            String cn = nextName;
            nextName = null;
            Class<?> c = null;
            //先取到下一个实现类名称，并加载这个类
            try {
                c = Class.forName(cn, false, loader);
            } catch (ClassNotFoundException x) {
                fail(service,
                     "Provider " + cn + " not found");
            }
            if (!service.isAssignableFrom(c)) {
                fail(service,
                     "Provider " + cn  + " not a subtype");
            }
            try {
                S p = service.cast(c.newInstance());
                providers.put(cn, p);
                return p;
            } catch (Throwable x) {
                fail(service,
                     "Provider " + cn + " could not be instantiated",
                     x);
            }
            throw new Error();          // This cannot happen
        }

        public boolean hasNext() {
            if (acc == null) {
                return hasNextService();
            } else {
                PrivilegedAction<Boolean> action = new PrivilegedAction<Boolean>() {
                    public Boolean run() { return hasNextService(); }
                };
                return AccessController.doPrivileged(action, acc);
            }
        }

        public S next() {
            if (acc == null) {
                return nextService();
            } else {
                PrivilegedAction<S> action = new PrivilegedAction<S>() {
                    public S run() { return nextService(); }
                };
                return AccessController.doPrivileged(action, acc);
            }
        }

        public void remove() {
            throw new UnsupportedOperationException();
        }

    }
```

其中parse的方式非常粗犷。

```java
    private Iterator<String> parse(Class<?> service, URL u)
        throws ServiceConfigurationError
    {
        InputStream in = null;
        BufferedReader r = null;
        ArrayList<String> names = new ArrayList<>();
        try {
            in = u.openStream();
            r = new BufferedReader(new InputStreamReader(in, "utf-8"));
            int lc = 1;
            while ((lc = parseLine(service, u, r, lc, names)) >= 0);
        } catch (IOException x) {
            fail(service, "Error reading configuration file", x);
        } finally {
            try {
                if (r != null) r.close();
                if (in != null) in.close();
            } catch (IOException y) {
                fail(service, "Error closing configuration file", y);
            }
        }
        return names.iterator();
    }

    ////每次读取一行，如果有#号，则截取#号前面的内容
    private int parseLine(Class<?> service, URL u, BufferedReader r, int lc,
                          List<String> names)
        throws IOException, ServiceConfigurationError
    {
        String ln = r.readLine();
        if (ln == null) {
            return -1;
        }
        int ci = ln.indexOf('#');
        if (ci >= 0) ln = ln.substring(0, ci);
        ln = ln.trim();
        int n = ln.length();
        if (n != 0) {
            if ((ln.indexOf(' ') >= 0) || (ln.indexOf('\t') >= 0))
                fail(service, u, lc, "Illegal configuration-file syntax");
            int cp = ln.codePointAt(0);
            if (!Character.isJavaIdentifierStart(cp))
                fail(service, u, lc, "Illegal provider-class name: " + ln);
            for (int i = Character.charCount(cp); i < n; i += Character.charCount(cp)) {
                cp = ln.codePointAt(i);
                if (!Character.isJavaIdentifierPart(cp) && (cp != '.'))
                    fail(service, u, lc, "Illegal provider-class name: " + ln);
            }
            //将parse到的行加入到names中
            if (!providers.containsKey(ln) && !names.contains(ln))
                names.add(ln);
        }
        return lc + 1;
    }
```
