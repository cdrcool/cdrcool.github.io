---
title: 并发设计模式--线程本地存储模式(ThreadLocal)
date: 2019-09-26 16:20:01
categories: Java并发
---
## 概念
所谓“没有共享，就没有伤害”，ThreadLocal 是一个本地线程副本变量工具类，它主要用于将私有线程和该线程存放的副本对象做一个映射，各个线程之间的变量互不干扰，在高并发场景下，可以实现无状态的调用，特别适用于各个线程依赖不同的变量值完成操作的场景。

## 应用示例
```java
/**
 * 实现线程安全的DateFormat
 *
 * @author cdrcool
 */
class SafeDateFormat {
    private static final ThreadLocal<DateFormat> TL =
            ThreadLocal.withInitial(() -> new SimpleDateFormat("yyyy-MM-dd HH:mm:ss"));

    public static DateFormat get() {
        return TL.get();
    }
}

//不同线程执⾏下⾯代码，返回的dateFormat是不同的
DateFormat dateFormat = SafeDateFormat.get();
```

## 实现原理
### 自定义实现
在解释 ThreadLocal 的工作原理之前，我们可以先自己想想：如果让我们来实现 ThreadLocal 的功能，会怎么设计呢？ThreadLocal 的目标是让不同的线程有不同的变量 v，那最直接的方法就是创建一个 Map，它的 Key 是线程，value 是每个线程拥有的变量 V，ThreadLocal 内部持有这样的一个 Map 就可以了。如下代码所示：
```java
class MyThreadLocal<T> {
    Map<Thread, T> locals = new ConcurrentHashMap<>();
    //获取线程变量
    T get() {
        return locals.get(Thread.currentThread());
    }
    //设置线程变量
    void	set(T	t)	{
        locals.put(Thread.currentThread(),	t);
    }
}
```

### Java实现
Java的实现里面也有一个Map，叫做 ThreadLocalMap ，不过持有 ThreadLocalMap 的不是 ThreadLocal，而是 Thread。Thread 这个类内部有一个私有属性 threadLocals，其类型就是 ThreadLocalMap，ThreadLocalMap 的 key 是 ThreadLocal。我们可以结合下面的示意图和精简之后的 Java 实现代码来理解。

![ThreaLocalMap示意图](/images/java/ThreaLocalMap示意图.png)

```java
class Thread {
    // 内部持有ThreadLocalMap
    ThreadLocal.ThreadLocalMap threadLocals;
}

class ThreadLocal<T> {
    ...

    public T get()	{
        // ⾸先获取线程持有的ThreadLocalMap，等价于：ThreadLocalMap map = Thread.currentThread().threadLocals;
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            // 在ThreadLocalMap中查找变量
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        return setInitialValue();
    }

    static class ThreadLocalMap {
        // Entry定义
        static class Entry extends WeakReference<ThreadLocal<?>>{
            Object value;
        }

        ...

        // 内部是数组⽽不是Map（线性探测法）
        Entry[]	table;

        ...

        // 根据ThreadLocal查找Entry
        private Entry getEntry(ThreadLocal<?> key) {
            // 省略查找逻辑
        }

        ...
    }
}
```

初看上去，我们的设计方案和 Java 的实现仅仅是 Map 的持有方不同而已，我们的设计里面 Map 属于 ThreadLocal，而 Java 的实现里面 ThreadLocalMap 则是属于 Thread。这两种方式哪种更合理呢？
很显然 Java 的实现更合理一些。在 Java 的实现方案里面，ThreadLocal 仅仅是一个代理工具类，内部并不持有任何与线程相关的数据，所有和线程相关的数据都存储在Thread里面，这样的设计容易理解。而从数据的亲缘性上来讲，ThreadLocalMap 属于 Thread 也更加合理。
当然还有一个更加深层次的原因，那就是不容易产生内存泄露。在我们的设计方案中，ThreadLocal 持有的 Map 会持有 Thread 对象的引用，这就意味着，只要 ThreadLocal 对象存在，那么 Map 中的 Thread 对象就永远
不会被回收。ThreadLocal 的生命周期往往都比线程要长，所以这种设计方案很容易导致内存泄露。而 Java 的实现中 Thread 持有 ThreadLocalMap，而且 ThreadLocalMap 里对 ThreadLocal 的引用还是弱引用（WeakReference），所以只要 Thread 对象可以被回收，那么 ThreadLocalMap 就能被回收。Java 的这种实现方案虽然看上去复杂一些，但是更加安全。

