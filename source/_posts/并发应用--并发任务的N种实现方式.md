---
title: 并发应用--并发任务的N种实现方式
date: 2019-09-18 22:16:54
categories: Java并发
---
日常开发中，可能会遇到某个业务方法中需要调用多个 rpc 接口，这时就可以考虑利用多线程技术提高执行效率。

## Thread.join
其作用是调用线程等待子线程完成后,才能继续用下运行。

示例：
```java
@Test
public void joinTest() throws InterruptedException {
    // 存放任务及其执行结果映射
    Map<String, String> taskResults = new HashMap<>();

    // 模拟任务1
    Runnable task1 = () -> {
        // 模拟任务执行
        try {
            System.out.println("任务1执行中");
            Thread.sleep(200);
            taskResults.put("task1", "result1");
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    };

    // 模拟任务2
    Runnable task2 = () -> {
        // 模拟任务执行
        try {
            System.out.println("任务2执行中");
            Thread.sleep(200);
            taskResults.put("task2", "result2");
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    };

    // 模拟任务3
    Runnable task3 = () -> {
        // 模拟任务执行
        try {
            System.out.println("任务3执行中");
            Thread.sleep(200);
            taskResults.put("task3", "result3");
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    };

    Thread thread1 = new Thread(task1);
    Thread thread2 = new Thread(task2);
    Thread thread3 = new Thread(task3);

    thread1.start();
    thread2.start();
    thread3.start();

    // 主线程等待子线程执行完毕
    thread1.join();
    thread2.join();
    thread3.join();

    System.out.println("所有任务已执行结束，任务结果：" + taskResults.toString());
}
```

输出：
```text
任务2执行中
任务1执行中
任务3执行中
所有任务已执行结束，任务结果：{task1=result1, task2=result2, task3=result3}
```

缺点：
要调用 Thread.join() 方法，就得显示创建线程，从而不能利用线程池以减少资源的消耗。

## Thread.invokeAll
其作用是触发执行任务列表，返回的结果顺序与任务在任务列表中的顺序一致。所有线程执行完任务后才返回结果。如果设置了超时时间，未超时完成则正常返回结果，如果超时未完成则报异常。

示例：
```java
@Test
public void invokeAllTest() throws InterruptedException, ExecutionException {
    ExecutorService executor = Executors.newFixedThreadPool(5);

    // 模拟任务1
    Callable<String> task1 = () -> {
        // 模拟任务执行
        System.out.println("任务1执行中");
        Thread.sleep(200);
        return "result1";
    };

    // 模拟任务2
    Callable<String> task2 = () -> {
        // 模拟任务执行
        System.out.println("任务2执行中");
        Thread.sleep(200);
        return "result2";
    };

    // 模拟任务3
    Callable<String> task3 = () -> {
        // 模拟任务执行
        System.out.println("任务3执行中");
        Thread.sleep(200);
        return "result3";
    };

    // 同时提交多个任务，并等任务都执行完毕后返回
    List<Future<String>> futures = executor.invokeAll(Arrays.asList(task1, task2, task3));

    for (Future future : futures) {
        System.out.println(future.get());
    }
}
```

输出：
```text
任务2执行中
任务1执行中
任务3执行中
result1
result2
result3
```

观察"do work"输出可知线程是并行执行的，观察"result"输出可知future结果是顺序序输出的。虽然输出与前面的**FutureTask**方式一样，但效率它会高些，因为`FutureTask`的`get`方法会阻塞后面的操作。

## FutureTask
`FutureTask`适用于异步获取执行结果或取消执行任务的场景。
由于它同时实现了`Runnable`和`Future`接口，因此它既可以作为`Runnable`被线程执行，又可以作为`Future`得到`Callable`的返回值。

