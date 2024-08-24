---
slug:  summary-of-thread-collaboration-methods-in-java
title: Java 线程间协作方法总结
authors: [solidSpoon]
tags: []
---

本文总结一下我知道的 Java 线程间协作的方式，以计算斐波那契数列为例，新线程负责计算，主线程取得结果。

## 不使用多线程并发工具
### 使用循环判断
指定一个变量作为信号，用循环的方式监控这个变量
```java
/**
 * 使用循环不断判断
 */
public class NoLockMethod {
    private volatile Integer valve = null;

    public void sum (int num) {
        valve = fibo(num);
    }

    private int fibo(int a) {
        if (a < 2) {
            return 1;
        }
        return fibo(a - 1) + fibo(a - 2);
    }

    public int getValue() {
        while (valve == null) {}
        return valve;
    }

    public static void main(String[] args) {
        long start = System.currentTimeMillis();
        // 再这里创建一个线程或线城池
        // 异步执行 下面方法

        final NoLockMethod method = new NoLockMethod();
        Thread thread = new Thread(() -> {
            method.sum(45);
        });
        thread.start();
        int result = method.getValue(); // 这是得到的返回值

        // 确保拿到 resut 并输出
        System.out.println("异步计算的结果为：" + result);
        System.out.println("使用时间：" + (System.currentTimeMillis() - start) + " ms");

        // 然后退出 main 线程
    }
}
/*
异步计算的结果为：1836311903
使用时间：6438 ms
 */
```
### 使用 thread.join()
```java
/**
 * 使用 Thread Join
 */
public class ThreadJoinMethod {
    public static void main(String[] args) throws InterruptedException {
        long start = System.currentTimeMillis();
        // 在这里创建一个线程或线程池
        // 异步执行下面方法
        AtomicInteger value = new AtomicInteger();
        Thread thread = new Thread(() ->{
            value.set(sum());
        });
        thread.start();
        thread.join();

        int result = value.get(); // 这是拿到的返回值

        // 确保 拿到 result 并输出
        System.out.println("异步计算结果为：" + result);
        System.out.println("使用时间：" + (System.currentTimeMillis() - start) + " ms");
        // 然后退出 main 线程

    }

    public static int sum () {
        return fibo(45);
    }

    private static int fibo(int a) {
        if (a < 2) {
            return 1;
        }
        return fibo(a - 1) + fibo(a - 2);
    }
}

/*
异步计算结果为：1836311903
使用时间：5413 ms
 */
```
## 使用多线程并发工具
### 不使用 Future（使用类似等待-通知机制）
#### Synchronized-wait-notify
```java
/**
 * 通过的管程等待-通知机制，来获取值
 * wait() notify()
 */
public class SynchronizedMethod {
    private volatile Integer value = null;

    synchronized public void sum (int num) {
        value = fibo(num);
        notifyAll();
    }

    private int fibo(int a) {
        if ( a < 2) {
            return 1;
        }
        return fibo(a-1) + fibo(a-2);
    }

    synchronized public int getValue() throws InterruptedException {
        while (value == null) {
            wait();
        }
        return value;
    }

    public static void main(String[] args) throws InterruptedException {
        long start = System.currentTimeMillis();
        // 在这里创建一个线程或线程池
        // 异步执行下面方法
        final SynchronizedMethod method = new SynchronizedMethod();
        Thread thread = new Thread(() -> {
            method.sum(45);
        });
        thread.start();

        int result = method.getValue();// 这是得到的返回值

        // 确保拿到 result 并输出
        System.out.println("异步计算的结果为：" + result);
        System.out.println("使用时间：" + (System.currentTimeMillis() - start) + " ms");
    }
}
/*
异步计算的结果为：1836311903
使用时间：5198 ms
 */

```
#### Semaphore
```java
/**
 * Semaphore 方式
 */
public class SemaphoreMethod {
    private volatile Integer value = null;
    final Semaphore semaphore = new Semaphore(1);

    public void sum(int num) throws InterruptedException {
        this.value = fibo(num);
        semaphore.release();
    }

    private int fibo(int a) {
        if (a < 2) {
            return 1;
        }
        return fibo(a - 1) + fibo(a - 2);
    }

    public int getValue() throws InterruptedException {
        int result;
        semaphore.acquire();
        result = this.value;
        semaphore.release();
        return result;
    }

    public static void main(String[] args) throws InterruptedException {
        long start = System.currentTimeMillis();
        // 在这里创建一个线程或线程池
        // 异步执行下面方法

        final SemaphoreMethod method = new SemaphoreMethod();
        method.semaphore.acquire();
        Thread thread = new Thread(() -> {
            try {
                method.sum(45);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });
        thread.start();
        int result = method.getValue();
        System.out.println("异步计算结果为" + result);
        System.out.println("使用时间：" + (System.currentTimeMillis() - start) + " ms");
        // 然后退出 main 线程
    }
}
```
#### Lock-Condition
```java
public class LockConditionMethod {
    private volatile Integer value = null;
    private Lock lock = new ReentrantLock();
    private Condition calComplete = lock.newCondition();

    public void sum(int num) {
        lock.lock();
        value = fibo(num);
        calComplete.signal();
        lock.unlock();
    }

    private int fibo(int a) {
        if (a < 2) {
            return 1;
        }
        return fibo(a - 1) + fibo(a - 2);
    }

    public int getValue() {
        lock.lock();
        while (value == null) {
            try {
                calComplete.await();
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                lock.unlock();
            }
        }
        return value;
    }

    public static void main(String[] args) {
        long start = System.currentTimeMillis();
        // 在这里创建一个线程或线程池
        // 异步执行下面方法
        final LockConditionMethod method = new LockConditionMethod();
        Thread thread = new Thread(() -> {
            method.sum(45);
        });
        thread.start();
        int result = method.getValue(); // 这是得到的返回值

        // 确保拿到 result 并输出
        System.out.println("异步计算的结果为：" + result);
        System.out.println("使用时间：" + (System.currentTimeMillis() - start) + " ms");
    }
}
/*
异步计算的结果为：1836311903
使用时间：5402 ms
 */
```
#### CyclicBarrier
```java
/**
 * CyclicBarrierMethod 方式
 */
public class CyclicBarrierMethod {
    private  volatile Integer value = null;
    CyclicBarrier barrier;

    public void setBarrier(CyclicBarrier barrier) {
        this.barrier = barrier;
    }

    public void sum(int num) throws BrokenBarrierException, InterruptedException {
        value = fibo(num);
        System.out.println(barrier.await());
    }

    private int fibo(int a) {
        if(a < 2) {
            return 1;
        }
        return fibo(a-1) + fibo(a-2);
    }

    public int getValue(){
        return value;
    }

    public static void main(String[] args) {
        long start = System.currentTimeMillis();
        final CyclicBarrierMethod method = new CyclicBarrierMethod();
        CyclicBarrier barrier = new CyclicBarrier(1, () -> {
            int result = 0; // 这是得到的反回值
            result = method.getValue();

            // 确保拿到 result 并输出
            System.out.println("异步计算结果为：" + result);
            System.out.println("使用时间为：" + (System.currentTimeMillis() - start) + " ms");
        });
        method.setBarrier(barrier);

        Thread thread = new Thread(() -> {
            try {
                method.sum(45);
            } catch (BrokenBarrierException e) {
                e.printStackTrace();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });
        thread.start();

        // 然后退出 main 线程
    }
}
```
#### CountDownLatch
```java
public class CountDownLatchMethod {
    private volatile Integer value = null;
    private CountDownLatch latch;

    public void sum(int num) {
        value = fibo(num);
        latch.countDown();
    }

    private int fibo(int a) {
        if (a < 2) {
            return 1;
        }
        return fibo(a - 1)  + fibo(a - 2);
    }

    private int getValue() throws InterruptedException {
        latch.await();
        return value;
    }

    /**
     * latch 没有重置功能，这个函数用来传入新的
     * @param latch
     */
    public void setLatch(CountDownLatch latch) {
        this.latch = latch;
    }

    public static void main(String[] args) throws InterruptedException {
        long start = System.currentTimeMillis();
        // 在这里创建一个线程或线程池
        // 异步执行下面方法
        CountDownLatch latch = new CountDownLatch(1);
        final CountDownLatchMethod method = new CountDownLatchMethod();
        method.setLatch(latch);
        Thread thread = new Thread(() ->{
            method.sum(45);
        });
        thread.start();
        int result = method.getValue(); // 这是得到的返回值

        // 确保 拿到 result 并输出
        System.out.println("异步计算结果为：" + result);
        System.out.println("使用时间：" + (System.currentTimeMillis() - start) + " ms");
        // 然后退出 main 线程
    }
}
/*
异步计算结果为：1836311903
使用时间：5318 ms
 */
```
### 使用 Future（使用线程池的 submit）
#### Future
```java
/**
 * Future
 */
public class FutureMethod implements Callable {

    private long sum(int num) {
        return fibo(num);
    }

    private int fibo(int a) {
        if (a < 2) {
            return 1;
        }
        return fibo(a - 1) + fibo(a - 2);
    }

    @Override
    public Object call() throws Exception {
        return sum(45);
    }

    public static void main(String[] args) throws ExecutionException, InterruptedException {
        long start = System.currentTimeMillis();
        // 在这里创建一个线程或线程池
        // 异步执行 下面方法
        ExecutorService executorService = Executors.newFixedThreadPool(1);
        Future<Long> future = executorService.submit(new FutureMethod());
        long result = future.get(); // 这是得到的返回值i
        // 确保 拿到 result 并输出
        System.out.println("异步计算结果为：" + result);
        System.out.println("使用时间：" + (System.currentTimeMillis() - start) + "ms");
        // 然后退出 main 线程
        executorService.shutdown();
    }
}
/*
异步计算结果为：1836311903
使用时间：5277ms
 */
```
#### FutureTask
```java
/**
 * FutureTask
 */
public class FutureTaskMethod {

    /**
     * 取结果
     */
    static class Get implements Callable<Integer> {
        FutureTask<Integer> sum;

        public Get(FutureTask<Integer> sum) {
            this.sum = sum;
        }

        @Override
        public Integer call() throws Exception {
            return sum.get();
        }
    }

    /**
     * 求结果
     */
    static class Sum implements Callable<Integer> {
        @Override
        public Integer call() throws Exception {
            return fibo(45);
        }

        private int fibo(int a) {
            if (a < 2) {
                return 1;
            }
            return fibo(a - 1) + fibo(a - 2);
        }
    }

    public static void main(String[] args) throws ExecutionException, InterruptedException {
        long start = System.currentTimeMillis();
        // 在这里创建一个线程或线程池
        // 异步执行下面方法
        FutureTask<Integer> sum = new FutureTask<>(new Sum());
        FutureTask<Integer> get = new FutureTask<>(new Get(sum));

        Thread sumT = new Thread(sum);
        sumT.start();
        Thread getT = new Thread(get);
        getT.start();

        int result = get.get(); // 这是得到的返回值

        // 确保拿到 result 并输出
        System.out.println("异步计算的结果为：" + result);
        System.out.println("使用时间：" + (System.currentTimeMillis() - start) + " ms");

        // 然后退出 main 线程
    }
}
```
`call()` 没有输入参数，所以用两个 `call()` ，一个用来指定固定的输入参数，令一个用来获取结果。
#### CompletableFuture
```java
/**
 * CompletableFuture 方式
 */
public class CompleteableMethod {
    public static void main(String[] args) {
        long start = System.currentTimeMillis();
        // 在这里创建一个线程或线程池
        // 异步执行下面方法

        int result = CompletableFuture.supplyAsync(() -> sum()).join();

        // 确保 拿到 result 并输出
        System.out.println("异步计算的结果为：" + result);
        System.out.println("使用时间：" + (System.currentTimeMillis() - start) + " ms");
    }

    private static int sum() {
        return fibo(45);
    }
    private static int fibo(int a){
        if (a < 2) {
            return 1;
        }
        return fibo(a - 1) + fibo(a - 2);
    }
}
```