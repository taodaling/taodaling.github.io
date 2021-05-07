---
categories: code
layout: post
---

# ThreadLocal

上周我们领导和我们分享了一些Java的内存溢出问题，其中涉及到ThreadLocal引用对象内存溢出的情况。但是因为当时没读过源码，听得一知半解，这里读一下ThreadLocal的源码好了。本文的源码是来自OpenJDK 1.8中的内容。

首先什么是ThreadLocal，ThreadLocal可以作为一个线程独有的对象的一个容器，每个线程在通过`get`方法访问ThreadLocal内部包装的对象时得到结果可能是不同的。

ThreadLocal的玩法一般有两类：
- 将一些创建费时，且不是线程安全的对象放到里面去。这样就能为每个线程仅创建一个对象，加快程序的执行速度。比如`SimpleDateFormat`就是一个很好的例子。
- 在ThreadLocal放入当前上下文，常见的比如slf4j中的MDC。

由于存的东西千奇百怪，所以用个泛型不奇怪。

```java
public class ThreadLocal<T> {
}
```

我们可以直接创建一个空的ThreadLocal对象，注意你不能在创建ThreadLocal的时候就把对象放进去，因为我们需要为每个线程都维护一个对象。ThreadLocal提供了一个静态工厂方法，可以在线程第一次访问的时候自动创建对象。

```java
public class ThreadLocal<T> {
    /**
     * Creates a thread local variable. The initial value of the variable is
     * determined by invoking the {@code get} method on the {@code Supplier}.
     *
     * @param <S> the type of the thread local's value
     * @param supplier the supplier to be used to determine the initial value
     * @return a new thread local variable
     * @throws NullPointerException if the specified supplier is null
     * @since 1.8
     */
    public static <S> ThreadLocal<S> withInitial(Supplier<? extends S> supplier) {
        return new SuppliedThreadLocal<>(supplier);
    }

    /**
     * Creates a thread local variable.
     * @see #withInitial(java.util.function.Supplier)
     */
    public ThreadLocal() {
    }
}
```

看看我们怎么从ThreadLocal中拿对象。

```java
public class ThreadLocal<T> {
    /**
     * Returns the value in the current thread's copy of this
     * thread-local variable.  If the variable has no value for the
     * current thread, it is first initialized to the value returned
     * by an invocation of the {@link #initialValue} method.
     *
     * @return the current thread's value of this thread-local
     */
    public T get() {
        //当前线程
        Thread t = Thread.currentThread();
        //拿关联当前线程的本地变量map
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                //拿到了对象，就直接返回
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        //否则初始化
        return setInitialValue();
    }
}
```

我们先看看这个线程map是个啥。

```java
    /**
     * Get the map associated with a ThreadLocal. Overridden in
     * InheritableThreadLocal.
     *
     * @param  t the current thread
     * @return the map
     */
    ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
    }
```

可以看到实际上是线程的某个成员变量。

```java
public
class Thread implements Runnable {
    /* ThreadLocal values pertaining to this thread. This map is maintained
     * by the ThreadLocal class. */
    ThreadLocal.ThreadLocalMap threadLocals = null;
}
```

可以看到这个成员变量初始值为null，且构造器中也没有进行初始化，那它啥时候初始化呢。实际上它是在ThreadLocal中初始化的，奇怪啊！

```java
public class ThreadLocal<T> {
    /**
     * Create the map associated with a ThreadLocal. Overridden in
     * InheritableThreadLocal.
     *
     * @param t the current thread
     * @param firstValue value for the initial entry of the map
     */
    void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }
}
```

那啥时候我们的createMap会被调用呢，它会在线程第一次访问且线程的threadLocals变量为null的时候被初始化。

```java
public class ThreadLocal<T> {
    /**
     * Variant of set() to establish initialValue. Used instead
     * of set() in case user has overridden the set() method.
     *
     * @return the initial value
     */
     //这个方法就是在get时候被调用的
    private T setInitialValue() {
        T value = initialValue();
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
        return value;
    }

    /**
     * Sets the current thread's copy of this thread-local variable
     * to the specified value.  Most subclasses will have no need to
     * override this method, relying solely on the {@link #initialValue}
     * method to set the values of thread-locals.
     *
     * @param value the value to be stored in the current thread's copy of
     *        this thread-local.
     */
    public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }
}
```

在remove的时候，如果线程没有threadLocals当然就不用初始化了。

