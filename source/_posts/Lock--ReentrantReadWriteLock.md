---
title: Lock--ReentrantReadWriteLock
date: 2019-09-22 18:17:43
categories: Java并发
---
实际工作中，为了优化性能，我们经常会使用缓存，例如缓存元数据、缓存基础数据等，这就是一种典型的读多写少应用场景。缓存之所以能提升性能，一个重要的条件就是缓存的数据一定是读多写少的，例如元数据和基础数据基本上不会发生变化（写少），但是使用它们的地方却很多（读多）。
针对读多写少这种并发场景，Java SDK并发包提供了读写锁`ReadWriteLock`，非常容易使用，并且性能很好。

## 读写锁
读写锁，并不是Java语言特有的，而是一个广为使用的通用技术，所有的读写锁都遵守以下三条基本原则：
* 允许多个线程同时读共享变量；
* 只允许一个线程写共享变量；
* 如果一个写线程正在执行写操作，此时禁止读线程读共享变量。

读写锁与互斥锁的一个重要区别就是读写锁允许多个线程同时读共享变量，而互斥锁是不允许的，这是读写锁在读多写少场景下性能优于互斥锁的关键。但读写锁的写操作是互斥的，当一个线程在写共享变量的时候，是不允许其他线程执行写操作和读操作。

## 使用示例
### 实现缓存
```java
**
 * 基于ReentrantReadWriteLock实现缓存
 */
class Cache {
    private static Map<String, Object> map = new HashMap<>();
    private static ReentrantReadWriteLock rwl = new ReentrantReadWriteLock();
    private static Lock r = rwl.readLock();
    private static Lock w = rwl.writeLock();

    public static Object get(String key) {
        r.lock();
        try {
            return map.get(key);
        } finally {
            r.unlock();
        }
    }

    public static Object put(String key, Object value) {
        w.lock();
        try {
            return map.put(key, value);
        } finally {
            w.unlock();
        }
    }

    public static void clear() {
        w.lock();
        try {
            map.clear();
        } finally {
            w.unlock();
        }
    }
}
```

## UML
![ReentrantReadWriteLock UML](/images/java/ReentrantReadWriteLock%20UML.png)

## 实现原理
`ReentrantReadWriteLock`内部维护了一对锁：一个读锁（readerLock）和一个写锁（writerLock）。这两个锁都是基于AQS实现的：它们内部聚合了同一个`sync`。那么`sync`如如何同时表示读锁和写锁的呢？`ReentrantReadWriteLock`通过“按位切割使用”巧妙地将同步状态变量`state`切分成了读锁和写锁两个部位，即高16位表示写，低16位表示读。

`ReentrantReadWriteLock`支持公平锁与非公平锁的获取，默认实现为非公平锁，如果要实现公平锁，往构造函数中传参：trues即可。

有一点需要注意，那就是只有写锁支持条件变量，读锁是不支持条件变量的，读锁调用`newCondition()`会抛出`UnsupportedOperationException`异常。

## 锁升级/锁降级
### 锁升级（不支持）
同一个线程中，在没有释放读锁的情况下，就去申请写锁，这属于锁升级。下面的代码演示了锁升级：
```java
// 读缓存
r.lock();
try {
    v = m.get(key);
    if (v == null) {
        w.lock();
        try {
            // 再次验证并更新缓存
            // 省略详细代码
        } finally{
            w.unlock();
        }
    }
} finally{
    r.unlock();
}
```
代码看上去好像没有问题，先是获取读锁，然后再升级为写锁，但是`ReadWriteLock`并**不支持这种锁升级**，因为当读锁还没有释放时，此时获取写锁，会导致写锁永久等待，最终导致相关线程都被阻塞，永远也没有机会被唤醒。

### 锁降级（支持）
同一个线程中，在没有释放写锁的情况下，就去申请读锁，这属于锁降级。下面的代码演示了锁降级：
```java
class CachedData {
    Object data;
    volatile boolean cacheValid;

    final ReadWriteLock rwl = new ReentrantReadWriteLock();
    // 读锁
    final Lock r = rwl.readLock();
    // 写锁
    final Lock w = rwl.writeLock();

    void processCachedData() {
        // 获取读锁
        r.lock();
        if (!cacheValid) {
            // 释放读锁，因为不允许读锁的升级
            r.unlock();
            // 获取写锁
            w.lock();
            try {
                // 再次检查状态
                if (!cacheValid) {
                    data = ...
                    cacheValid = true;
                }
                // 释放写锁前，降级为读锁
                // 降级是可以的
                r.lock();
            } finally {
                // 释放写锁
                w.unlock();
            }
        }

        // 此处仍然持有读锁
        try {
            use(data);
        }
        finally {
            r.unlock();
        }
    }
}
```

问题：锁降级中读锁的获取是否有必要？
答：要必要，主要是为了保证数据的可见性，如果当前线程不获取读锁而直接释放写锁，假设此刻另一个线程（T）获取了写锁并修改了数据，那么当前线程是无法感知线程T的数据更新。如果当前线程获取读锁，即遵循锁降级的步骤，则线程T将会被阻塞，知道当前线程使用数据并释放读锁之后，线程T才能获取写锁进行数据更新。
