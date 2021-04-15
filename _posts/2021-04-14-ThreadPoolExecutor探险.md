---
categories: techonology
layout: post
---

- Table
{:toc}

# 源码

`ThreadPoolExecutor`中有若干个构造器，它们都转发到一个完整的构造器中（相当于提供一些默认参数）。

完整的构造器需要下面参数：

- corePoolSize：表示线程池稳定时需要多少个线程（核心线程）
- maximumPoolSize：表示最多允许创建多少个线程
- keepAliveTime：如果线程池中存活线程数超过稳定线程数，那么多出的线程最多只能空闲keepAliveTime时间
- unit：keepAliveTime的单位
- workQueue：提交的自定义阻塞队列，作为存放暂时无力执行的任务的缓冲区
- threadFactory：允许你按照自己的要求创建线程池中的线程
- handler：负责处理那些即无法被处理，同时也无法被加入阻塞队列的任务。

```java
public class ThreadPoolExecutor extends AbstractExecutorService {
    public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue) {
        this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
             Executors.defaultThreadFactory(), defaultHandler);
    }
    public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory) {
        this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
             threadFactory, defaultHandler);
    }
    public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              RejectedExecutionHandler handler) {
        this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
             Executors.defaultThreadFactory(), handler);
    }

    public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler) {
        if (corePoolSize < 0 ||
            maximumPoolSize <= 0 ||
            maximumPoolSize < corePoolSize ||
            keepAliveTime < 0)
            throw new IllegalArgumentException();
        if (workQueue == null || threadFactory == null || handler == null)
            throw new NullPointerException();
        this.acc = System.getSecurityManager() == null ?
                null :
                AccessController.getContext();
        this.corePoolSize = corePoolSize;
        this.maximumPoolSize = maximumPoolSize;
        this.workQueue = workQueue;
        this.keepAliveTime = unit.toNanos(keepAliveTime);
        this.threadFactory = threadFactory;
        this.handler = handler;
    }
}
```

ThreadPoolExecutor中使用一些特殊的掩码来表示线程池当前的状态：

```java
public class ThreadPoolExecutor extends AbstractExecutorService {
    //ctl表示的是当前线程池的运行状态。
    //其中第31位表示线程池正常工作，否则线程池处于停止中或已经停止。
    //第29，30位表示当前线程池的状态
    //后面的29位用来记录当前存活的线程数
    private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
    private static final int COUNT_BITS = Integer.SIZE - 3;
    private static final int CAPACITY   = (1 << COUNT_BITS) - 1;

    // runState is stored in the high-order bits
    private static final int RUNNING    = -1 << COUNT_BITS;
    private static final int SHUTDOWN   =  0 << COUNT_BITS;
    private static final int STOP       =  1 << COUNT_BITS;
    private static final int TIDYING    =  2 << COUNT_BITS;
    private static final int TERMINATED =  3 << COUNT_BITS;

    // Packing and unpacking ctl
    private static int runStateOf(int c)     { return c & ~CAPACITY; }
    private static int workerCountOf(int c)  { return c & CAPACITY; }
    private static int ctlOf(int rs, int wc) { return rs | wc; }

    /*
     * Bit field accessors that don't require unpacking ctl.
     * These depend on the bit layout and on workerCount being never negative.
     */

    private static boolean runStateLessThan(int c, int s) {
        return c < s;
    }

    private static boolean runStateAtLeast(int c, int s) {
        return c >= s;
    }

    private static boolean isRunning(int c) {
        return c < SHUTDOWN;
    }
}
```

线程池的状态分为：

- RUNNING：接受新任务，并且可以处理缓冲区中的任务
- SHUTDOWN：不接受新任务，但是可以处理缓冲区中的任务
- STOP：不接受新任务，也不处理缓冲区中的任务，并且会中断正在执行的任务
- TIDYING：所有任务都终止了，工作线程数减少到0，这时候会调用`terminated()`方法进行后置操作，之后状态会切换到TERMINATED
- TERMINATED：真的结束了

可以发现状态只会从较小的数值转移到较大的数值。

来看看具体如何提交任务的。