```java
public class ThreadLocal<T> {
    /**
     * Removes the current thread's value for this thread-local
     * variable.  If this thread-local variable is subsequently
     * {@linkplain #get read} by the current thread, its value will be
     * reinitialized by invoking its {@link #initialValue} method,
     * unless its value is {@linkplain #set set} by the current thread
     * in the interim.  This may result in multiple invocations of the
     * {@code initialValue} method in the current thread.
     *
     * @since 1.5
     */
     public void remove() {
         ThreadLocalMap m = getMap(Thread.currentThread());
         if (m != null)
             m.remove(this);
     }
}
```

线程的threadLocals变量，这里比较有趣的就是它用的并不是标准库的HashMap，而是自己搞的一个内部类。这玩意可不是HashMap，自然就少了红黑树那套恐怖的代码了，所以我们抱着愉快的心情读下去。

```java
public class ThreadLocal<T> {
    /**
     * ThreadLocalMap is a customized hash map suitable only for
     * maintaining thread local values. No operations are exported
     * outside of the ThreadLocal class. The class is package private to
     * allow declaration of fields in class Thread.  To help deal with
     * very large and long-lived usages, the hash table entries use
     * WeakReferences for keys. However, since reference queues are not
     * used, stale entries are guaranteed to be removed only when
     * the table starts running out of space.
     */
    static class ThreadLocalMap {
    }
}
```

ThreadLocalMap中的Entry也是不同寻常，继承了WeakReference。Entry的关键字一定是ThreadLocal类型的，且弱引用的是这个key。

```java
    static class ThreadLocalMap {
        /**
         * The entries in this hash map extend WeakReference, using
         * its main ref field as the key (which is always a
         * ThreadLocal object).  Note that null keys (i.e. entry.get()
         * == null) mean that the key is no longer referenced, so the
         * entry can be expunged from table.  Such entries are referred to
         * as "stale entries" in the code that follows.
         */
        static class Entry extends WeakReference<ThreadLocal<?>> {
            /** The value associated with this ThreadLocal. */
            Object value;

            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }
  }
```

继续往下读。可以发现还是那经典的味道。

```java
  static class ThreadLocalMap {
        /**
         * The initial capacity -- MUST be a power of two.
         */
         //初始大小
        private static final int INITIAL_CAPACITY = 16;

        /**
         * The table, resized as necessary.
         * table.length MUST always be a power of two.
         */
         //链表头
        private Entry[] table;

        /**
         * The number of entries in the table.
         */
         //当前插了几个元素了
        private int size = 0;

        /**
         * The next size value at which to resize.
         */
         //下次扩展的阈值
        private int threshold; // Default to 0
  
  }
```

它暴露出去的构造器加了奇怪的参数，不太明白为啥这样写。

```java
  static class ThreadLocalMap {
        /**
         * Construct a new map initially containing (firstKey, firstValue).
         * ThreadLocalMaps are constructed lazily, so we only create
         * one when we have at least one entry to put in it.
         */
        ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
            table = new Entry[INITIAL_CAPACITY];
            int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
            table[i] = new Entry(firstKey, firstValue);
            size = 1;
            setThreshold(INITIAL_CAPACITY);
        }
  }
```

看看如何放键值对，好像put改名成set了。

```java
  static class ThreadLocalMap {
        /**
         * Set the value associated with key.
         *
         * @param key the thread local object
         * @param value the value to be set
         */
        private void set(ThreadLocal<?> key, Object value) {

            // We don't use a fast path as with get() because it is at
            // least as common to use set() to create new entries as
            // it is to replace existing ones, in which case, a fast
            // path would fail more often than not.

            Entry[] tab = table;
            int len = tab.length;
            int i = key.threadLocalHashCode & (len-1);

            for (Entry e = tab[i];
                 e != null;
                 e = tab[i = nextIndex(i, len)]) {
                ThreadLocal<?> k = e.get();

                if (k == key) {
                    //找到了，直接更新值
                    e.value = value;
                    return;
                }

                if (k == null) {
                    //如果k过期了，直接就入住这里了
                    replaceStaleEntry(key, value, i);
                    return;
                }
            }

            tab[i] = new Entry(key, value);
            int sz = ++size;
            if (!cleanSomeSlots(i, sz) && sz >= threshold)
                rehash();
        }
  }
```

上面的代码可以发现，`ThreadLocalMap`似乎想和`HashMap`撇清关系，它甚至没有使用HashMap采用的链表法解决冲突，取而代之的是使用了开放寻址法。不过它重新寻址的函数也是有够寒酸的，直接找下一个。

```java
static class ThreadLocalMap {
        /**
         * Increment i modulo len.
         */
        private static int nextIndex(int i, int len) {
            return ((i + 1 < len) ? i + 1 : 0);
        }
}
```

顺便回去看看ThreadLocal中存的`threadLocalHashCode`是个啥。

