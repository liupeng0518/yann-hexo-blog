---
title: 通过代码学习 - Java中的Synchronized
date: 2018-03-28 15:44:58
tags:
---

## Synchronized 关键字

#### 无Synchronized
Synchronized是互斥锁，非常接近底层Mux的概念，就是把内存的一块区间给锁住，只允许一个线程使用。

<!--more-->

```java
class Sender {
    private int count = 0;

    public void printIncrease() {
        count = count + 1;  //这里注意不能为 count++；这是一个原子性操作
        System.out.println(Thread.currentThread().toString() + ":" + "count is " + count);
    }
}
```
我们首先创建一个Sender类，我们声明一个printIncrease函数，注意此时没有 synchronized 关键字。我们再构建一个测试Demo,模拟50个线程同时运行。

```java
public class Synchronization {

    public static void main(String[] args) {
        Sender s = new Sender();
        List<Thread> runnables = new ArrayList<>();


        for(int i = 0; i<50;i++){
            runnables.add(new Thread(s::printIncrease));
        }

        runnables.forEach(Thread::start);
        runnables.forEach((r) -> {
            try {
                r.join();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });

    }
}
```

理论上我们最后一次的打印出来的Counter应该是49，因为我们运行了50次。但是在本机的结果是如下图（根据机器都不太一样）

```bash
Thread[Thread-44,5,main]:count is 44
Thread[Thread-45,5,main]:count is 44
Thread[Thread-46,5,main]:count is 46
Thread[Thread-48,5,main]:count is 47
Thread[Thread-49,5,main]:count is 47
Thread[Thread-47,5,main]:count is 47
```
### Synchronized方式
那我们此时加上 synchronized

```java
class Sender {
    private int count = 0;

    public synchronized void printIncrease() {
         count = count + 1;
         System.out.println(Thread.currentThread().toString() + ":" + "count is " + count);
    }
}
```

结果如图如下:

```bash
Thread[Thread-47,5,main]:count is 47
Thread[Thread-48,5,main]:count is 49
Thread[Thread-49,5,main]:count is 50
```

我们可以看出来synchronized第一特性就是同步函数的特性。

### Synchronized 属性
我们换一种方式，我们synchronized 这个 count 属性，结果呢？

```java
class Sender {
    private Integer count = 0;

    public void printIncrease() {
        synchronized (count) {
            count = count + 1;
        }
        System.out.println(Thread.currentThread().toString() + ":" + "count is " + count);
    }
}
```

```bash
Thread[Thread-1,5,main]:count is 0
Thread[Thread-4,5,main]:count is 0
Thread[Thread-3,5,main]:count is 0
Thread[Thread-2,5,main]:count is 0
Thread[Thread-0,5,main]:count is 0
Thread[Thread-6,5,main]:count is 4
```

我们发现没有完成我的需求，这是为什么呢，这是因为count并非是一个final的对象，其实每一个锁都是加在不同的对象上的。那我们应该怎么写呢？

```java
class Sender {
    private final Object lock = new Object();
    private Integer count = 0;

    public void printIncrease() {
        synchronized (lock) {
            count = count + 1;
        }
        System.out.println(Thread.currentThread().toString() + ":" + "count is " + count);
    }
}
```

我们在内部申明一个变量就可以达到这样的效果。但是结果是怎么样的呢？

```bash
Thread[pool-1-thread-2,5,main]:synchronization.Sender@65570b43count is 2
Thread[pool-1-thread-1,5,main]:synchronization.Sender@65570b43count is 2
Thread[pool-1-thread-3,5,main]:synchronization.Sender@65570b43count is 3
```