```java
public class ThreadPoolExecutor extends AbstractExecutorService {
    public class ThreadPoolExecutor extends AbstractExecutorService {
    public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
         //这里分三种情况处理
         //1. 如果当前线程数少于稳定线程数，创建新的线程驱动任务
         //2. 如果缓冲区还有空间，就加入到缓冲区
         //3. 如果线程数少于最大线程数，创建新的线程驱动任务
        int c = ctl.get();
        if (workerCountOf(c) < corePoolSize) {
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
        if (isRunning(c) && workQueue.offer(command)) {
            //这里要处理另外一个线程尝试停止线程池的情况。
            //这时候我们可以试着从缓冲区中删除刚刚加入的任务
            int recheck = ctl.get();
            if (! isRunning(recheck) && remove(command))
                //成功了，好消息
                reject(command);
            else if (workerCountOf(recheck) == 0)
                //失败了，只能创建新的一个线程把刚刚提交的任务干掉了
                addWorker(null, false);
        }
        else if (!addWorker(command, false))
            //真的不行了
            reject(command);
    }

    void reject(Runnable command) {
        handler.rejectedExecution(command, this);
    }

    public boolean remove(Runnable task) {
        boolean removed = workQueue.remove(task);
        tryTerminate(); // In case SHUTDOWN and now empty
        return removed;
    }
}
```

先不关注线程池关闭的情况，看看`addWorker`具体的操作。

```java
public class ThreadPoolExecutor extends AbstractExecutorService {
    private boolean addWorker(Runnable firstTask, boolean core) {
        retry:
        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            //如果状态为SHUTDOWN或更高，就应该不处理
            //但是有一种情况是，缓冲区无法删除新提交的任务，这时候必须建一个新的线程来处理
            if (rs >= SHUTDOWN &&
                ! (rs == SHUTDOWN &&
                   firstTask == null &&
                   ! workQueue.isEmpty()))
                return false;

            for (;;) {
                int wc = workerCountOf(c);
                //如果线程数不能再增加了，就报告失败
                if (wc >= CAPACITY ||
                    wc >= (core ? corePoolSize : maximumPoolSize))
                    return false;
                //成功增加了一个工作线程名额，就跳出两层循环
                if (compareAndIncrementWorkerCount(c))
                    break retry;
                //如果线程池状态改变了，就跳出一层循环
                c = ctl.get();  // Re-read ctl
                if (runStateOf(c) != rs)
                    continue retry;
                // else CAS failed due to workerCount change; retry inner loop
            }
        }

        boolean workerStarted = false;
        boolean workerAdded = false;
        Worker w = null;
        try {
            //新建一个Worker
            w = new Worker(firstTask);
            final Thread t = w.thread;
            if (t != null) {
                final ReentrantLock mainLock = this.mainLock;
                mainLock.lock();
                try {
                    //这里要再次检查
                    int rs = runStateOf(ctl.get());

                    if (rs < SHUTDOWN ||
                        (rs == SHUTDOWN && firstTask == null)) {
                        //这里不允许线程提前启动
                        if (t.isAlive()) // precheck that t is startable
                            throw new IllegalThreadStateException();
                        //记录worker
                        workers.add(w);
                        int s = workers.size();
                        //记录最大的线程数目
                        if (s > largestPoolSize)
                            largestPoolSize = s;
                        workerAdded = true;
                    }
                } finally {
                    mainLock.unlock();
                }
                if (workerAdded) {
                    //如果成功加入线程，才允许启动
                    t.start();
                    workerStarted = true;
                }
            }
        } finally {
            //启动失败处理
            if (! workerStarted)
                addWorkerFailed(w);
        }
        return workerStarted;
    }

    private void addWorkerFailed(Worker w) {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            //既然启动失败，就移除worker，减少worker的数目
            if (w != null)
                workers.remove(w);
            decrementWorkerCount();
            //这时候可能满足了TIDYING的要求，尝试停止线程池
            tryTerminate();
        } finally {
            mainLock.unlock();
        }
    }
}
```

线程中创建的线程并不是真的用来执行你提交的Task的，而是会执行一个内置的循环任务，每次执行手头的工作，或者从缓冲区中取下一个任务。

