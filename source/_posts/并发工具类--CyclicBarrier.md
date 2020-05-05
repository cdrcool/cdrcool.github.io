---
title: 并发工具类--CyclicBarrier
date: 2019-09-23 19:22:06
categories: Java并发
---
`CyclicBarrier`，即可重用屏障/栅栏，它允许一组线程彼此等待到达一个共同的障碍点，并可以设置线程都到达障碍点后要执行的命令。`CyclicBarrier`在包含固定大小的线程的程序中非常有用，这些线程有时必须彼此等待。这个屏障被称为循环，因为它可以在释放等待的线程之后重用。

## UML
![CyclicBarrier](/images/java/CyclicBarrier.png)

## 示例
### 并发任务
```java
/**
 * 模拟并发任务
 */
@Test
public void tasks() throws InterruptedException, BrokenBarrierException {
    // 存放任务及其执行结果映射
    Map<String, String> taskResults = new HashMap<>();

    // 创建栅栏：等待指定数目的任务都执行完后，输出所有任务执行结果
    // 注意：还需要考虑主线程
    CyclicBarrier barrier = new CyclicBarrier(4, () -> {
        System.out.println("所有任务已执行结束，任务结果：" + taskResults.toString());
    });

    // 模拟任务1
    Runnable task1 = () -> {
        // 模拟任务执行
        try {
            System.out.println("任务1执行中");
            Thread.sleep(200);
            taskResults.put("task1", "result1");
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        try {
            // 执行完成，等待屏障
            barrier.await();
        } catch (InterruptedException | BrokenBarrierException e) {
            e.printStackTrace();
        }
    };

    // 模拟任务2
    Runnable task2 = () -> {
        // 模拟任务执行
        try {
            System.out.println("任务2执行中");
            Thread.sleep(200);
            taskResults.put("task2", "result2");
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        try {
            // 执行完成，等待屏障
            barrier.await();
        } catch (InterruptedException | BrokenBarrierException e) {
            e.printStackTrace();
        }
    };

    // 模拟任务3
    Runnable task3 = () -> {
        // 模拟任务执行
        try {
            System.out.println("任务3执行中");
            Thread.sleep(200);
            taskResults.put("task3", "result3");
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        try {
            // 执行完成，等待屏障
            barrier.await();
        } catch (InterruptedException | BrokenBarrierException e) {
            e.printStackTrace();
        }
    };

    // 提交任务1
    new Thread(task1).start();
    // 提交任务2
    new Thread(task2).start();
    // 提交任务3
    new Thread(task3).start();

    // 主线程等待屏障
    barrier.await();
}
```
```text
任务2执行中
任务1执行中
任务3执行中
所有任务已执行结束，任务结果：{task1=result1, task2=result2, task3=result3}
```

## 源码
`CyclicBarrier`是通过内部聚合`ReentrantLock`和等待队列`Condition`来实现并发控制的。

```java
public CyclicBarrier(int parties, Runnable barrierAction) {
    if (parties <= 0) throw new IllegalArgumentException();
    // 线程个数
    this.parties = parties;
    // 还未到达屏障点的线程个数
    this.count = parties;
    // 所有线程到达屏障点后，要执行的命令
    this.barrierCommand = barrierAction;
}

public CyclicBarrier(int parties) {
    this(parties, null);
}

public int await() throws InterruptedException, BrokenBarrierException {
    try {
        return dowait(false, 0L);
    } catch (TimeoutException toe) {
        throw new Error(toe); // cannot happen
    }
}

private int dowait(boolean timed, long nanos)
    throws InterruptedException, BrokenBarrierException,
           TimeoutException {
    // 利用可重入锁，保证同时只有一个线程能进入
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        final Generation g = generation;

        if (g.broken)
            throw new BrokenBarrierException();

        if (Thread.interrupted()) {
            breakBarrier();
            throw new InterruptedException();
        }

        // 每次调用该方法，count自减
        int index = --count;
        // 等于0时，表示所有线程都已到达屏障
        if (index == 0) {  // tripped
            boolean ranAction = false;
            try {
                // 如果构造函数有传入command，就执行
                final Runnable command = barrierCommand;
                if (command != null)
                    command.run();
                ranAction = true;
                // 唤醒所有等待线程，并重置
                nextGeneration();
                return 0;
            } finally {
                if (!ranAction)
                    breakBarrier();
            }
        }

        // loop until tripped, broken, interrupted, or timed out
        // 所有线程没有全部到达屏障时
        for (;;) {
            try {
                // 如果未设置超时，调用Contion.await()，将线程放入等待队列
                if (!timed)
                    trip.await();
                // 
                else if (nanos > 0L)
                    // 如果设置了超时，调用Contion.awaitNanos(nanos)，将线程放入等待队列
                    nanos = trip.awaitNanos(nanos);
            } catch (InterruptedException ie) {
                if (g == generation && ! g.broken) {
                    breakBarrier();
                    throw ie;
                } else {
                    // We're about to finish waiting even if we had not
                    // been interrupted, so this interrupt is deemed to
                    // "belong" to subsequent execution.
                    Thread.currentThread().interrupt();
                }
            }

            if (g.broken)
                throw new BrokenBarrierException();

            if (g != generation)
                return index;

            if (timed && nanos <= 0L) {
                breakBarrier();
                throw new TimeoutException();
            }
        }
    } finally {
        lock.unlock();
    }
}
```

## 对比 CountDownLatch
* `CountDownLatch`主要用来解决一个线程等待多个线程的场景，而`CyclicBarrier`是一组线程之间互相等待
* `CountDownLatch`的计数器是不能循环利用的，也就是说一旦计数器减到0，再有线程调用`await()`，该线程会直接通过。但`CyclicBarrier`的计数器是可以循环利用的，而且具备自动重置的功能，一旦计数器减到0会自动重置到最开始设置的初始值。
* `CyclicBarrier`还可以设置回调函数，可以说是功能丰富。