示例：
```java
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

输出：
```text
任务2执行中
任务1执行中
任务3执行中
result1
result2
result3
```

缺点：
由于调用 Future.get() 操作会阻塞当前线程，这样假如 task1 任务执行的时间很长，那么即使其他 tasks 都已执行完毕，我们也得等待 task 执行完，从而不能优先处理最先完成的任务结果。

## CountDownLatch
`CountDownLatch`，通常称之为闭锁，它允许一个或多个线程等待，直到在其他线程中执行的一组操作完成。

示例：
```java
@Test
public void countDownLatchTest() throws InterruptedException {
    // 创建线程池
    ExecutorService executor = Executors.newFixedThreadPool(5);

    // 存放任务及其执行结果映射
    Map<String, String> taskResults = new HashMap<>();

    // 创建闭锁：当count减为0时，线程才会继续往下执行
    CountDownLatch countDownLatch = new CountDownLatch(3);

    // 模拟任务1
    Runnable task1 = () -> {
        // 模拟任务执行
        try {
            System.out.println("任务1执行中");
            Thread.sleep(200);
            taskResults.put("task1", "result1");
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        countDownLatch.countDown();
    };

    // 模拟任务2
    Runnable task2 = () -> {
        // 模拟任务执行
        try {
            System.out.println("任务2执行中");
            Thread.sleep(200);
            taskResults.put("task2", "result2");
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        countDownLatch.countDown();
    };

    // 模拟任务3
    Runnable task3 = () -> {
        // 模拟任务执行
        try {
            System.out.println("任务3执行中");
            Thread.sleep(200);
            taskResults.put("task3", "result3");
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        countDownLatch.countDown();
    };

    // 提交任务1
    executor.execute(task1);
    // 提交任务2
    executor.execute(task2);
    // 提交任务3
    executor.execute(task3);

    // 等待所有任务执行完毕
    countDownLatch.await();

    System.out.println("所有任务已执行结束，任务结果：" + taskResults.toString());
}
```

输出：
```text
任务2执行中
任务1执行中
任务3执行中
所有任务已执行结束，任务结果：{task1=result1, task2=result2, task3=result3}
```

## CyclicBarrier
`CyclicBarrier`，即可重用屏障/栅栏，它允许一组线程彼此等待到达一个共同的障碍点，并可以设置线程都到达障碍点后要执行的命令。

示例：
```java
@Test
public void cyclicBarrierTest() throws InterruptedException, BrokenBarrierException {
    // 存放任务及其执行结果映射
    Map<String, String> taskResults = new HashMap<>();

    // 创建栅栏：等待指定数目的任务都执行完后，输出所有任务执行结果
    CyclicBarrier barrier = new CyclicBarrier(4, () -> {
        System.out.println("所有任务已执行结束，任务结果：" + taskResults.toString());
    });

    // 模拟任务1
    Runnable task1 = () -> {
        // 模拟任务执行
        try {
            System.out.println("任务1执行中");
            Thread.sleep(200);
            taskResults.put("task1", "result1");
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        try {
            // 执行完成，等待屏障
            barrier.await();
        } catch (InterruptedException | BrokenBarrierException e) {
            e.printStackTrace();
        }
    };

    // 模拟任务2
    Runnable task2 = () -> {
        // 模拟任务执行
        try {
            System.out.println("任务2执行中");
            Thread.sleep(200);
            taskResults.put("task2", "result2");
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        try {
            // 执行完成，等待屏障
            barrier.await();
        } catch (InterruptedException | BrokenBarrierException e) {
            e.printStackTrace();
        }
    };

    // 模拟任务3
    Runnable task3 = () -> {
        // 模拟任务执行
        try {
            System.out.println("任务3执行中");
            Thread.sleep(200);
            taskResults.put("task3", "result3");
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        try {
            // 执行完成，等待屏障
            barrier.await();
        } catch (InterruptedException | BrokenBarrierException e) {
            e.printStackTrace();
        }
    };

    // 提交任务1
    new Thread(task1).start();
    // 提交任务2
    new Thread(task2).start();
    // 提交任务3
    new Thread(task3).start();

    // 主线程等待屏障
    barrier.await();
}
```

输出：
```text
任务2执行中
任务1执行中
任务3执行中
所有任务已执行结束，任务结果：{task1=result1, task2=result2, task3=result3}
```

## CompletionService
`CompletionService`适用于异步获取并行任务执行结果。
使用`Future`和`Callable`可以获取线程执行结果，但获取方式确是阻塞的，根据添加到线程池中的线程顺序，依次获取，获取不到就阻塞。`CompletionService`采用轮询的方式，可以做到异步非阻塞获取执行结果。

示例：
```java
@Test
public void completionServiceTest() {
    ExecutorService executor = Executors.newFixedThreadPool(5);
    CompletionService<String> completionService = new ExecutorCompletionService<>(executor);

    // 模拟任务1
    Callable<String> task1 = () -> {
        // 模拟任务执行
        System.out.println("任务1执行中");
        Thread.sleep(200);
        return "result1";
    };

    // 模拟任务2
    Callable<String> task2 = () -> {
        // 模拟任务执行
        System.out.println("任务2执行中");
        Thread.sleep(200);
        return "result2";
    };

    // 模拟任务3
    Callable<String> task3 = () -> {
        // 模拟任务执行
        System.out.println("任务3执行中");
        Thread.sleep(200);
        return "result3";
    };

    // 提交任务1
    completionService.submit(task1);
    // 提交任务2
    completionService.submit(task2);
    // 提交任务3
    completionService.submit(task3);

    // 获取任务结果
    IntStream.range(0, 3).forEach(index -> {
        try {
            // 按任务执行完毕的顺序，获取任务结果
            Future<String> future = completionService.take();
            System.out.println(future.get());
        } catch (InterruptedException | ExecutionException e) {
            e.printStackTrace();
        }
    });
}
```

输出：
```text
任务3执行中
任务1执行中
任务2执行中
result3
result2
result1
```

## CompletableFuture
在Java8中，`CompletableFuture`提供了非常强大的`Future`的扩展功能，可以帮助我们简化异步编程的复杂性，并且提供了函数式编程的能力，可以通过回调的方式处理计算结果，也提供了转换和组合`CompletableFuture`的方法。

示例：
```java
@Test
public void completableFutureTest() {
    /*// 创建线程池
    ExecutorService executor = Executors.newFixedThreadPool(10);

    // 模拟任务1
    CompletableFuture<String> future1 = CompletableFuture.supplyAsync(() -> {
                // 模拟任务执行
                try {
                    System.out.println("任务1执行中");
                    Thread.sleep(200);
                    System.out.println("任务1执行完毕");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                return "result1";
            }, executor
    );

    // 模拟任务2
    CompletableFuture<String> future2 = CompletableFuture.supplyAsync(() -> {
                // 模拟任务执行
                try {
                    System.out.println("任务2执行中");
                    Thread.sleep(200);
                    System.out.println("任务2执行完毕");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                return "result2";
            }, executor
    );

    // 模拟任务3
    CompletableFuture<String> future3 = CompletableFuture.supplyAsync(() -> {
                // 模拟任务执行
                try {
                    System.out.println("任务3执行中");
                    Thread.sleep(200);
                    System.out.println("任务3执行完毕");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                return "result3";
            }, executor
    );

    List<String> results = Stream.of(future1, future2, future3)
            .map(CompletableFuture::join)
            .collect(Collectors.toList());
    System.out.println("所有任务已执行结束，任务结果：" + results.toString());*/

    // 创建线程池
    ExecutorService executor = Executors.newFixedThreadPool(10);

    Map<String, String> results = new HashMap<>();

    // 模拟任务1
    CompletableFuture<String> future1 = CompletableFuture.supplyAsync(() -> {
                // 模拟任务执行
                try {
                    System.out.println("任务1执行中");
                    Thread.sleep(200);
                    System.out.println("任务1执行完毕");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                return "result1";
            }, executor
    ).whenComplete((v, e) -> results.put("task1", v));

    // 模拟任务2
    CompletableFuture<String> future2 = CompletableFuture.supplyAsync(() -> {
                // 模拟任务执行
                try {
                    System.out.println("任务2执行中");
                    Thread.sleep(200);
                    System.out.println("任务2执行完毕");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                return "result2";
            }, executor
    ).whenComplete((v, e) -> results.put("task2", v));

    // 模拟任务3
    CompletableFuture<String> future3 = CompletableFuture.supplyAsync(() -> {
                // 模拟任务执行
                try {
                    System.out.println("任务3执行中");
                    Thread.sleep(200);
                    System.out.println("任务3执行完毕");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                return "result3";
            }, executor
    ).whenComplete((v, e) -> results.put("task3", v));

    CompletableFuture.allOf(future1, future2, future3)
            .thenRun(() -> System.out.println("所有任务已执行结束，任务结果：" + results.toString()))
            .join();
}
```

输出：
```text
任务2执行中
任务1执行中
任务3执行中
任务2执行完毕
任务1执行完毕
任务3执行完毕
所有任务已执行结束，任务结果：[result1, result2, result3]
```

## ParallelStream
对于计算密集型任务，可以使用`ParallelStream`，其底层使用Fork/Join框架实现。

示例：
```java
@Test
public void forkJoinTest() {
    IntStream.range(0, 10).parallel().forEach((index) -> System.out.println(String.format("任务%s执行中", index)));

    List<String> results = IntStream.range(0, 10).parallel()
            .mapToObj((index) -> {
                System.out.println(String.format("任务%s执行中", index));
                return String.format("result%s", index);
            }).collect(Collectors.toList());

    System.out.println("所有任务已执行结束，任务结果：" + results.toString());
}
```

输出：
```text
任务6执行中
任务5执行中
任务2执行中
任务4执行中
任务1执行中
任务3执行中
任务0执行中
任务8执行中
任务7执行中
任务9执行中
任务6执行中
任务1执行中
任务2执行中
任务4执行中
任务3执行中
任务8执行中
任务0执行中
任务7执行中
任务5执行中
任务9执行中
所有任务已执行结束，任务结果：[result0, result1, result2, result3, result4, result5, result6, result7, result8, result9]
```

缺点：
默认情况下所有的并行流计算都共享一个 ForkJoinPool，这个共享的 ForkJoinPool 默认的线程数是 CPU 的核数；如果所有的并行流计算都是 CPU 密集型计算的话，完全没有问题，但是如果存在 I/O 密集型的并行流计算，那么很可能会因为一个很慢的 I/O 计算而拖慢整个系统的性能。所以建议用不同的 ForkJoinPool 执行不同类型的计算任务。