### 原理
![ThreadLocal原理图](/images/java/ThreadLocal原理图.webp)

ThreadLocal的原理：每个Thread内部维护着一个ThreadLocalMap，它是一个Map。这个映射表的Key是一个弱引用，其实就是ThreadLocal本身，Value是真正存的线程变量Object。
也就是说ThreadLocal本身并不真正存储线程的变量值，它只是一个工具，用来维护Thread内部的Map，帮助存和取。注意上图的虚线，它代表一个弱引用类型，而弱引用的生命周期只能存活到下次GC前。

## 内存泄漏
Java 的 ThreadLocal 实现应该称得上深思熟虑了，不过即便如此深思熟虑，还是不能百分百地让程序员避免内存泄露，例如在线程池中使用 ThreadLocal，如果不谨慎就可能导致内存泄露。

之所以会内存泄露，是因为在线程池中线程的存活时间太长，往往都是和程序同生共死的，这就意味着 Thread 持有的 ThreadLocalMap 一直都不会被回收，但是 ThreadLocalMap 中的 Entry 对 ThreadLocal 是弱引用（WeakReference），这意味着只要ThreadLocal结束了自己的生命周期是可以被回收掉的（在下次JVM垃圾收集时），但是 Entry 中的 value 却是被 Entry 强引用的，所以即便value的生命周期结束了，value 也是无法被回收的，从而导致内存泄露。

### 基本保证
Java 做了一些措施来保证 ThreadLocal 尽量不会内存泄漏：在 ThreadLocal 的 get()、set()、remove() 方法调用的时候会清除掉线程 ThreadLocalMap 中所有 Entry 中 key 为 null 的 value，并将整个 Entry 设置为 null，利于下次内存回收。
但这样也并不能保证 ThreadLocal 不会发生内存泄漏，例如：
* 使用 static 的 ThreadLocal，延长了 ThreadLocal 的生命周期，可能导致的内存泄漏。
* 分配使用了 ThreadLocal 又不再调用 get()、set()、remove() 方法，那么就会导致内存泄漏。

### 避免措施
每次使用完 ThreadLocal，都调用它的 remove() 方法手动释放对 value的强引用，清除数据。
```java
ExecutorService es;
ThreadLocal tl;
es.execute(() -> {
    // ThreadLocal增加变量
    tl.set(obj);
    try {
    // 省略业务逻辑代码
    } finally {
        // ⼿动清理ThreadLocal
        tl.remove();
    }
});
```

## InheritableThreadLocal与继承性
通过 ThreadLocal 创建的线程变量，其子线程是无法继承的。也就是说我们在线程中通过 ThreadLocal 创建了线程变量 V，而后该线程创建了子线程，那么在子线程中是无法通过 ThreadLocal 来访问父线程的线程变量 V 的。
如果需要子线程继承父线程的线程变量，我们可以使用 InheritableThreadLocal 来支持这种特性，InheritableThreadLocal 是 ThreadLocal 子类，所以用法和 ThreadLocal相同。
不过，完全不建议大家在线程池中使用 InheritableThreadLocal，不仅仅是因为它具有 ThreadLocal 相同的缺点——可能导致内存泄露，更重要的原因是：线程池中线程的创建是动态的，很容易导致继承关系错乱，如果我们的业务逻辑依赖 InheritableThreadLocal，那么很可能导致业务逻辑计算错误，而这个错误往往比内存泄露更要命。