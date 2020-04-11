---
title: JVM 性能监控与调优
date: 2020-04-06 13:44:00
categories: JVM
---
## JVM 参数
### 标准参数
在 JVM 各版本中基本不变，相对稳定。

* -help
* -server -client
* -version -showversion
* -cp -classpath

### X 参数
在 JVM 各版本中可能会变，变化较小。

* -Xint：解释执行
* -Xcomp：第一次使用就编译成本地代码
* -Xmixed：混合模式，JVM 自己来决定是否编译成本地代码

### XX 参数
相对不稳定，主要用于 JVM 调优和 Debug。

Boolean 类型：
格式：-XX:[+-]<name>，标识启用或者禁用 name 属性。

非 Boolean 类型：
格式：-XX:<name>=<value>，表示 name 属性的值是 value。

* -XX:+UserConcMarkSweepGC
* -XX:+UseG1GC

* -XX:MaxGCPauseMillis
* -XX:GCTimeRatio
* -Xms：等价于 -XX:InitialHeapSize，表示 JVM 启动时分配的内存。
* -Xmx：等价于 -XX:MaxHeapSize，表示 JVM 运行过程中分配的最大内存。
* -Xss：等价于 -XX:ThreadStackSize，表示 JVM 启动的每个线程分配的内存大小。

* -XX:+HeapDumpOnOutOfMemoryError
* -XX:HeapDumpPath

## jps
```bash
jps
```

## jstat
### 类加载
```bash
jstat -class pid

Loaded  Bytes  Unloaded  Bytes     Time
 16136 28826.9        1     0.9      36.67
```

### 垃圾收集
```bash
jstat -gc 17204

S0C    S1C    S0U    S1U      EC       EU        OC         OU       MC     MU    CCSC   CCSU   YGC     YGCT    FGC    FGCT     GCT
27136.0 20992.0  0.0   20985.8 330240.0 105111.7  139264.0   41337.2   82688.0 77595.2 11776.0 10855.5     19    1.042   3      0.528    1.570
```

* S0C、S1C、S0U、S1U：S0 和 S1 的总量与使用量
* EC、EU：Eden 区总量与使用量
* OC、OU：Old 区总量与使用量
* MC、MU：Metaspace 区总量与使用量
* CCSC、CCSU：压缩类空间总量与使用量
* YGC、YGCT：YoungGC 的次数与时间
* FGC、FGCT：FullGC 的次数与实践
* GCT：总的 GC 时间

### JIT 编译
```bash
jstat -compiler 17204

Compiled Failed Invalid   Time   FailedType FailedMethod
    8432      3       0     4.32          1 org/apache/http/client/utils/URLEncodedUtils parse
```

## jmap
```bash
jmap -dump:format=b,file=heap.hprof pid
```

```bash
jmap -heap pid
```

## jstat
```bash
jstack pid
```

## jvisualvm

## BTrace