```java
public class ThreadPoolExecutor extends AbstractExecutorService {
    final void runWorker(Worker w) {
        Thread wt = Thread.currentThread();
        //w.firstTask是创建线程是绑定的任务（线程是惰性创建的）
        Runnable task = w.firstTask;
        w.firstTask = null;
        //这里unlock会将worker的state重置为0，只有state为0的worker允许打断
        w.unlock(); // allow interrupts
        //completedAbruptly表示线程是否因为异常而退出。
        //比如你在beforeExecute或afterExecute中抛出异常
        //或者在Runnable任务中抛出运行时异常
        boolean completedAbruptly = true;
        try {
            //这里会一直循环到无任务可做为止
            while (task != null || (task = getTask()) != null) {
                //只有取到任务才上锁，保证线程池有机会抢到锁，并中断线程
                w.lock();
                // If pool is stopping, ensure thread is interrupted;
                // if not, ensure thread is not interrupted.  This
                // requires a recheck in second case to deal with
                // shutdownNow race while clearing interrupt
                //如果处于STOP状态，就中断当前线程
                if ((runStateAtLeast(ctl.get(), STOP) ||
                     (Thread.interrupted() &&
                      runStateAtLeast(ctl.get(), STOP))) &&
                    !wt.isInterrupted())
                    wt.interrupt();
                try {
                    //做些前置操作，默认啥也不做
                    beforeExecute(wt, task);
                    Throwable thrown = null;
                    try {
                        //驱动任务，并捕获异常
                        //这里相当于异常可以抛，但是线程不能死
                        task.run();
                    } catch (RuntimeException x) {
                        thrown = x; throw x;
                    } catch (Error x) {
                        thrown = x; throw x;
                    } catch (Throwable x) {
                        thrown = x; throw new Error(x);
                    } finally {
                        //做一些后置操作
                        afterExecute(task, thrown);
                    }
                } finally {
                    //这里解锁，允许竞争
                    task = null;
                    w.completedTasks++;
                    w.unlock();
                }
            }
            completedAbruptly = false;
        } finally {
            processWorkerExit(w, completedAbruptly);
        }
    }
}
```

默认`beforeExecute`和`afterExecute`允许我们在线程驱动任务的前后，做一些操作，默认行为都是留白。

`processWorkerExit`用于处理线程的结束事件，这时候需要更新线程池中的`ctl`变量。

```java
public class ThreadPoolExecutor extends AbstractExecutorService {
    private void processWorkerExit(Worker w, boolean completedAbruptly) {
        //异常退出的线程在这里减少工作线程数
        if (completedAbruptly) // If abrupt, then workerCount wasn't adjusted
            decrementWorkerCount();

        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            completedTaskCount += w.completedTasks;
            workers.remove(w);
        } finally {
            mainLock.unlock();
        }

        tryTerminate();

        int c = ctl.get();
        //还没有到STOP状态，积压的任务依旧需要被考虑
        if (runStateLessThan(c, STOP)) {
            if (!completedAbruptly) {
                //特殊处理非异常退出的情况
                //设置必要的线程处理缓冲区堆积的任务
                //这里如果允许核心线程超时退出，那么最少的核心线程数目应该为0
                int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
                //有积压的任务
                if (min == 0 && ! workQueue.isEmpty())
                    min = 1;
                //如果现在的线程数目足够，就不新加线程
                if (workerCountOf(c) >= min)
                    return; // replacement not needed
            }

            //异常退出或核心线程不足的时候需要新加一个新的线程
            addWorker(null, false);
        }
    }
}
```

由于创建线程的价格很昂贵（分配栈空间，线程的启动过程等等），因此最好不要通过运行时异常结束Runnable，因为这会导致线程池中线程的销毁和新建，这时候线程池的线程复用的优势就会消失。

我们要结束线程池的时候会调用`shutdown`，或者`shutdown`。前者会处理完所有堆积在缓冲区和手头的任务，但是不会在接受新的任务提交了（对应SHUTDOWN状态），后者则会丢弃堆积的任务，并且尝试停止手头上的任务（对应STOP状态）。

```java
public class ThreadPoolExecutor extends AbstractExecutorService {
    public void shutdown() {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            checkShutdownAccess();
            //将状态转移到SHUTDOWN
            advanceRunState(SHUTDOWN);
            //由于不会再执行新的提交任务了，所以可以停止那些多余的空闲线程
            interruptIdleWorkers();
            onShutdown(); // hook for ScheduledThreadPoolExecutor
        } finally {
            mainLock.unlock();
        }
        tryTerminate();
    }

    //一个钩子，在ThreadPoolExecutor类中
    void onShutdown() {
    }

    
    public List<Runnable> shutdownNow() {
        //tasks中存储缓冲区中积压的将被丢弃的任务
        List<Runnable> tasks;
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            checkShutdownAccess();
            //将状态转移到STOP
            advanceRunState(STOP);
            //这里需要中断所有的工作线程
            interruptWorkers();
            tasks = drainQueue();
        } finally {
            mainLock.unlock();
        }
        tryTerminate();
        return tasks;
    }
}
```

中断工作线程的代码如下：

```java
public class ThreadPoolExecutor extends AbstractExecutorService {
    private void interruptWorkers() {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            //全部调用Worker的interruptIfStarted方法
            for (Worker w : workers)
                w.interruptIfStarted();
        } finally {
            mainLock.unlock();
        }
    }
}
```

可以发现主要逻辑处于`Worker#interruptIfStarted`中。

