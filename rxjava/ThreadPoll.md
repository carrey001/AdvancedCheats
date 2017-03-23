## Java线程池
---

**线程池的作用** 就是限制系统中执行线程的数量。 根据系统的环境情况，可以自动或者手动设置线程数量达到运行的最佳效果；减少资源浪费。用线程池控制线程数量，在其他线程排队等候是， 一个任务执行完毕再从队列中取最前面的任务开始执行。若队列中没有等待进程、线程池的这一资源处于等待。 当一个新任务需要运行的时候。如果线程池没有等待的工作线程就可以开始运行了，否则进入等待队列。

**为什么要使用线程池**

-  减少创建和销毁线程的次数。每个工作线程都可以被重复利用。执行多个任务
- 可以根据系统的承受能力，调整线程池中的工作线程的数目，防止消耗过多内存

**线程池中的几个类的关系**
--
类 | 说明
--|--
Executor | 真正的线程池接口顶级接口，只有一个方法execute（Runnable)
ExecutorService|次顶级接口主要声明了一些线程池的操作的方法（包括唤醒、关闭、提交等）
ScheduledExecutorService|能和Timer/TimerTask类似，解决那些需要任务重复执行的问题。
ThreadPoolExecutor|ExecutorService的默认实现
ScheduledThreadPoolExecutor|继承ThreadPoolExecutor的ScheduledExecutorService接口实现，周期性任务调度的类实现

要配置一个线程池是比较复杂的、尤其是对线程池原理不是很理解的情况下。所以在Executors类里面提供了一下静态工厂，生成一下常用的线程池。
  - newSingleThreadExecutor ：创建一个单线程池。该线程池只有一个线程在工作。
  - newFixedThreadPool ： 创建一个固定大小的线程池。每次提交一个任务就创建一个线程。直到达到线程池的最大限制。
  - newCachedThreadPool ： 创建一个可缓存的线程池，如果线程池的大小超过了处理任务需要的线程。就会回收部分空闲线程。当任务数增加时，此线程池又能智能添加新线程来处理任务。此线程池不会对线程池大小做限制，线程池大小完全依赖于操作系统（或者说JVM）能够创建的最大线程大小。
  - newScheduledThreadPool ：创建一个大小无限的线程池。此线程池支持定时以及周期性执行任务的需求。

## ThreadPoolExecutor详解

ThreadPoolExecutor的完整构造函数方法签名是

```
/**
   * Creates a new {@code ThreadPoolExecutor} with the given initial
   * parameters.
   *
   * @param corePoolSize the number of threads to keep in the pool, even
   *        if they are idle, unless {@code allowCoreThreadTimeOut} is set
   * @param maximumPoolSize the maximum number of threads to allow in the
   *        pool
   * @param keepAliveTime when the number of threads is greater than
   *        the core, this is the maximum time that excess idle threads
   *        will wait for new tasks before terminating.
   * @param unit the time unit for the {@code keepAliveTime} argument
   * @param workQueue the queue to use for holding tasks before they are
   *        executed.  This queue will hold only the {@code Runnable}
   *        tasks submitted by the {@code execute} method.
   * @param threadFactory the factory to use when the executor
   *        creates a new thread
   * @param handler the handler to use when execution is blocked
   *        because the thread bounds and queue capacities are reached
   * @throws IllegalArgumentException if one of the following holds:<br>
   *         {@code corePoolSize < 0}<br>
   *         {@code keepAliveTime < 0}<br>
   *         {@code maximumPoolSize <= 0}<br>
   *         {@code maximumPoolSize < corePoolSize}
   * @throws NullPointerException if {@code workQueue}
   *         or {@code threadFactory} or {@code handler} is null
   */
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
      this.corePoolSize = corePoolSize;
      this.maximumPoolSize = maximumPoolSize;
      this.workQueue = workQueue;
      this.keepAliveTime = unit.toNanos(keepAliveTime);
      this.threadFactory = threadFactory;
      this.handler = handler;
  }

```

- corePoolSize:池中所保存的线程数，包括空闲线程
- maximumPoolSize:池中允许的最大线程数
- keepAliveTime: 当线程数大于核心线程数时，此为终止前多余的空闲线程等待新任务的最长时间。   
- unit :keepAliveTime参数的时间单位
- workQueue :执行前用于保持任务的队列。该队列仅保持由execute方法提交的Runnable任务
- threadFactory :执行程序创建新线程时使用的工厂
- handler ： 由于超出线程范围和队列容量而执行被阻塞时所使用的处理程序。

