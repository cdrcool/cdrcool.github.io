---
title: 并发工具类--Future与FutureTask
date: 2019-09-21 17:52:21
categories: Java并发
---
## 概念
线程的创建方式中有两种，一种是实现`Runnable`接口，另一种是继承`Thread`，但是这两种方式都有个缺点，那就是在任务执行完成之后无法获取返回结果，于是就有了`Callable`接口，而`Future`接口与`FutureTask`类就是用来配合取得返回的结果。

## UML
![Future UML](/images/java/Future%20UML.png)

* Callable：具有返回值的异步任务
* Future: 表示异步计算的结果
* FutureTask：同时实现了Callable与Future接口

### Future
`Future`表示异步计算的结果，它提供了以下接口：
* V get()
获取异步执行的结果，如果没有结果可用，此方法会阻塞直到异步计算完成。
* V get(Long timeout , TimeUnit unit)
获取异步执行结果，如果没有结果可用，此方法会阻塞，但是会有时间限制，如果阻塞时间超过设定的timeout时间，该方法将抛出异常。
* boolean isDone()
如果任务执行结束，无论是正常结束或是中途取消还是发生异常，都返回true。
* boolean isCancelled()
如果任务完成前被取消，则返回true。
* boolean cancel(boolean mayInterruptRunning)
如果任务还没开始，执行`cancel(...)`方法将返回false；如果任务已经启动，执行`cancel(true)`方法将以中断执行此任务线程的方式来试图停止任务，如果停止成功，返回true；当任务已经启动，执行`cancel(false)`方法将不会对正在执行的任务线程产生影响(让线程正常执行到完成)，此时返回false；当任务已经完成，执行`cancel(...)`方法将返回false。`mayInterruptRunning`参数表示是否中断执行中的线程。

通过方法分析我们也知道实际上`Future`提供了3种功能：
* 能够中断执行中的任务
* 判断任务是否执行完成
* 获取任务执行完成后的结果
但是我们必须明白Future只是一个接口，我们无法直接创建对象，因此就需要其实现类`FutureTask`。

### FutureTask
`FutureTask`表示一个可取消的异步计算。它是`Future`接口的基本实现，提供了启动、取消计算、查询计算是否完成以及检索计算结果的方法。只有在计算完成后才能检索到结果，否则`get`方法将阻塞当前线程。一旦计算完成，就不能重新启动或取消计算(除非使用`runAndReset`调用计算)。
`FutureTask`可以用来包装`Callable`或`Runnable`对象。因为`FutureTask`实现了`Runnable`，所以可以将`FutureTask`提交给`Executor`执行。
除了作为一个独立的类之外，这个类还提供了`protected`功能，这在创建定制的任务类时可能很有用。

## 使用示例
```java
/**
 * 模拟并发任务
 */
@Test
public void test1() throws ExecutionException, InterruptedException {
    // 创建线程池
    ExecutorService executor = Executors.newFixedThreadPool(10);

    // 模拟任务1
    FutureTask<String> task1 = new FutureTask<>(() -> {
        System.out.println(Thread.currentThread().getName() + ": task1");
        Thread.sleep(200);
        return "result1";
    });
    // 模拟任务2
    FutureTask<String> task2 = new FutureTask<>(() -> {
        System.out.println(Thread.currentThread().getName() + ": task2");
        Thread.sleep(200);
        return "result2";
    });
    // 模拟任务3
    FutureTask<String> task3 = new FutureTask<>(() -> {
        System.out.println(Thread.currentThread().getName() + ": task3");
        Thread.sleep(200);
        return "result3";
    });

    // 提交任务1
    executor.submit(task1);
    // 提交任务2
    executor.submit(task2);
    // 提交任务3
    executor.submit(task3);

    // 获取任务1的执行结果
    String result1 = task1.get();
    // 获取任务2的执行结果
    String result2 = task2.get();
    // 获取任务3的执行结果
    String result3 = task3.get();

    System.out.println(result1);
    System.out.println(result2);
    System.out.println(result3);
}
```
```text
pool-1-thread-2: task2
pool-1-thread-1: task1
pool-1-thread-3: task3
result1
result2
result3
```

## 实现原理
### 状态流转
在`FutureTask`类中，定义了状态变量`state`，它有7个可能的状态，不同的状态决定了`FutureTask`的不同的行为。
```java
/**
 * 状态变量
 */
private volatile int state;
/**
 * 新建的
 */
private static final int NEW          = 0;
/**
 * 完成的中间状态
 */
private static final int COMPLETING   = 1;
/**
 * 已正常完成
 */
private static final int NORMAL       = 2;
/**
 * 异常终止
 */
private static final int EXCEPTIONAL  = 3;
/**
 * 已取消
 */
private static final int CANCELLED    = 4;
/**
 * 中断时的中间状态
 */
private static final int INTERRUPTING = 5;
/**
 * 已中断
 */
private static final int INTERRUPTED  = 6;
```

状态间可能的流转如下图：
![FutureTask State](/images/java/FutureTask State.png)

### 构造函数
即接受`Callable`，又接受`Runnable`，如果是`Runnable`，会将其适配为`Callable`。

```java
public FutureTask(Callable<V> callable) {
    if (callable == null)
        throw new NullPointerException();
    this.callable = callable;
    this.state = NEW;
}

public FutureTask(Runnable runnable, V result) {
    // 适配Runnable对象
    this.callable = Executors.callable(runnable, result);
    this.state = NEW;
}
```

### 成员
```java
/** 
 * 任务对象
 */
private Callable<V> callable;

/**
 * 任务执行结果
 */
private Object outcome;

/** 
 * 当前执行的的线程
 */
private volatile Thread runner;

/** 
 * 被阻塞的线程链
 */
private volatile WaitNode waiters;
```

