---
title: 并发工具类--CompletionService
date: 2019-09-24 17:13:43
categories: Java并发
---
我们可以通过向`ExecutorService`提交`Callable`或`FutureTask`来异步获取线程执行结果，但这种方式的缺陷在于`Future.get()`会阻塞主线程，导致即使和它同时进行的其它线程已经执行完毕，也要等待这个耗时线程执行完才能获取结果，大大影响运行效率。
`ExecutorCompletionService`聚合了`Executor`和`BlockingQueue`，利用它我们每次都能从阻塞队列中获取到最近执行完毕的`futureTask`，而避免等待某个耗时较长的任务执行。

## 使用示例
```java
/**
 * 模拟电商询价系统
 */
@Test
public void enquiry() {
    // 创建线程池
    ExecutorService executor = Executors.newFixedThreadPool(3);
    // 创建CompletionService
    CompletionService<Integer> service = new ExecutorCompletionService<>(executor);

    // 异步向电商S1询价
    service.submit(this::getPriceByS1);
    // 异步向电商S2询价
    service.submit(this::getPriceByS2);
    // 异步向电商S3询价
    service.submit(this::getPriceByS3);

    // 获取任务结果
    IntStream.range(0, 3).forEach(index -> {
        try {
            // 按任务执行完毕的顺序，获取任务结果
            Future<Integer> future = service.take();
            Integer value = future.get();
            System.out.println("value: " + value);

            // 将询价结果异步保存到数据库
            executor.execute(() -> save(value));
        } catch (InterruptedException | ExecutionException e) {
            e.printStackTrace();
        }
    });
}

private Integer getPriceByS1() {
    return 100;
}

private Integer getPriceByS2() {
    return 200;
}

private Integer getPriceByS3() {
    return 300;
}

private void save(Integer value) {
    System.out.println("save value: " + value);
}

/**
 * 模拟从多个途径获取值，如果有一个途径获取成功，就取消所有任务并返回结果
 */
@Test
public void multiPath() {
    // 创建线程池
    ExecutorService executor = Executors.newFixedThreadPool(3);
    // 创建CompletionService
    CompletionService<Integer> service = new ExecutorCompletionService<>(executor);

    // ⽤于保存Future对象
    List<Future<Integer>> futures = new ArrayList<>(3);

    // 提交异步任务，并保存future到futures
    futures.add(service.submit(this::getPriceByS1));
    futures.add(service.submit(this::getPriceByS2));
    futures.add(service.submit(this::getPriceByS3));

    // 获取最快返回的任务执⾏结果
    Integer value = 0;
    try {
        // 只要有⼀个成功返回，则break
        for (int i = 0; i < 3; ++i) {
            try {
                value = service.take().get();
            } catch (InterruptedException | ExecutionException e) {
                e.printStackTrace();
            }

            // 简单地通过判空来检查是否成功返回
            if (value != null) {
                break;
            }
        }
    } finally {
        System.out.println("value: " + value);

        // 取消所有任务
        futures.forEach(future -> future.cancel(true));
    }
}
```

如上，我们通过调用`submit(Callable<V> task)`方法向`CompletionService`提交任务，获取任务结果则是先调用`take()`获取到`futureTask`，再调用`futureTask.get()`获取任务执行结果，如果`take()`没获取到`futureTask`，主线程就会一直阻塞。

## UML
![CompletionService UML](/images/java/CompletionService%20UML.png)

## 实现原理
`CompletionService`内部聚合了`Executor`和`BlockingQueue`，这样向其提交的任务都会交由`Executor`执行，任务结果则存放在`BlockingQueue`中，于是就能利用`BlockingQueue`的特性，使得在获取任务结果时，如果还没有任务完成，就可以选择阻塞或返回null。
至于任务结果是如何存放到`BlockingQueue`中的，则是通过将任务包装成`QueueingFuture`，`QueueingFuture`继承自`FutureTask`并覆盖了`done()`方法：task自行完毕后将结果保存到`BlockingQueue`中。

源码如下：
```java
public class ExecutorCompletionService<V> implements CompletionService<V> {
    /**
     * 线程池，由构造函数传入
     */
    private final Executor executor;
    /**
     * executor类型为AbstractExecutorService，aes的值就是executor，否则为null
    */
    private final AbstractExecutorService aes;
    /**
     * 阻塞队列，线程执行完毕后往该队列中插入值，主线程每次从该队列中获取值
     */
    private final BlockingQueue<Future<V>> completionQueue;

    /**
     * FutureTask extension to enqueue upon completion
     * 继承FutureTask，主要是实现了里面的空方法done
     */
    private class QueueingFuture extends FutureTask<Void> {
        QueueingFuture(RunnableFuture<V> task) {
            super(task, null);
            this.task = task;
        }
        /**
         * futureTask执行完毕后，就往阻塞队列中插入值
         */
        protected void done() { completionQueue.add(task); }
        private final Future<V> task;
    }

    private RunnableFuture<V> newTaskFor(Callable<V> task) {
        if (aes == null)
            return new FutureTask<V>(task);
        else
            return aes.newTaskFor(task);
    }

    private RunnableFuture<V> newTaskFor(Runnable task, V result) {
        if (aes == null)
            return new FutureTask<V>(task, result);
        else
            return aes.newTaskFor(task, result);
    }

    public ExecutorCompletionService(Executor executor) {
        if (executor == null)
            throw new NullPointerException();
        this.executor = executor;
        this.aes = (executor instanceof AbstractExecutorService) ?
            (AbstractExecutorService) executor : null;
        this.completionQueue = new LinkedBlockingQueue<Future<V>>();
    }

    public ExecutorCompletionService(Executor executor,
                                     BlockingQueue<Future<V>> completionQueue) {
        if (executor == null || completionQueue == null)
            throw new NullPointerException();
        this.executor = executor;
        this.aes = (executor instanceof AbstractExecutorService) ?
            (AbstractExecutorService) executor : null;
        this.completionQueue = completionQueue;
    }

    /**
     * 提交任务
     */
    public Future<V> submit(Callable<V> task) {
        if (task == null) throw new NullPointerException();
        RunnableFuture<V> f = newTaskFor(task);
        // 包装原始task，以实现task执行完毕后，会往阻塞队列中插入task
        executor.execute(new QueueingFuture(f));
        return f;
    }

    /**
     * 获取执行完毕的任务：从阻塞队列中获取，如果队列为空就会一直阻塞
     */
    public Future<V> submit(Runnable task, V result) {
        if (task == null) throw new NullPointerException();
        RunnableFuture<V> f = newTaskFor(task, result);
        executor.execute(new QueueingFuture(f));
        return f;
    }

    /**
     * 获取执行完毕的任务：从阻塞队列中获取，如果队列为空就会一直阻塞
     */
    public Future<V> take() throws InterruptedException {
        return completionQueue.take();
    }

    /**
     * 获取执行完毕的任务：从阻塞队列中获取，如果队列为空就会返回null
     */
    public Future<V> poll() {
        return completionQueue.poll();
    }

    /**
     * 获取执行完毕的任务：从阻塞队列中获取，如果队列为空就会先阻塞timeout，超时后还为空就返回null
     */
    public Future<V> poll(long timeout, TimeUnit unit)
            throws InterruptedException {
        return completionQueue.poll(timeout, unit);
    }

}
```