## 解析Executors静态工厂创建的几种线程池的源码

ExecutorService  newFixedThreadPool (int nThreads):固定大小线程池
ExecutorService newFixedThreadPool(int nThreads, ThreadFactory threadFactory) ：固定大小线程池自己创建线程工厂

```
/**
    * Creates a thread pool that reuses a fixed number of threads
    * operating off a shared unbounded queue, using the provided
    * ThreadFactory to create new threads when needed.  At any point,
    * at most {@code nThreads} threads will be active processing
    * tasks.  If additional tasks are submitted when all threads are
    * active, they will wait in the queue until a thread is
    * available.  If any thread terminates due to a failure during
    * execution prior to shutdown, a new one will take its place if
    * needed to execute subsequent tasks.  The threads in the pool will
    * exist until it is explicitly {@link ExecutorService#shutdown
    * shutdown}.
    *
    * @param nThreads the number of threads in the pool
    * @param threadFactory the factory to use when creating new threads
    * @return the newly created thread pool
    * @throws NullPointerException if threadFactory is null
    * @throws IllegalArgumentException if {@code nThreads <= 0}
    */
   public static ExecutorService newFixedThreadPool(int nThreads, ThreadFactory threadFactory) {
       return new ThreadPoolExecutor(nThreads, nThreads,
                                     0L, TimeUnit.MILLISECONDS,
                                     new LinkedBlockingQueue<Runnable>(),
                                     threadFactory);
   }
```
从构造方法我们可以看到，corePoolSize 和maximumPoolSize 的大小是一样的，
keepAliveTime 设置的时间是0.BlockingQueue 选择的是LinkedBlockingQueue 这个队列的特点是无界。


ExecutorService  newSingleThreadExecutor()：单线程
```
public static ExecutorService newSingleThreadExecutor() {
      return new FinalizableDelegatedExecutorService
          (new ThreadPoolExecutor(1, 1,
                                  0L, TimeUnit.MILLISECONDS,
                                  new LinkedBlockingQueue<Runnable>()));
  }
```
    corePoolSize 和maximumPoolSize 都是1

ExecutorService newCachedThreadPool()：无界线程池，可以进行自动线程回收
```
public static ExecutorService newCachedThreadPool() {
       return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                     60L, TimeUnit.SECONDS,
                                     new SynchronousQueue<Runnable>());
   }

```
    maximumPoolSize 是big其次BlockingQueue 的选择是SynchronousQueue。SynchronousQueue 这个队列我们看类注释就可以知道，每个插入操作必须等待另一个线程的对应移除操作
```
/**
 * A {@linkplain BlockingQueue blocking queue} in which each insert
 * operation must wait for a corresponding remove operation by another
 * thread, and vice versa.

```

**BlockingQueue<Runnable> workQueue** ： 这个参数的描述可以知道，它的作用是用于传输和保持提交的任务，可以使用此队列和线程池大小进行交互。
如果运行的线程少于corePoolSize 则Executor始终首选添加新的线程。而不进行排队。**（如果当前运行的线程小于corePoolSize，则任务根本不会存放、添加到queue中，而是直接运行）**。
如果运行的线程大于等于corePoolSize，则Executor始终将请求加入队列，而不添加新的线程。
 如果无法加入队列，则创建新的线程，但是在超出maximumPoolSize的情况下，该任务会被拒绝。

queue 排队的三种通用策略：
- 直接提价：工作队列默认选项是SynchronousQueue,它将任务直接提交给线程而不是保持它们。
- 无界队列：使用无界队列（通常是LinkedBlockingQueue）将导致所有的核心线程都忙时，新任务在队列中等待。这样创建的线程就不会超过corePoolSize。（maximumPoolSize 相当于无限大） 任务执行互不影响
- 有界队列：使用有限队列（ArrayBlockingQueue）有助于防止资源耗尽。但是可能较难调账和控制队列大小和最大池的大小坑你需要相互折中。使用大型队列和小型线程池可以最大度的降低cpu使用率等资源开销。但是可能导致吞吐量降低。反之会增大资源开销 也会降低吞吐量。
---

线程池分为四个组成部分
* 线程池管理者 ThreadPoolExecutor
* 工作线程 Runnable
* 任务接口 Runnable中的
* 任务队列 BlockingQueue