### 核心方法
#### run
```java
public void run() {
    // 避免多个线程同时执行任务
    // 当第一个线程抢占任务成功后，后面的线程就什么也不做
    if (state != NEW ||
        !UNSAFE.compareAndSwapObject(this, runnerOffset,
                                     null, Thread.currentThread()))
        return;
    try {
        Callable<V> c = callable;
        if (c != null && state == NEW) {
            V result;
            boolean ran;
            try {
                result = c.call();
                ran = true;
            } catch (Throwable ex) {
                result = null;
                ran = false;
                // 设置异常值
                setException(ex);
            }
            if (ran)
                // 设置正常返回值
                set(result);
        }
    } finally {
        // runner must be non-null until state is settled to
        // prevent concurrent calls to run()
        // 重置runner
        runner = null;
        // state must be re-read after nulling runner to prevent
        // leaked interrupts
        int s = state;
        // 任务处于中断中的状态，则进行中断操作
        if (s >= INTERRUPTING)
            handlePossibleCancellationInterrupt(s);
    }
}

protected void setException(Throwable t) {
    // 将状态位设置成中间状态COMPLETING
    if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
        // 设置任务执行结果为异常对象
        outcome = t;
        // 将状态更为最终状态EXCEPTIO
        UNSAFE.putOrderedInt(this, stateOffset, EXCEPTIONAL);
        // 唤醒所有等待的线程，并调用done
        finishCompletion();
    }
}

protected void set(V v) {
    // 将状态位设置成中间状态COMPLETING
    if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
        // 设置任务执行结果为正常返回值
        outcome = v;
        // 将状态更为最终状态NORMAL
        UNSAFE.putOrderedInt(this, stateOffset, NORMAL);
        // 唤醒所有等待的线程，并调用done
        finishCompletion();
    }
}

private void finishCompletion() {
    // assert state > COMPLETING;
    // 断言此时state > COMPLETING
    for (WaitNode q; (q = waiters) != null;) {
        // 尝试将waiters全部置为null
        if (UNSAFE.compareAndSwapObject(this, waitersOffset, q, null)) {
            for (;;) {
                Thread t = q.thread;
                if (t != null) {
                    q.thread = null;
                    // 唤醒线程
                    LockSupport.unpark(t);
                }
                WaitNode next = q.next;
                if (next == null)
                    break;
                q.next = null; // unlink to help gc
                q = next;
            }
            break;
        }
    }

    done();

    // 减少内存占用
    callable = null;
}

private void handlePossibleCancellationInterrupt(int s) {
    // It is possible for our interrupter to stall before getting a
    // chance to interrupt us.  Let's spin-wait patiently.
    if (s == INTERRUPTING)
        while (state == INTERRUPTING)
            // 让出cpu时间片，等待cancel(true)执行完成，此时INTERRUPTING必然能更成INTERRUPTED
            Thread.yield(); // wait out pending interrupt

    // assert state == INTERRUPTED;

    // We want to clear any interrupt we may have received from
    // cancel(true).  However, it is permissible to use interrupts
    // as an independent mechanism for a task to communicate with
    // its caller, and there is no way to clear only the
    // cancellation interrupt.
    //
    // Thread.interrupted();
}
```

#### get
```java
public V get() throws InterruptedException, ExecutionException {
    int s = state;
    // 如果是新建或完成的中间状态，则等待任务执行完
    if (s <= COMPLETING)
        s = awaitDone(false, 0L);
    // 返回已完成任务的结果或抛出异常
    return report(s);
}

private int awaitDone(boolean timed, long nanos)
    throws InterruptedException {
    final long deadline = timed ? System.nanoTime() + nanos : 0L;
    WaitNode q = null;
    boolean queued = false;
    for (;;) {
        // 如果线程已中断，则直接将当前节点q从waiters中移出
        if (Thread.interrupted()) {
            removeWaiter(q);
            throw new InterruptedException();
        }

        int s = state;
        // 最终状态，返回任务执行结果
        if (s > COMPLETING) {
            if (q != null)
                q.thread = null;
            return s;
        }
        // 中间状态，则等待任务执行完
        else if (s == COMPLETING) // cannot time out yet
            Thread.yield();
        // 如果发现尚未有节点，则创建节点
        else if (q == null)
            q = new WaitNode();
        // 如果当前节点尚未入队，则将当前节点放到waiters中
        else if (!queued)
            queued = UNSAFE.compareAndSwapObject(this, waitersOffset,
                                                 q.next = waiters, q);
        // 线程被阻塞指定时间后再唤醒
        else if (timed) {
            nanos = deadline - System.nanoTime();
            if (nanos <= 0L) {
                removeWaiter(q);
                return state;
            }
            LockSupport.parkNanos(this, nanos);
        }
        // 线程一直被阻塞直到被其他线程唤醒
        else
            LockSupport.park(this);
    }
}
```

#### cancel
```java
public boolean cancel(boolean mayInterruptIfRunning) {
    // 如果状态不为NEW，且无法将状态更新为INTERRUPTING或CANCELLED，则直接返回取消失败
    if (!(state == NEW &&
          UNSAFE.compareAndSwapInt(this, stateOffset, NEW,
              mayInterruptIfRunning ? INTERRUPTING : CANCELLED)))
        return false;
    try {    // in case call to interrupt throws exception
        // 允许运行中进行中断操作
        if (mayInterruptIfRunning) {
            try {
                Thread t = runner;
                if (t != null)
                    t.interrupt();
            } finally { // final state
                // 中断成功，则置为最终状态
                UNSAFE.putOrderedInt(this, stateOffset, INTERRUPTED);
            }
        }
    } finally {
        // 唤醒所有等待的线程，并调用done
        finishCompletion();
    }
    return true;
}
```