---
title: 并发设计模式--GuardedSuspension模式
date: 2019-10-23 20:13:00
categories: Java并发
---
## 概念
所谓 Guarded Suspension，直译过来就是“保护性地暂停”，如果执行现在的处理会造成问题，就让执行处理的线程进行等待，这种模式通过让线程等待来保证实例的安全性。

比如，项目组团建要外出聚餐，我们提前预订了一个包间，然后兴冲冲地奔过去，到那儿后大堂经理看了一眼包间，发现服务员正在收拾，就会告诉我们：“您预订的包间服务员正在收拾，请您稍等片刻。”过了一会，大堂经理发现包间已经收拾完了，于是马上带我们去包间就餐。

## 结构图
下图就是 Guarded Suspension 模式的结构图，非常简单，一个对象 GuardedObject ，内部有一个成员变量——受保护的对象，以及两个成员方法 —— get(Predicate<T> p) 和 onChanged(T obj) 方法。

![Guarded Suspension模式结构图](/images/java/Guarded Suspension模式结构图.png)

其中，对象 GuardedObject 就是我们前面提到的大堂经理，受保护对象就是餐厅里面的包间；受保护对象的 get() 方法对应的是我们的就餐，就餐的前提条件是包间已经收拾好了，参数 p 就是用来描述这个前提条件的；受保护对象的 onChanged() 方法对应的是服务员把包间收拾好了，通过 onChanged() 方法可以 fire 一个事件，而这个事件往往能改变前提条件 p 的计算结果。下图中，左侧的绿色线程就是需要就餐的顾客，而右侧的蓝色线程就是收拾包间的服务员。

## 模板代码
GuardedObject 的内部实现非常简单，是管程的一个经典用法，核心是：get() 方法通过条件变量的 await() 方法实现等待，onChanged() 方法通过条件变量的 signalAll() 方法实现唤醒功能。

```java
class GuardedObject<T> {
    // 受保护的对象
    T obj;
    final Lock lock = new ReentrantLock();
    final Condition done = lock.newCondition();
    final int timeout = 1;

    // 获取受保护对象  
    T get(Predicate<T> p) {
        lock.lock();
        try {
            // MESA管程推荐写法
            while(!p.test(obj)) {
                done.await(timeout, 
                TimeUnit.SECONDS);
                }
            }
        catch(InterruptedException e) {
            throw new RuntimeException(e);
        }finally{
            lock.unlock();
        }
        // 返回⾮空的受保护对象
        return obj;
    }

    // 事件通知⽅法
    void onChanged(T obj) {
        lock.lock();
        try {
            this.obj = obj;
            done.signalAll();
        } finally {
            lock.unlock();
        }
    }
}
```

## 扩展
Guarded	Suspension 模式本质上是一种等待唤醒机制的实现，只不过 Guarded Suspension 模式将其规范化了。规范化的好处是你无需重头思考如何实现，也无需担心实现程序的可理解性问题，同时也能避免一不小心写出个 Bug 来。但 Guarded Suspension 模式在解决实际问题的时候，往往还是需要扩展的。下面以**Web请求异步响应**为例演示如何扩展。

```java
class GuardedObject<T> {
    //受保护的对象
    T obj;
    final Lock lock = new ReentrantLock();
    final Condition done = lock.newCondition();
    final int timeout = 2;

    // 保存所有GuardedObject
    final static Map<Object, GuardedObject> gos = new ConcurrentHashMap<>();

    // 静态⽅法创建GuardedObject
    static <K> GuardedObject create(K key) {
        GuardedObject go=new GuardedObject();
        gos.put(key, go);
        return go;
    }

    static <K, T> void fireEvent(K key, T obj) {
        GuardedObject go=gos.remove(key);
        if (go != null){
            go.onChanged(obj);
        }
    }

    // 获取受保护对象
    T get(Predicate<T> p) {
        lock.lock();
        try {
            // MESA管程推荐写法
            while(!p.test(obj)) {
                done.await(timeout, TimeUnit.SECONDS);
            }
        } catch(InterruptedException e) {
            throw new RuntimeException(e);
        } finally {
            lock.unlock();
        }
        //返回⾮空的受保护对象
        return obj;
    }

    // 事件通知⽅法
    void onChanged(T obj) {
        lock.lock();
        try {
            this.obj = obj;
            done.signalAll();
        } finally {
            lock.unlock();
        }
    }
}

// 处理浏览器发来的请求
Respond handleWebReq() {
    int id = 序号⽣成器.get();
    // 创建⼀消息
    Message msg1 = new Message(id, "{...}");
    // 创建GuardedObject实例
    GuardedObject<Message> go = GuardedObject.create(id);
 
    // 发送消息
    send(msg1);

    // 等待MQ消息
    Message r = go.get(t->t != null);  
}

void onMessage(Message msg) {
    // 唤醒等待的线程
    GuardedObject.fireEvent(msg.id, msg);
}
```