```java
public class ThreadPoolExecutor extends AbstractExecutorService {
    private final class Worker
        extends AbstractQueuedSynchronizer
        implements Runnable {
        Worker(Runnable firstTask) {
            setState(-1); // inhibit interrupts until runWorker
            this.firstTask = firstTask;
            this.thread = getThreadFactory().newThread(this);
        }

        protected boolean tryRelease(int unused) {
            setExclusiveOwnerThread(null);
            setState(0);
            return true;
        }

        void interruptIfStarted() {
            Thread t;
            //只有worker正式启动后才允许打断
            if (getState() >= 0 && (t = thread) != null && !t.isInterrupted()) {
                try {
                    t.interrupt();
                } catch (SecurityException ignore) {
                }
            }
        }
    }
}
```

上面介绍的代码中多次出现了`tryTerminate`，看看它到底是个啥。

```java
public class ThreadPoolExecutor extends AbstractExecutorService {
    final void tryTerminate() {
        for (;;) {
            int c = ctl.get();
            //只有状态为SHUTDOWN或者STOP才有意义
            //如果是SHUTDOWN还必须等缓冲区清空后才能执行下一步
            if (isRunning(c) ||
                runStateAtLeast(c, TIDYING) ||
                (runStateOf(c) == SHUTDOWN && ! workQueue.isEmpty()))
                return;
            
            //还有工作线程存活，ok，试着把没活干的空闲线程结束掉
            if (workerCountOf(c) != 0) { // Eligible to terminate
                interruptIdleWorkers(ONLY_ONE);
                return;
            }

            final ReentrantLock mainLock = this.mainLock;
            mainLock.lock();
            try {
                //尝试切换到TIDYING并执行最后的清理操作
                if (ctl.compareAndSet(c, ctlOf(TIDYING, 0))) {
                    try {
                        //执行结束前的前置事件
                        terminated();
                    } finally {
                        //最后转移到TERMINATED状态
                        ctl.set(ctlOf(TERMINATED, 0));
                        termination.signalAll();
                    }
                    return;
                }
            } finally {
                mainLock.unlock();
            }
            // else retry on failed CAS
        }
    }

    protected void terminated() { }
}
```

大概上面就是线程池的总体流程了。

`ThreadPoolExecutor`的父类`AbstractExecutorService`中存在一些方法的默认实现，我们也看看好了。比如说`Callable`类型的提交。

```java
public abstract class AbstractExecutorService implements ExecutorService {
    public <T> Future<T> submit(Callable<T> task) {
        if (task == null) throw new NullPointerException();
        RunnableFuture<T> ftask = newTaskFor(task);
        execute(ftask);
        return ftask;
    }
    protected <T> RunnableFuture<T> newTaskFor(Callable<T> callable) {
        return new FutureTask<T>(callable);
    }
}
```

让我们看看`FutureTask`又是何方神圣。

```java
//同时实现了Runnable和Future<V>接口，其实际上是一个适配器
public class FutureTask<V> implements RunnableFuture<V> {
    public void run() {
        //将执行线程从null切换为当前线程
        //这意味着FutureTask不会被并发执行
        if (state != NEW ||
            !UNSAFE.compareAndSwapObject(this, runnerOffset,
                                         null, Thread.currentThread()))
            return;
        try {
            Callable<V> c = callable;
            if (c != null && state == NEW) {
                V result;
                //ran记录是否成功处理了任务
                boolean ran;
                try {
                    result = c.call();
                    ran = true;
                } catch (Throwable ex) {
                    result = null;
                    ran = false;
                    setException(ex);
                }
                if (ran)
                    //记录结果
                    set(result);
            }
        } finally {
            runner = null;
            // state must be re-read after nulling runner to prevent
            // leaked interrupts
            int s = state;
            if (s >= INTERRUPTING)
                handlePossibleCancellationInterrupt(s);
        }
    }
}
```

看看怎么记录最终结果的。

```java
public class FutureTask<V> implements RunnableFuture<V> {
    private void finishCompletion() {
        // assert state > COMPLETING;
        //这里会遍历整个等待结果的线程组成的链表
        for (WaitNode q; (q = waiters) != null;) {
            if (UNSAFE.compareAndSwapObject(this, waitersOffset, q, null)) {
                for (;;) {
                    Thread t = q.thread;
                    if (t != null) {
                        q.thread = null;
                        //恢复执行
                        LockSupport.unpark(t);
                    }
                    WaitNode next = q.next;
                    if (next == null)
                        break;
                    q.next = null; // unlink to help gc
                    q = next;
                }
                break;
            }
        }

        done();

        //这里会释放callable，因此这意味着FutureTask是不能被重复执行的
        callable = null;        // to reduce footprint
    }

    //一个钩子
    protected void done() { }
}
```