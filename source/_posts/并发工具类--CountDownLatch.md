---
title: 并发工具类--CountDownLatch
date: 2019-09-23 19:22:04
categories: Java并发
---
`CountDownLatch`，通常称之为闭锁，它允许一个或多个线程等待，直到在其他线程中执行的一组操作完成。

## UML
![CountDownLatch](/images/java/CountDownLatch.png)

## 使用示例
### 并发任务
```java
/**
 * 模拟并发任务
 */
@Test
public void tasks() throws InterruptedException {
    // 创建线程池
    ExecutorService executor = Executors.newFixedThreadPool(5);

    // 存放任务及其执行结果映射
    Map<String, String> taskResults = new HashMap<>();

    // 创建闭锁：当count减为0时，线程才会继续往下执行
    CountDownLatch countDownLatch = new CountDownLatch(3);

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

        countDownLatch.countDown();
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

        countDownLatch.countDown();
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

        countDownLatch.countDown();
    };

    // 提交任务1
    executor.execute(task1);
    // 提交任务2
    executor.execute(task2);
    // 提交任务3
    executor.execute(task3);

    // 等待所有任务执行完毕
    countDownLatch.await();

    System.out.println("所有任务已执行结束，任务结果：" + taskResults.toString());
}
```
```text
任务2执行中
任务1执行中
任务3执行中
所有任务已执行结束，任务结果：{task1=result1, task2=result2, task3=result3}
```

### 赛跑
```java
/**
 * 模拟赛跑，发令枪响后同时开跑，所有人到达后比赛结束
 */
@Test
public void race() throws InterruptedException {
    // 创建线程池
    ExecutorService executor = Executors.newFixedThreadPool(10);

    // 创建闭锁begin：控制线程同时开始
    CountDownLatch begin = new CountDownLatch(1);
    // 创建闭锁end：控制当线程都结束后，主线程才继续执行
    CountDownLatch end = new CountDownLatch(10);

    IntStream.range(0, 10).forEach(index -> {
        executor.execute(() -> {
            // 选手员等待发令枪
            try {
                begin.await();
                System.out.println(String.format("选手：%s正在赛跑中", index));
            } catch (InterruptedException e) {
                e.printStackTrace();
            }

            // 选手赛跑耗时
            try {
                Thread.sleep(new Random().nextInt(10) * 300);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }

            System.out.println(String.format("选手：%s到达终点", index));

            end.countDown();
        });
    });

    System.out.println("预备，跑！");
    // 发令枪响，所有选手开跑
    begin.countDown();

    // 等待所有选手到达终点
    end.await();

    System.out.println("比赛结束！");
}
```

## 实现原理
`Semaphore`是通过内部聚合`AbstractQueuedSynchronizer`的子类来`Sync`实现并发控制的。

```java
public class CountDownLatch {
    private static final class Sync extends AbstractQueuedSynchronizer {
        private static final long serialVersionUID = 4982264981922014374L;

        Sync(int count) {
            setState(count);
        }

        int getCount() {
            return getState();
        }

        protected int tryAcquireShared(int acquires) {
            return (getState() == 0) ? 1 : -1;
        }

        protected boolean tryReleaseShared(int releases) {
            // Decrement count; signal when transition to zero
            for (;;) {
                int c = getState();
                if (c == 0)
                    return false;
                int nextc = c-1;
                if (compareAndSetState(c, nextc))
                    return nextc == 0;
            }
        }
    }

    private final Sync sync;

    public CountDownLatch(int count) {
        if (count < 0) throw new IllegalArgumentException("count < 0");
        this.sync = new Sync(count);
    }

    public void await() throws InterruptedException {
        sync.acquireSharedInterruptibly(1);
    }

    public boolean await(long timeout, TimeUnit unit)
        throws InterruptedException {
        return sync.tryAcquireSharedNanos(1, unit.toNanos(timeout));
    }

    public void countDown() {
        sync.releaseShared(1);
    }

    public long getCount() {
        return sync.getCount();
    }

    public String toString() {
        return super.toString() + "[Count = " + sync.getCount() + "]";
    }
}
```