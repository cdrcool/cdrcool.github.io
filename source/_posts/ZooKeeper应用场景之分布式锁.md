---
title: ZooKeeper 应用场景之分布式锁
date: 2020-02-01 19:01:00
categories: ZooKeeper
---
分布式锁是控制分布式系统间同步访问共享资源的一种方式。如果不同的系统或同一个系统的不同主机之间共享了同一个资源，那么访问这些资源的时候，需要使用互斥的手段来防止彼此之间的干扰，以保证一致性，这种情况就需要使用分布式锁。

## 思路
使用临时顺序 znode 来表示获取锁的请求，创建最小后缀数字 znode 的用户成功拿到锁。

![分布式锁示例1](/images/zookeeper/分布式锁示例1.png)

### 避免羊群效应（herd effect）
把锁请求者按照后缀数字进行排队，后缀数字小的锁请求者先获取锁。如果所有的锁请求者都 watch 锁持有者，当代表锁请求者的 znode 被删除以后，所有的锁请求者都会通知到，但是只有一个锁请求者能拿到锁。这就是羊群效应。

为了避免羊群效应，每个锁请求者 watch 它前面的锁请求者。每次锁被释放，只会有一个锁请求者会被通知到。这样做还让锁的分配更具有公平性，锁的分配遵循先到先得的原则。

![分布式锁示例2](/images/zookeeper/分布式锁示例2.png)

## 实现
### bash 命令
1. 终端 1 创建节点：`create -e /lock`
2. 终端 2 创建节点：`create -e /lock`，此时会提示：Node already exists: /lock
3. 终端 2 watch 节点：`stat -w /lock`
4. 终端 1 关闭：`quit`
5. 终端 2 创建节点：`create -e /lock`，由于终端 1 会释放掉锁，因此能创建成功

### Curator
```java
/**
 * Simulates some external resource that can only be access by one process at a time
 */
@Slf4j
public class FakeLimitedResource {
    private final AtomicBoolean inUse = new AtomicBoolean(false);

    public void use() {
        // in a real application this would be accessing/manipulating a shared resource

        if (!inUse.compareAndSet(false, true)) {
            throw new IllegalStateException("Needs to be used by one client at a time");
        }

        try {
            Thread.sleep(3 * new Random().nextInt(1));
        } catch (InterruptedException e) {
            log.error("Sleep with exception", e);
        } finally {
            inUse.set(false);
        }
    }
}

/**
 * 资源调用者
 */
@Slf4j
public class ResourceInvoker {
    /**
     * 公共受限资源
     */
    public FakeLimitedResource resource;
    /**
     * 分布式锁
     */
    private InterProcessMutex lock;
    /**
     * 获取锁之前等待的时间（单位：秒）
     */
    private int waitSeconds;

    public ResourceInvoker(FakeLimitedResource resource, InterProcessMutex lock, int waitSeconds) {
        this.resource = resource;
        this.lock = lock;
        this.waitSeconds = waitSeconds;
    }

    public void invoke() throws Exception {
        String threadName = Thread.currentThread().getName();

        if (waitSeconds <= 0) {
            lock.acquire();
        } else if (!lock.acquire(waitSeconds, TimeUnit.SECONDS)) {
            throw new IllegalStateException("Thread[" + threadName + "] could not acquire the lock");
        }

        try {
            log.info("Thread[{}] had the lock", threadName);
            resource.use();
        } finally {
            log.info("Thread[{}] releasing the lock", threadName);
            // always release the lock in a finally block
            lock.release();
        }
    }
}

/**
 * 分布式锁使用示例
 */
public class LockingExample {
    private final CuratorFramework client;

    public LockingExample(CuratorFramework client) {
        this.client = client;
    }

    public void execute() throws InterruptedException {
        FakeLimitedResource resource = new FakeLimitedResource();
        InterProcessMutex lock = new InterProcessMutex(client, "/examples/lock");
        ResourceInvoker invoker = new ResourceInvoker(resource, lock, 3);

        int poolSize = 5;
        int repetitions = poolSize * 10;
        ExecutorService service = new ThreadPoolExecutor(
                poolSize,
                poolSize,
                0L,
                TimeUnit.MILLISECONDS,
                new LinkedBlockingQueue<>(),
                new CustomThreadFactory("lock"));
        Callable<Void> task = () -> {
            try {
                for (int j = 0; j < repetitions; ++j) {
                    invoker.invoke();
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
            return null;
        };

        for (int i = 0; i < poolSize; ++i) {
            service.submit(task);
        }

        service.shutdown();
        service.awaitTermination(10, TimeUnit.MINUTES);
    }
}
```