```java
public class ThreadLocal<T> {
    /**
     * ThreadLocals rely on per-thread linear-probe hash maps attached
     * to each thread (Thread.threadLocals and
     * inheritableThreadLocals).  The ThreadLocal objects act as keys,
     * searched via threadLocalHashCode.  This is a custom hash code
     * (useful only within ThreadLocalMaps) that eliminates collisions
     * in the common case where consecutively constructed ThreadLocals
     * are used by the same threads, while remaining well-behaved in
     * less common cases.
     */
    private final int threadLocalHashCode = nextHashCode();

    /**
     * The next hash code to be given out. Updated atomically. Starts at
     * zero.
     */
    private static AtomicInteger nextHashCode =
        new AtomicInteger();

    /**
     * The difference between successively generated hash codes - turns
     * implicit sequential thread-local IDs into near-optimally spread
     * multiplicative hash values for power-of-two-sized tables.
     */
    private static final int HASH_INCREMENT = 0x61c88647;

    /**
     * Returns the next hash code.
     */
    private static int nextHashCode() {
        return nextHashCode.getAndAdd(HASH_INCREMENT);
    }
}
```

可以看到第$k$个创建的ThreadLocal的哈希值为$(k-1)\times 0x61c88647$。

由于使用了弱引用键，因此这里有一个问题，那就是咋做回收操作。

```java
static class ThreadLocalMap {
        /**
         * Expunge a stale entry by rehashing any possibly colliding entries
         * lying between staleSlot and the next null slot.  This also expunges
         * any other stale entries encountered before the trailing null.  See
         * Knuth, Section 6.4
         *
         * @param staleSlot index of slot known to have null key
         * @return the index of the next null slot after staleSlot
         * (all between staleSlot and this slot will have been checked
         * for expunging).
         */
         //清理某个slot
        private int expungeStaleEntry(int staleSlot) {
            Entry[] tab = table;
            int len = tab.length;

            // expunge entry at staleSlot
            tab[staleSlot].value = null;
            tab[staleSlot] = null;
            size--;

            // Rehash until we encounter null
            Entry e;
            int i;
            //由于是开放寻址法，因此一旦一个键被移除了，后面的所有键都需要被提升，我们循环直到找到第一个null
            //这表示不可能有数被加在尾部了
            for (i = nextIndex(staleSlot, len);
                 (e = tab[i]) != null;
                 i = nextIndex(i, len)) {
                ThreadLocal<?> k = e.get();
                if (k == null) {
                    //过期了，删了
                    e.value = null;
                    tab[i] = null;
                    size--;
                } else {
                  //没过期，我们进行重哈希（它的位置可能在前面，也有可能就是这里不变）
                    int h = k.threadLocalHashCode & (len - 1);
                    if (h != i) {
                        tab[i] = null;

                        // Unlike Knuth 6.4 Algorithm R, we must scan until
                        // null because multiple entries could have been stale.
                        while (tab[h] != null)
                            h = nextIndex(h, len);
                        tab[h] = e;
                    }
                }
            }
            return i;
        }
}
```

里面还有一段神奇的代码，启发式的清理一些过期的key，这里的代码很显然不是完整的清理，感觉这些作者开始放飞自我了。这段代码在插入查找删除等map的基本操作时都会被调用，因此会将所有的基本操作的时间复杂度提高到$O(\log_2len)$，len是数组的长度。

```java
static class ThreadLocalMap {
        private boolean cleanSomeSlots(int i, int n) {
            boolean removed = false;
            Entry[] tab = table;
            int len = tab.length;
            do {
                i = nextIndex(i, len);
                Entry e = tab[i];
                if (e != null && e.get() == null) {
                    n = len;
                    removed = true;
                    i = expungeStaleEntry(i);
                }
            } while ( (n >>>= 1) != 0);
            return removed;
        }
}
```

可以发现如果一个key过期了，是不一定能保证释放的，由于是启发式清理，因此可能有些key永远不会被扫到。

但是实际上作者还是留了一手，就是在重哈希的时候做了全局的清理。

```java
static class ThreadLocalMap {
        /**
         * Re-pack and/or re-size the table. First scan the entire
         * table removing stale entries. If this doesn't sufficiently
         * shrink the size of the table, double the table size.
         */
        private void rehash() {
            expungeStaleEntries();

            // Use lower threshold for doubling to avoid hysteresis
            if (size >= threshold - threshold / 4)
                resize();
        }

        /**
         * Expunge all stale entries in the table.
         */
        private void expungeStaleEntries() {
            Entry[] tab = table;
            int len = tab.length;
            for (int j = 0; j < len; j++) {
                Entry e = tab[j];
                if (e != null && e.get() == null)
                    expungeStaleEntry(j);
            }
        }
}
```