我们依然发现有并发的问题，很多同学会觉得是 [DCL双检锁](https://www.cnblogs.com/redcreen/archive/2011/03/29/1998802.html) 的问题，实际上并非是，这里的问题在于
System.out.println 的代码是

```java
public void println(String x) {
        synchronized (this) {
            print(x);
            newLine();
        }
    }
```
我们发现这里其实有自己的一个Lock，相当在过程中会出现线程之间修改了Counter的值，但是我们可以看出来最后值是对的，说明我们执行的次数是正确的，如果避免这个问题呢？

***方法一***

```java
public void printIncrease() {
        synchronized (lock) {
            count = count + 1;
            System.out.println(Thread.currentThread().toString() + ":" + this.toString() + "count is " + count);
        }
    }
```

***方法二***
```java
class Sender {
    private final Object lock = new Object();
    private volatile int count = 0;

    public void printIncrease() {
        int realCount;
        synchronized (lock) {
            count = count + 1;
            realCount = count;
        }

        System.out.println(Thread.currentThread().toString() + ":" + this.toString() + "count is " + realCount);
    }
}
```

### Synchronized 类型

我们再尝试将 此类型的 Class 锁住，效果也可以达成，因为Class其实也在我们内存中的一块区域。

```java
class Sender {
    private Integer count = 0;

    public void printIncrease() {
        synchronized (Sender.class) {
            count = count + 1;
        }
        System.out.println(Thread.currentThread().toString() + ":" + "count is " + count);
    }
}
```

```bash
Thread[Thread-47,5,main]:count is 48
Thread[Thread-48,5,main]:count is 49
Thread[Thread-49,5,main]:count is 50
```


### Synchronized 多实例
OK，这个时候如果申明多个实例呢？尝试一下。


```java
public class Synchronization {

    public static void main(String[] args) throws InterruptedException {
        Sender s = new Sender();
        Sender s2 = new Sender();
        LinkedList<SimpleCallback> runnables = new LinkedList<>();

        for (int i = 0; i < 50; i++) {
            if (i % 2 == 0) {
                runnables.add(i, new SimpleCallback(s));
            } else {
                runnables.add(i, new SimpleCallback(s2));
            }
        }

        ExecutorService executorService = Executors.newFixedThreadPool(50);
        executorService.invokeAll(runnables);
        executorService.awaitTermination(5, TimeUnit.SECONDS);
        executorService.shutdown();
    }
}


class Sender {
    private final Object lock = new Object();
    private volatile int count = 0;

    public void printIncrease() {
        int realCount;
        synchronized (lock) {
            count = count + 1;
            realCount = count;
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }

        System.out.println(Thread.currentThread().toString() + ":" + this.toString() + "count is " + realCount);
    }
}

class SimpleCallback implements Callable<Void> {

    private final Sender sender;

    SimpleCallback(Sender sender) {
        this.sender = sender;
    }

    @Override
    public Void call() throws Exception {
        sender.printIncrease();
        return null;
    }
}


```

```bash
Thread[pool-1-thread-2,5,main]:synchronization.Sender@af58fe3count is 1
Thread[pool-1-thread-1,5,main]:synchronization.Sender@17748af7count is 1
1s 之后
Thread[pool-1-thread-50,5,main]:synchronization.Sender@af58fe3count is 2
Thread[pool-1-thread-49,5,main]:synchronization.Sender@17748af7count is 2
```
我们可以发现其实锁的本质就在于增加某一个变量的内存空间，如果是不同的对象自然是不同的锁。大胆的猜测如果锁加在类上，因为是唯一的地址空间那因为是一个接着一个启动的。

```java
class Sender {
    private final Object lock = new Object();
    private volatile int count = 0;

    public void printIncrease() {
        int realCount;
        synchronized (Sender.class) {
            count = count + 1;
            realCount = count;
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }

        System.out.println(Thread.currentThread().toString() + ":" + this.toString() + "count is " + realCount);
    }
}
```
我们可以看见结果 
```bash
Thread[pool-1-thread-1,5,main]:synchronization.Sender@4878c7d7count is 1
1s 之后
Thread[pool-1-thread-50,5,main]:synchronization.Sender@7b267845count is 1
1s 之后
Thread[pool-1-thread-49,5,main]:synchronization.Sender@4878c7d7count is 2
1s 之后
```


### Synchronized 可重入性
这个特性主要是针对当前线程而言的，可重入即是自己可以再次获得自己的内部锁，在尝试获取对象锁时，如果当前线程已经拥有了此对象的锁，则把锁的计数器加一，在释放锁时则对应地减一，当锁计数器为0时表示锁完全被释放。

```java
class DeadLock {

    public synchronized void method1() {
    }

    public synchronized void method2() {
        this.method1();
    }

    public static void main(String[] args) {
        DeadLock deadLock = new DeadLock();
        deadLock.method2();
    }
}
```

### Synchronized 的非公平

```bash
TODO 如何复现
```


### 其他
synchronized最后一个特性（缺点）就是不可中断性，在所有等待的线程中，你们唯一能做的就是等，而实际情况可能是有些任务等了足够久了，我要取消此任务去干别的事情，此时synchronized是无法帮你实现的，它把所有实现机制都交给了JVM，提供了方便的同时也体现出了自己的局限性。 




