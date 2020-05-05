---
title: 并发设计模式--Copy-on-Write模式
date: 2019-10-22 09:38:59
categories: Java并发
---
## 概念
所谓 Copy-on-Write，经常被缩写为 COW 或者 CoW，顾名思义就是写时复制。不可变对象的写操作往往都是使用 Copy-on-Write 方法解决的，当然Copy-on-Write的应用领域并不局限于 Immutability 模式。

## 应用实例
### Java集合
Java 中的 CopyOnWriteArrayList 和 CopyOnWriteArraySet 这两个 Copy-on-Write 容器，它们背后的设计思想就是 Copy-on-Write。

### 操作系统
类 Unix 的操作系统中创建进程的 API 是 fork()，传统的 fork() 函数会创建父进程的一个完整副本，例如父进程的地址空间现在用到了 1G 的内存，那么 fork() 子进程的时候要复制父进程整个进程的地址空间（占有 1G 内存）给子进程，这个过程是很耗时的。而 Linux 中的 fork() 函数就聪明得多了，fork() 子进程的时候，并不复制整个进程的地址空间，而是让父子进程共享同一个地址空间；只用在父进程或者子进程需要写入的时候才会复制地址空间，从而使父子进程拥有各自的地址空间。本质上来讲，父子进程的地址空间以及数据都是要隔离的，使用 Copy-on-Write 更多地体现的是一种延时策略，只有在真正需要复制的时候才复制，而不是提前复制好，同时 Copy-on-Write 还支持按需复制，所以 Copy-on-Write 在操作系统领域是能够提升性能的。相比较而言，Java 提供的 Copy-on-Write 容器，由于在修改的同时会复制整个容器，所以在提升读操作性能的同时，是以内存复制为代价的。

### Docker容器镜像

### 分布式源码管理系统Git

### 函数式编程
Copy-on-Write 最大的应用领域还是在函数式编程领域。函数式编程的基础是不可变性 （Immutability），所以函数式编程里面所有的修改操作都需要 Copy-on-Write 来解决。

### 路由动态更新
```java
// 路由信息
public final class Router{
    private final String  ip;
    private final Integer port;
    private final String  iface;

    // 构造函数
    public Router(String ip, Integer port, String iface) {
        this.ip = ip;
        this.port = port;
        this.iface = iface;
    }

    // 重写equals⽅法
    public boolean equals(Object obj) {
        if (obj instanceof Router) {
            Router r = (Router)obj;
            return iface.equals(r.iface) && ip.equals(r.ip) && port.equals(r.port);
        }
        return false;
    }

    public int hashCode() {
        //省略hashCode相关代码
    }
}

// 路由表信息
public class RouterTable {
    // Key:接⼝名
    // Value:路由集合
    ConcurrentHashMap<String, CopyOnWriteArraySet<Router>> rt = new ConcurrentHashMap<>();
    
    // 根据接⼝名获取路由表
    public Set<Router> get(String iface) {
        return rt.get(iface);
    }
    
    // 删除路由
    public void remove(Router router) {
        Set<Router> set=rt.get(router.iface);
        if (set != null) {
            set.remove(router);
        }
    }
    
    // 增加路由
    public void add(Router router) {
        Set<Router> set = rt.computeIfAbsent(route.iface, r -> new CopyOnWriteArraySet<>());
        set.add(router);
    }
}
```