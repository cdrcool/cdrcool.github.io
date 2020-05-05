---
title: 并发设计模式---Balking模式
date: 2019-10-22 20:51:10
categories: Java并发
---
## 概念
如果现在不适合执行这个操作，或者没必要执行这个操作，就停止处理，直接返回。
Balking 模式与 Guarded Suspension 模式一样，也存在守护条件。在 Balking 模式中，如果守护条件不成立，则立即中断处理。这与 Guarded Suspension 模式有所不同，因为 Guarded Suspension 模式是一直等待至可以运行。

## 应用示例
### 文档编辑
```java
boolean changed = false;

// ⾃动存盘操作
void autoSave() {
    synchronized(this) {
        if (!changed) {
            return;
        }
        changed = false;
    }

    // 执⾏存盘操作
    this.execSave();
}

// 编辑操作
void edit() {
    // 省略编辑逻辑
    ......
    change();
}

// 改变状态
void change() {
    synchronized(this) {
        changed = true;
    }
}
```

## 总结
Balking 模式和 Guarded Suspension 模式从实现上看似乎没有多大的关系，Balking 模式只需要用互斥锁就能解决，而 Guarded Suspension 模式则要用到管程这种高级的并发原语；但是从应用的角度来看，它们解决的都是“线程安全的if”语义，不同之处在于，Guarded Suspension 模式会等待 if 条件为真，而 Balking 模式不会等待。
Balking 模式的经典实现是使用互斥锁，我们可以使用 Java 语言的内置 synchronized，也可以使用 SDK 提供 Lock；如果我们对互斥锁的性能不满意，可以尝试采用 volatile 方案，不过使用 volatile 方案需要我们更加谨慎。当然我们也可以尝试使用双重检查方案来优化性能，双重检查中的第一次检查，完全是出于对性能的考量：避免执行加锁操作，因为加锁操作很耗时。而加锁之后的二次检查，则是出于对安全性负责。