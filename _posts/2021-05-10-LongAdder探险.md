---
categories: techonology
layout: post
---

- Table
{:toc}

# 说明

本篇代码讲的是java 1.8.0_191中的代码。

# 源码

`LongAdder`是并发包下的一个存储长整形的一个容器，你可以并发的增加其中存储长整形的值。在`ConcurrentHashMap`中，就是用`LongAdder`的技术来维护其`size`属性的。简单来说，在存在竞争的情况下，`LongAdder`的性能要比`AtomicLong`要好。

其原理实际上就是维护多个计数器来进行计数，并且计数器的数量会随着竞争的发生，而动态扩容。而获取内部值的时候，则是需要加总所有计数器的和。

## Cell

Cell实现的就是上面提到的计数器。

```java
    @sun.misc.Contended static final class Cell {
        //保证可见性
        volatile long value;
        Cell(long x) { value = x; }
        final boolean cas(long cmp, long val) {
            //通过cas操作来交换值
            return UNSAFE.compareAndSwapLong(this, valueOffset, cmp, val);
        }

        // Unsafe mechanics
        private static final sun.misc.Unsafe UNSAFE;
        private static final long valueOffset;
        static {
            try {
                //初始化偏移量
                UNSAFE = sun.misc.Unsafe.getUnsafe();
                Class<?> ak = Cell.class;
                valueOffset = UNSAFE.objectFieldOffset
                    (ak.getDeclaredField("value"));
            } catch (Exception e) {
                throw new Error(e);
            }
        }
    }
```

## LongAdder

下面看一下LongAdder代码。

```java
public class LongAdder extends Striped64 implements Serializable {
    //扩容或创建Cells的时候作为自旋锁使用
    transient volatile int cellsBusy;
    //多个计数器，初始为null，非null的时候大小为二的幂次
    transient volatile Cell[] cells;
    //无竞争的情况下，使用base计数
    transient volatile long base;
}
```

先看一下如何统计内部计数值。

```java
public class LongAdder extends Striped64 implements Serializable {
    public long sum() {
        //遍历所有的cell，加总和就是结果了
        Cell[] as = cells; Cell a;
        long sum = base;
        if (as != null) {
            for (int i = 0; i < as.length; ++i) {
                if ((a = as[i]) != null)
                    sum += a.value;
            }
        }
        return sum;
    }
}
```

下面看一下如何给内部计数增加一个固定值。

```java
public class LongAdder extends Striped64 implements Serializable {
    //转发给add通用方法
    public void increment() {
        add(1L);
    }

    public void decrement() {
        add(-1L);
    }

    public void add(long x) {
        Cell[] as; long b, v; int m; Cell a;
        //如果cells未创建，且base被正确修改，那么流程结束，否则需要通过cells来计数
        if ((as = cells) != null || !casBase(b = base, b + x)) {
            //不使用base计数的情况
            //uncontended表示是否存在竞争
            boolean uncontended = true;
            //如果cells未初始化，或者长度为0，或者争夺的cell未创建，或者竞争失败，都需要转发给longAccumulate来处理。
            //如果竞争成功，则结束
            if (as == null || (m = as.length - 1) < 0 ||
                //这里通过getProbe来获取当前线程对应的随机数，来分散竞争压力
                (a = as[getProbe() & m]) == null ||
                !(uncontended = a.cas(v = a.value, v + x)))
                longAccumulate(x, null, uncontended);
        }
    }

    //getProbe方法返回当前线程绑定的probe属性，其实质上是一个随机数。
    static final int getProbe() {
        return UNSAFE.getInt(Thread.currentThread(), PROBE);
    }
}
```

可以发现代码转发到了`longAccumulate`方法上，具体看一下。

