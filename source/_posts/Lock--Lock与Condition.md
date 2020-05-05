---
title: Lock--Lock--Lock与Condition
date: 2019-09-22 18:17:41
categories: Java并发
---
Java SDK并发包通过`Lock`和`Condition`两个接口来实现管程，其中`Lock`用于解决互斥问题，`Condition`用于解决同步问题。

## Lock
相较于内置加锁机制`synchronized`，显示锁`Lock`除了支持类似`synchronized`隐式加锁的方法外，还支持超时、非阻塞、可中断的方式获取锁，这三种方式为我们编写更加安全、健壮的并发程序提供了很大的便利。

### UML
![Lock](/images/java/Lock.png)

## Condition
### UML
![Condition](/images/java/Condition.png)

## 使用示例
### RPC请求（Dubbo源码）
```java
// 创建锁与条件变量
private final Lock lock = new ReentrantLock();
private final Condition done = lock.newCondition();

// 调用方通过该方法等待结果
Object get(int timeout){
    long start = System.nanoTime();
    lock.lock();
    try {
        while (!isDone()) {
            done.await(timeout);
            long cur=System.nanoTime();
            if (isDone() || cur-start > timeout){
                break;
            }
        }
    } finally {
        lock.unlock();
    }
    if (!isDone()) {
        throw new TimeoutException();
    }
    return returnFromResponse();
}
// RPC 结果是否已经返回
boolean isDone() {
    return response != null;
}

// RPC 结果返回时调用该方法
private void doReceived(Response res) {
    lock.lock();
    try {
        response = res;
        if (done != null) {
            done.signal();
        }
    } finally {
        lock.unlock();
    }
}
```