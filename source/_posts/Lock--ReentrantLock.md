---
title: Lock--ReentrantLock
date: 2019-09-22 18:17:42
categories: Java并发
---
`ReentrantLock`实现了`Lock`接口，并提供了与`synchronized`相同的互斥性和内存可见性。与`synchronized`一样，`ReentrantLock`还提供了可重入的加锁语义，且同时支持公平锁和非公平锁的获取。`ReentrantLock`支持在`Lock`接口中定义的所有锁获取方式，并且与`synchronized`相比，它还为处理锁的不可用性问题提供了更高的灵活性。

## 使用示例
### 实现阻塞队列
```java
/**
 * 基于ReentrantLock和Condition实现有界阻塞队列
 *
 * @param <T>
 */
class BoundedQueue<T> {
    /**
     * 数组元数
     */
    private Object[] items;
    /**
     * 添加的下标
     */
    private int addIndex;
    /**
     * 删除的下标
     */
    private int removeIndex;
    /**
     * 数组当前数量
     */
    private int count;

    private Lock lock = new ReentrantLock();

    /**
     * 数组为空且要移除元素时，await；添加元素时，signal
     */
    private Condition notEmpty = lock.newCondition();

    /**
     * 数组已满且要添加元素时，await；移除元素时，signal
     */
    private Condition notFull = lock.newCondition();

    public BoundedQueue(int size) {
        items = new Object[size];
    }

    public void add(T t) throws InterruptedException {
        lock.lock();
        try {
            while (count == items.length) {
                notFull.await();
            }
            items[addIndex] = t;
            if (++addIndex == items.length) {
                addIndex = 0;
            }
            ++count;
            notEmpty.signal();
        } finally {
            lock.unlock();
        }
    }

    public T remove() throws InterruptedException {
        lock.lock();
        try {
            while (count == 0) {
                notEmpty.await();
            }
            Object x = items[removeIndex];
            if (++removeIndex == items.length) {
                removeIndex = 0;
            }
            --count;
            notFull.signal();
            //noinspection unchecked
            return (T) x;
        } finally {
            lock.unlock();
        }
    }
}
```

## UML
![ReentrantLock UML](/images/java/ReentrantLock%20UML.png)

## 实现原理
`ReentrantLock`是通过内部聚合`AbstractQueuedSynchronizer`的子类来`Sync`实现并发控制的。在其内部定义了基类`Sync`及其两个子类`NonfairSync`和`FairSync`，以支持公平锁与非公平锁的获取。

```java
/**
 * Base of synchronization control for this lock. Subclassed
 * into fair and nonfair versions below. Uses AQS state to
 * represent the number of holds on the lock.
 *
 * 队列同步器基类，有公平和非公平两个版本
 */
abstract static class Sync extends AbstractQueuedSynchronizer {
    private static final long serialVersionUID = -5179523762034025860L;

    /**
     * Performs {@link Lock#lock}. The main reason for subclassing
     * is to allow fast path for nonfair version.
     */
    abstract void lock();

    /**
     * Performs non-fair tryLock.  tryAcquire is implemented in
     * subclasses, but both need nonfair try for trylock method.
     *
     * 公平和非公平子类中，tryLock都会执行非公平获取锁
     */
    final boolean nonfairTryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        // 等于0时，表示还没线程获取到该锁
        if (c == 0) {
            if (compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        // 如果是已经获取到锁的线程，自增并重置state
        else if (current == getExclusiveOwnerThread()) {
            int nextc = c + acquires;
            if (nextc < 0) // overflow
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        // 其它线程不可再获取锁
        return false;
    }

    protected final boolean tryRelease(int releases) {
        int c = getState() - releases;
        if (Thread.currentThread() != getExclusiveOwnerThread())
            throw new IllegalMonitorStateException();
        boolean free = false;
        if (c == 0) {
            free = true;
            setExclusiveOwnerThread(null);
        }
        setState(c);
        return free;
    }

    protected final boolean isHeldExclusively() {
        // While we must in general read state before owner,
        // we don't need to do so to check if current thread is owner
        return getExclusiveOwnerThread() == Thread.currentThread();
    }

    final ConditionObject newCondition() {
        return new ConditionObject();
    }

    // Methods relayed from outer class

    final Thread getOwner() {
        return getState() == 0 ? null : getExclusiveOwnerThread();
    }

    final int getHoldCount() {
        return isHeldExclusively() ? getState() : 0;
    }

    final boolean isLocked() {
        return getState() != 0;
    }

    /**
     * Reconstitutes the instance from a stream (that is, deserializes it).
     */
    private void readObject(java.io.ObjectInputStream s)
        throws java.io.IOException, ClassNotFoundException {
        s.defaultReadObject();
        setState(0); // reset to unlocked state
    }
}

/**
 * Sync object for non-fair locks
 */
static final class NonfairSync extends Sync {
    private static final long serialVersionUID = 7316153563782823691L;

    /**
     * Performs lock.  Try immediate barge, backing up to normal
     * acquire on failure.
     */
    final void lock() {
        if (compareAndSetState(0, 1))
            setExclusiveOwnerThread(Thread.currentThread());
        else
            acquire(1);
    }

    protected final boolean tryAcquire(int acquires) {
        return nonfairTryAcquire(acquires);
    }
}

/**
 * Sync object for fair locks
 */
static final class FairSync extends Sync {
    private static final long serialVersionUID = -3000897897090466540L;

    final void lock() {
        acquire(1);
    }

    /**
     * Fair version of tryAcquire.  Don't grant access unless
     * recursive call or no waiters or is first.
     */
    protected final boolean tryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        // c=0意味着“锁没有被任何线程锁拥有”，
        if (c == 0) {
            // 若“锁没有被任何线程锁拥有”
            // 则判断“当前线程”是不是CLH队列中的第一个线程线程
            // 若是的话，则获取该锁，设置锁的状态，并切设置锁的拥有者为“当前线程”
            if (!hasQueuedPredecessors() &&
                compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        else if (current == getExclusiveOwnerThread()) {
            int nextc = c + acquires;
            if (nextc < 0)
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        return false;
    }
}
```