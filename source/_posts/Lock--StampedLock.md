---
title: Lock--StampedLock
date: 2019-09-22 18:17:44
categories: Java并发
---
我们知道在`ReadWriteLock`中写锁和读锁是互斥的，也就是如果有一个线程在写共享变量的话，其他线程读共享变量都会阻塞。
`StampedLock`把读分为了悲观读锁和乐观读，悲观读锁就等价于`ReadWriteLock`的读锁，而乐观读在一个线程写共享变量时，不会被阻塞，乐观读是不加锁的。所以没锁肯定是比有锁的性能好，这样的话在大并发读情况下效率就更高了！
`StampedLock`的用法稍稍有点不同，在获取写锁和悲观读锁时，都会返回一个stamp，解锁时需要传入这个stamp，在乐观读时是用来验证共享变量是否被其他线程写过。

## 使用示例
### 实现Point
```java
public class Point {
    private double x, y;
    private final StampedLock sl = new StampedLock();

    void move(double deltaX, double deltaY) { // an exclusively locked method
        // 排他锁-写锁
        long stamp = sl.writeLock();
        try {
            x += deltaX;
            y += deltaY;
        } finally {
            sl.unlockWrite(stamp);
        }
    }

    double distanceFromOrigin() { // A read-only method
        // 乐观读
        long stamp = sl.tryOptimisticRead();
        double currentX = x, currentY = y;
        if (!sl.validate(stamp)) {
            // 时间戳校验不同过，升级为悲观读锁
            stamp = sl.readLock();
            try {
                currentX = x;
                currentY = y;
            } finally {
                sl.unlockRead(stamp);
            }
        }
        return Math.sqrt(currentX * currentX + currentY * currentY);
    }

    void moveIfAtOrigin(double newX, double newY) { // upgrade
        // Could instead start with optimistic, not read mode
        long stamp = sl.readLock();
        try {
            while (x == 0.0 && y == 0.0) {
                // 尝试将读锁升级为写锁
                long ws = sl.tryConvertToWriteLock(stamp);
                if (ws != 0L) {
                    stamp = ws;
                    x = newX;
                    y = newY;
                    break;
                } else {
                    sl.unlockRead(stamp);
                    stamp = sl.writeLock();
                }
            }
        } finally {
            sl.unlock(stamp);
        }
    }
}
```

## UML
![ReentrantLock UML](/images/java/ReentrantLock%20UML.png)

## 注意事项
* StampedLock 不支持重入
* StampedLock 的悲观读锁、写锁都不支持条件变量
* 如果线程阻塞在`StampedLock`的`readLock()`或者`writeLock()`上时，此时调用该阻塞线程的`interrupt()`方法，会导致CPU飙升。所以，使用`StampedLock`一定不要调用中断操作，如果需要支持中断功能，一定使用可中断的悲观读锁`readLockInterruptibly()`和写锁`writeLockInterruptibly()`

## 模板代码
```javas
final StampedLock sl = new StampedLock();
// 乐观读
long stamp = sl.tryOptimisticRead();

// 读⼊⽅法局部变量
......

// 校验 stamp
if (!sl.validate(stamp)){
    // 升级为悲观读锁
    stamp = sl.readLock();
    try {
        // 读⼊⽅法局部变量
        .....
    } finally {
        // 释放悲观读锁
        sl.unlockRead(stamp);
    }
}

// 使⽤⽅法局部变量执⾏业务操作
......

long stamp = sl.writeLock();
try {
     // 写共享变量
     ......
} finally {
    sl.unlockWrite(stamp);
}
```