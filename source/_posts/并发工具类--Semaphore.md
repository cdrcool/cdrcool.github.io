---
title: 并发工具类--Semaphore
date: 2019-09-23 19:22:05
categories: Java并发
---
`Semaphore`，即信号量，是用来控制同时访问特定资源的线程数量，它通过协调各个线程，以保证合理的使用公共资源。

## 模型
信号量模型可以简单概括为：一个计数器，一个等待队列，三个方法。在信号量模型里，计数器和等待队列对外是透明的，所以只能通过信号量模型提供的三个方法来访问它们，这三个方法分别是：`init()`、`down()`和`up()`。我们可以结合下图来形象化地理解：
![信号量模型图](/images/java/信号量模型图.png)

这三个方法详细的语义具体如下所示：
* init()：设置计数器的初始值。
* down()：计数器的值减 1；如果此时计数器的值小于0，则当前线程将被阻塞，否则当前线程可以继续执行。
* up()：计数器的值加 1；如果此时计数器的值小于或者等于0，则唤醒等待队列中的一个线程，并将其从等待队列中移除。

这里提到的 init()、down() 和 up() 三个方法都是原子性的，并且这个原子性是由信号量模型的实现方保证的。

## UML
在Java SDK里面，信号量模型是由`Semaphore`实现的。
![Semaphore](/images/java/Semaphore.png)

## 使用示例
### 获取数据库连接
假设有以下需求：要读取几万个文件的数据，因为都是IO密集型任务，因此可以启动几十个线程并发地读取，但是如果读取到内存后，还需要存储到数据库中，而数据库的连接数只有10个，这时就必须控制只有10个线程能同时获取到数据库连接，否则会报错提示无法获取数据库连接。
```java
/**
 * 模拟获取数据库连接后持久化数据
 */
@Test
public void test1() throws InterruptedException {
    CountDownLatch countDownLatch = new CountDownLatch(30);

    // 创建线程池
    ExecutorService executor = Executors.newFixedThreadPool(30);
    // 创建信号量
    Semaphore semaphore = new Semaphore(10);

    IntStream.range(0, 30).forEach(index -> {
        executor.execute(() -> {
            try {
                semaphore.acquire();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }

            System.out.println("save data: " + index);

            semaphore.release();

            countDownLatch.countDown();
        });
    });

    countDownLatch.await();
}
```

### 资源池
```java
/**
 * 基于Semaphore实现资源池
 */
class Pool {
    private static final int MAX_AVAILABLE = 10;
    private final Semaphore available = new Semaphore(MAX_AVAILABLE, true);

    public Object getItem() throws InterruptedException {
        available.acquire();
        return getNextAvailableItem();
    }

    public void putItem(Object x) {
        if (markAsUnused(x)) {
            available.release();
        }
    }

    protected Object[] items = new String[]{"a", "b", "c", "d", "e", "f", "g", "h", "i", "j"};
    protected boolean[] used = new boolean[MAX_AVAILABLE];

    protected synchronized Object getNextAvailableItem() {
        for (int i = 0; i < MAX_AVAILABLE; ++i) {
            if (!used[i]) {
                used[i] = true;
                return items[i];
            }
        }
        return null;
    }

    protected synchronized boolean markAsUnused(Object item) {
        for (int i = 0; i < MAX_AVAILABLE; ++i) {
            if (item == items[i]) {
                if (used[i]) {
                    used[i] = false;
                    return true;
                } else {
                    return false;
                }
            }
        }
        return false;
    }
}
```

## 实现原理
`Semaphore`是通过内部聚合`AbstractQueuedSynchronizer`的子类来`Sync`实现并发控制的，并且它同时支持公平或非公平锁的获取。

```java
/**
 * 基类，有公平和非公平两个子类
 */
abstract static class Sync extends AbstractQueuedSynchronizer {
    private static final long serialVersionUID = 1192457210091910933L;

    Sync(int permits) {
        setState(permits);
    }

    final int getPermits() {
        return getState();
    }

    /**
     * 非公平获取
     */
    final int nonfairTryAcquireShared(int acquires) {
        for (;;) {
            int available = getState();
            int remaining = available - acquires;
            if (remaining < 0 ||
                compareAndSetState(available, remaining))
                return remaining;
        }
    }

    protected final boolean tryReleaseShared(int releases) {
        for (;;) {
            int current = getState();
            int next = current + releases;
            if (next < current) // overflow
                throw new Error("Maximum permit count exceeded");
            if (compareAndSetState(current, next))
                return true;
        }
    }

    final void reducePermits(int reductions) {
        for (;;) {
            int current = getState();
            int next = current - reductions;
            if (next > current) // underflow
                throw new Error("Permit count underflow");
            if (compareAndSetState(current, next))
                return;
        }
    }

    final int drainPermits() {
        for (;;) {
            int current = getState();
            if (current == 0 || compareAndSetState(current, 0))
                return current;
        }
    }
}

/**
 * 非公平版本
 */
static final class NonfairSync extends Sync {
    private static final long serialVersionUID = -2694183684443567898L;

    NonfairSync(int permits) {
        super(permits);
    }

    protected int tryAcquireShared(int acquires) {
        return nonfairTryAcquireShared(acquires);
    }
}

/**
 * 公平版本
 */
static final class FairSync extends Sync {
    private static final long serialVersionUID = 2014338818796000944L;

    FairSync(int permits) {
        super(permits);
    }

    protected int tryAcquireShared(int acquires) {
        for (;;) {
            // 首节点才有机会执行
            if (hasQueuedPredecessors())
                return -1;
            int available = getState();
            int remaining = available - acquires;
            if (remaining < 0 ||
                compareAndSetState(available, remaining))
                return remaining;
        }
    }
}
```