而重哈希仅发生在占用一个新的slot，且启发式清理没有成功，同时哈希表的大小达到阈值的时候。

```java
static class ThreadLocalMap {
        /**
         * Set the value associated with key.
         *
         * @param key the thread local object
         * @param value the value to be set
         */
        private void set(ThreadLocal<?> key, Object value) {
            if (!cleanSomeSlots(i, sz) && sz >= threshold)
                rehash();
        }
}
```

可以看出ThreadLocalMap的代码是比较复杂的，原因是为了处理弱引用。我们来考虑有哪些情况下会发生内存泄露。由于ThreadLocalMap的唯一引用落在Thread对象中，因此只要线程未退出（比如是线程池复用的线程），这时候它的threadLocals变量也不会被清理，导致threadLocals中的所有键值对都会被保留。键值对实现了弱引用，但是弱引用的仅仅是ThreadLocal这个变量，而ThreadLocal对应的值是强引用。因此你即使将ThreadLocal设置为null也不能保证资源会被释放，而是需要调用`ThreadLocal#clear()`来手动清理资源。

# InheritableThreadLocal

如果我们希望子线程能使用父线程的ThreadLocal对象，该怎么办。JDK里面提供了`InheritableThreadLocal`类型。它的代码非常简单

```java
public class InheritableThreadLocal<T> extends ThreadLocal<T> {
    /**
     * Computes the child's initial value for this inheritable thread-local
     * variable as a function of the parent's value at the time the child
     * thread is created.  This method is called from within the parent
     * thread before the child is started.
     * <p>
     * This method merely returns its input argument, and should be overridden
     * if a different behavior is desired.
     *
     * @param parentValue the parent thread's value
     * @return the child thread's initial value
     */
    protected T childValue(T parentValue) {
        return parentValue;
    }

    /**
    * 这里覆盖掉getMap。
    * 由于一个InheritableThreadLocal可能出现在多个ThreadLocalMap*中，
    * 因此我们不能在InheritableThreadLocal变量中存储自己所在的ThreadLocalMap，
    * 必须实时获取。
    */
    ThreadLocalMap getMap(Thread t) {
       return t.inheritableThreadLocals;
    }

    /**
     * Create the map associated with a ThreadLocal.
     *
     * @param t the current thread
     * @param firstValue value for the initial entry of the table.
     */
    void createMap(Thread t, T firstValue) {
        t.inheritableThreadLocals = new ThreadLocalMap(this, firstValue);
    }
}
```

可以发现它使用了线程中的另外一个变量`inheritableThreadLocals`。看看这个变量是怎么初始化和创建的。

```java
public
class Thread implements Runnable {
    private void init(ThreadGroup g, Runnable target, String name,
                      long stackSize, AccessControlContext acc,
                      boolean inheritThreadLocals) {
        ...
        Thread parent = currentThread();
        ...
        if (inheritThreadLocals && parent.inheritableThreadLocals != null)
            this.inheritableThreadLocals =
                ThreadLocal.createInheritedMap(parent.inheritableThreadLocals);

    }
}
```

可以发现线程在初始化的时候，如果发现父线程（负责调用初始化代码的线程）如果用需要被继承的ThreadLocal对象，那么就会创建属于自己的`inheritableThreadLocals`变量并初始化。初始化的代码很简单：

```java
public
class Thread implements Runnable {
    private ThreadLocalMap(ThreadLocalMap parentMap) {
            Entry[] parentTable = parentMap.table;
            int len = parentTable.length;
            setThreshold(len);
            table = new Entry[len];
            //遍历parentMap，并将所有键值对插入到新建的ThreadLocalMap中
            for (int j = 0; j < len; j++) {
                Entry e = parentTable[j];
                if (e != null) {
                    @SuppressWarnings("unchecked")
                    ThreadLocal<Object> key = (ThreadLocal<Object>) e.get();
                    if (key != null) {
                        Object value = key.childValue(e.value);
                        Entry c = new Entry(key, value);
                        int h = key.threadLocalHashCode & (len - 1);
                        while (table[h] != null)
                            h = nextIndex(h, len);
                        table[h] = c;
                        size++;
                    }
                }
            }
        }
}
```

可以发现`inheritableThreadLocals`仅在初始化的时候会复制所有父线程的`inheritableThreadLocals`，之后二者就是独立运行了，向一者的插入删除操作不会影响另外一个线程的变量，但是它们其中保存的`InheritableThreadLocal`变量是相同的。