```java
    final void longAccumulate(long x, LongBinaryOperator fn,
                              boolean wasUncontended) {
        //如果线程随机数未初始化，这里会通过调用ThreadLocalRandom.current()来强制初始化
        int h;
        if ((h = getProbe()) == 0) {
            ThreadLocalRandom.current(); // force initialization
            h = getProbe();
            //可以认为上一次竞争之所以失败，是因为线程随机数未初始化
            //这里重新设置为无竞争并重试
            wasUncontended = true;
        }
        //上一个用到的cell是否存在的标记
        boolean collide = false;                // True if last slot nonempty
        for (;;) {
            Cell[] as; Cell a; int n; long v;
            //cells已经创建且非空
            if ((as = cells) != null && (n = as.length) > 0) {
                //如果要用到的cell为空（未创建）
                if ((a = as[(n - 1) & h]) == null) {
                    //如果自旋锁可用
                    if (cellsBusy == 0) {       // Try to attach new Cell
                        //创建cell并且顺带把计数的任务完成
                        Cell r = new Cell(x);   // Optimistically create
                        //自旋锁可用，且抢锁成功
                        if (cellsBusy == 0 && casCellsBusy()) {
                            //是否创建cell的标记
                            boolean created = false;
                            try {               // Recheck under lock
                                //拿到锁后二次检查，防止局部比变量是脏数据
                                Cell[] rs; int m, j;
                                //cell依旧为未创建的状态
                                if ((rs = cells) != null &&
                                    (m = rs.length) > 0 &&
                                    rs[j = (m - 1) & h] == null) {
                                    //将cell放入cells数组
                                    rs[j] = r;
                                    //标记创建成功
                                    created = true;
                                }
                            } finally {
                                //释放锁
                                cellsBusy = 0;
                            }
                            //创建成功的话，由于顺带完成了增加计数的任务，因此可以直接退出了
                            if (created)
                                break;
                            continue;           // Slot is now non-empty
                        }
                    }
                    //标记上一次用到的cell还不存在
                    collide = false;
                }
                //如果之前失败原因是竞争存在
                else if (!wasUncontended)       // CAS already known to fail
                    //设置无竞争标记
                    wasUncontended = true;      // Continue after rehash
                //尝试更改cell，增加计数值
                else if (a.cas(v = a.value, ((fn == null) ? v + x :
                                             fn.applyAsLong(v, x))))
                    //成功，大功告成，退出即可
                    break;
                //如果数组长度已经达到了CPU核心数
                //或者cells刚刚被别的线程扩容过了，那么没必要扩展了
                else if (n >= NCPU || cells != as)
                    //这个分支表示，一但数组长度达到CPU核心数，就不再扩容了
                    //标记上一个用到的cell不存在
                    collide = false;            // At max size or stale
                //如果上一个用到的cell不存在，则标记为存在
                else if (!collide)
                    collide = true;
                //加锁，准备扩容
                else if (cellsBusy == 0 && casCellsBusy()) {
                    try {
                        //二次检查，放置局部变量脏数据
                        if (cells == as) {      // Expand table unless stale
                            //扩容为两倍大小，同样也是二的幂次
                            Cell[] rs = new Cell[n << 1];
                            //扩容后拷贝一下原先的cell
                            for (int i = 0; i < n; ++i)
                                rs[i] = as[i];
                            cells = rs;
                        }
                    } finally {
                        //释放锁
                        cellsBusy = 0;
                    }
                    //这时候线程随机数的哈希值可能会改变，所以不能断定slot非空
                    collide = false;
                    continue;                   // Retry with expanded table
                }

                //最终修改线程随机数为下一个随机数
                h = advanceProbe(h);
            }
            //目前自旋锁可用，尝试获取锁后初始化cells
            else if (cellsBusy == 0 && cells == as && casCellsBusy()) {
                boolean init = false;
                try {                           // Initialize table
                    //发现cells没有被替换过，这时候说明as不是脏数据，可以直接使用
                    if (cells == as) {
                        //第一次创建的cells大小为2，并顺带把将要用到的cell一同初始化掉
                        Cell[] rs = new Cell[2];
                        rs[h & 1] = new Cell(x);
                        cells = rs;
                        //标记初始化成功
                        init = true;
                    }
                } finally {
                    //释放锁
                    cellsBusy = 0;
                }
                //如果初始化成功，这意味着计数的任务也顺带一同完成了，可以退出了
                if (init)
                    break;
            }
            //通过cells计数方案失败，尝试能不能通过base进行计数
            else if (casBase(v = base, ((fn == null) ? v + x :
                                        fn.applyAsLong(v, x))))
                break;                          // Fall back on using base
        }
    }
```

总结一下流程就是。

- 如果cells未创建，则首先会通过创建大小为2的cells，同时顺带计数，流程结束。（如果抢锁失败，则会直接cas增加base变量）
- 如果线程散列到的cell未创建，则创建cell，同时顺带计数，流程结束
- 否则如果cells大小小于CPU核心数，则对cells进行扩容，扩容为原先的两倍大小。进行下一步。
- 重新计算线程随机数，再重头跑流程。

上面所有提到的创建Cell和初始化cells，以及扩容操作，都是通过对cellsBusy这个变量的CAS操作（加锁解锁）来实现的并发控制。