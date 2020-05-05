---
title: 并发设计模式--WorkerThread设计模式
date: 2019-10-23 21:10:12
categories: Java并发
---
## 概念
Worker Thread 设计模式有时也称为流水线设计模式，这种设计模式类似于工厂流水线。在 Worker Thread 模式中，工人线程 Worker thread 会逐个取回工作并进行处理，当所有工作全部完成后，工人线程会等待新的工作到来。

## 线程池
Java 线程池就是 Worker Thread 设计模式的一种运用。

Java 的线程池既能够避免无限制地创建线程导致 OOM，也能避免无限制地接收任务导致 OOM。强烈建议你用创建有界的队列来接收任务。
当请求量大于有界队列的容量时，就需要合理地拒绝请求。如何合理地拒绝呢？这需要我们结合具体的业务场景来制定，即便线程池默认的拒绝策略能够满足你的需求，也同样建议我们在创建线程池时，清晰地指明拒绝策略。
同时，为了便于调试和诊断问题，同样强烈建议你在实际工作中给线程赋予一个业务相关的名字。
```java
ExecutorService es = new ThreadPoolExecutor(
    50, 
    500,
    60L, 
    TimeUnit.SECONDS,
    // 注意要创建有界队列
    new LinkedBlockingQueue<Runnable>(2000),
    // 建议根据业务需求实现ThreadFactory
    r -> {
        return new Thread(r, "echo-"+ r.hashCode());
    },
    // 建议根据业务需求实现RejectedExecutionHandler
    new ThreadPoolExecutor.CallerRunsPolicy()
);
```

## 避免线程死锁
使用线程池过程中，还要注意一种线程死锁的场景。如果提交到相同线程池的任务不是相互独立的，而是有依赖关系的，那么就有可能导致线程死锁。

用下面的示例代码来模拟该应用，如果我们执行下面的这段代码，会发现它永远执行不到最后一行。执行过程中没有任何异常，但是应用已经停止响应了。
```java
// L1、L2阶段共⽤的线程池
ExecutorService es = Executors.newFixedThreadPool(2);
// L1阶段的闭锁
CountDownLatch l1 = new CountDownLatch(2);

for (int i=0; i<2; i++) {
    System.out.println("L1");
    // 执⾏L1阶段任务
    es.execute(() -> {
        // L2阶段的闭锁
        CountDownLatch l2 = new CountDownLatch(2);
        // 执⾏L2阶段⼦任务
        for (int j=0; j<2; j++) {
            es.execute(() -> {
                System.out.println("L2");
                l2.countDown();
            });
        }
        // 等待L2阶段任务执⾏完
        l2.await();
        l1.countDown();
    });
}

// 等着L1阶段任务执⾏完
l1.await();
System.out.println("end");
```

由此可见，提交到相同线程池中的任务一定是相互独立的，否则就一定要慎重。

