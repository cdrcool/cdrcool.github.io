---
title: 并发工具类--Executor框架
date: 2019-03-06 23:07:26
categories: Java并发
---
在Java类库中，任务执行的主要抽象不是`Thread`，而是**Executor**，虽然它是个接口，却为灵活且强大的异步任务执行框架提供了基础。
## 两级调度模型
在上层，Java多线程程序通常把应用分解为若干个任务，然后使用用户级的调度器（Executor框架）将这些任务映射为固定数量的线程；在底层，操作系统内核将这些线程映射到处理器上。这种两级调度模型的示意图如下图所示：
![Executor两级调度模型](/images/java/Executor两级调度模型.png)
从图中可以看出，应用程序通过Executor框架控制上层的调度；而下层的调度由操作系统内核控制，下层的调度不受应用程序的控制。
Executor基于生产者-消费者模式，提交任务的操作相当于生产者（生成待完成工作单元），执行任务的线程则相当于消费者（执行这些工作单元）。如果要在程序中实现生产者-消费者的设计，那么最简单的方式通常就是使用Executor。

## Executor框架类结构图
![Executor框架类结构图](/images/java/Executor框架类结构图.png)
由上图可知，Java线程池正是Executor框架的核心成员，应用程序向线程池提交任务，线程池内部执行任务。

## ExecutorService类图
![ExecutorService类图](/images/java/ExecutorService类图.png)
如上图，`Executor`接口只有一个`execute`方法，表示执行任务，`ExecutorService`在其基础上扩展了以下方法：
```java
/**
 * 提交任务并返回一个Future对象
 * 一旦任务执行成功，调用Futrue的get方法将会任务执行结果
  * 
 * @param task 提交的任务
 * @return a Future representing pending completion of the task
 */
<T> Future<T> submit(Callable<T> task);

/**
 * 提交任务并返回一个Future对象
 * 一旦任务执行成功，调用Futrue的get方法将会返回给定result
 * 
 * @param task 提交的任务
 * @param result 返回值
 * @return a Future representing pending completion of the task
 */
<T> Future<T> submit(Runnable task, T result);

/**
 * 提交任务并返回一个Future对象
 * 一旦任务执行成功，调用Futrue的get方法将会返回null
 *
 * @param task 提交的任务
 * @return a Future representing pending completion of the task
 */
<T> Future<T> submit(Runnable task);

/**
 * 关闭线程池
 * 不再接受新的任务，同时等待已经提交的任务执行完成——包括那些还未开始执行的任务
 */
void shutdown();

/**
 * 关闭线程池
 * 将尝试取消所有运行中的任务，并且不再启动队列中尚未开始执行的任务
 */
void shutdownNow();

/**
 * @return 线程池是否关闭
 */
boolean isShutdown();

/**
 * @return 线程池关闭后，是否所有的任务都执行完毕
 */
boolean isTerminated();

/**
 * 当前线程阻塞，直到
 * 等所有已提交的任务（包括正在运行的和队列中等待的）执行完
 * 或者等超时时间到
 * 或者线程被中断，抛出InterruptedException
 * @return true（shutdown请求后所有任务执行完毕）或false（已超时）
 */
boolean awaitTermination(long timeout, TimeUnit unit) throws InterruptedException;

/**
 * 执行给定任务列表，并当所有任务都执行完毕后，返回Future对象列表
 * @param tasks 提交的任务列表
 * @return a list of Futures representing the tasks, in the same sequential order as produced by the iterator for the
 * given task list, each of which has completed
 */
<T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks) throws InterruptedException;

/**
 * 执行给定任务列表，并当所有任务都执行完毕后，返回Future对象列表
 * 如果超时，还没有执行完毕的任务会被取消
 * @param tasks 提交的任务列表
 * @return a list of Futures representing the tasks, in the same sequential order as produced by the iterator for the
 * given task list. If the operation did not time out, each task will have completed. If it did time out, some
 * of these tasks will not have completed.
 */
T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks, long timeout, TimeUnit unit)
    throws InterruptedException;

/**
 * 执行给定任务列表，并当其中一个任务执行完毕后，返回Future对象
 * @param tasks 提交的任务列表
 * @return the result returned by one of the tasks
 */
<T> T invokeAny(Collection<? extends Callable<T>> tasks) throws InterruptedException, ExecutionException;

/**
 * 执行给定任务列表，并当其中一个任务执行完毕后，返回Future对象
 * 如果超时，抛出TimeoutException
 * @param tasks 提交的任务列表
 * @return the result returned by one of the tasks
 */
<T> T invokeAny(Collection<? extends Callable<T>> tasks, long timeout, TimeUnit unit)
    throws InterruptedException, ExecutionException, TimeoutException;
```

## Executor框架的使用
调用`Exexutors`类中的静态方法来创建线程池，然后向线程池提交任务。
![Executors类图.png](/images/java/Executors类图.png)

不过目前大厂的编码规范中基本上都不建议使用`Executors`了，最重要的原因是：`Executors`提供的很多方法默认使用的都是无界的`LinkedBlockingQueue`，高负载情境下，无界队列很容易导致OOM，而OOM会导致所有请求都无法处理，这是致命问题。所以强烈建议使用有界队列。