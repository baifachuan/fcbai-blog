---
title: 一种简单的死锁实现--Java Integer 缓存引起的死锁
tags: 编程基础
categories: 编程基础
abbrlink: 809cab72
date: 2020-10-09 17:38:10
---

### 死锁的背景知识

在写代码前先补充一下死锁的一些简单的知识：

```
死锁是指两个或两个以上的进程在执行过程中，因争夺资源而造成的一种互相等待的现象，若无外力作用，它们都将无法推进下去。
```


#### 产生死锁的原因主要是：

* （1） 因为系统资源不足。
* （2） 进程运行推进的顺序不合适。
* （3） 资源分配不当等。

如果系统资源充足，进程的资源请求都能够得到满足，死锁出现的可能性就很低，否则就会因争夺有限的资源而陷入死锁。

其次，进程运行推进顺序与速度不同，也可能产生死锁。



#### 产生死锁的四个必要条件：
* （1） 互斥条件：一个资源每次只能被一个进程使用。
* （2） 占有且等待：一个进程因请求资源而阻塞时，对已获得的资源保持不放。
* （3）不可强行占有:进程已获得的资源，在末使用完之前，不能强行剥夺。
* （4） 循环等待条件:若干进程之间形成一种头尾相接的循环等待资源关系。
这四个条件是死锁的必要条件，只要系统发生死锁，这些条件必然成立，而只要上述条件之
一不满足，就不会发生死锁。

#### 死锁的解除与预防：

理解了死锁的原因，尤其是产生死锁的四个必要条件，就可以最大可能地避免、预防和解除死锁。所以，在系统设计、进程调度等方面注意如何不让这四个必要条件成立，如何确
定资源的合理分配算法，避免进程永久占据系统资源。此外，也要防止进程在处于等待状态
的情况下占用资源。因此，对资源的分配要给予合理的规划




####  处理死锁的基本方法：

* 死锁预防：通过设置某些限制条件，去破坏死锁的四个条件中的一个或几个条件，来预防发生死锁。但由于所施加的限制条件往往太严格，因而导致系统资源利用率和系统吞吐量降低。

* 死锁避免：允许前三个必要条件，但通过明智的选择，确保永远不会到达死锁点，因此死锁避免比死锁预防允许更多的并发。

* 死锁检测：不须实现采取任何限制性措施，而是允许系统在运行过程发生死锁，但可通过系统设置的检测机构及时检测出死锁的发生，并精确地确定于死锁相关的进程和资源，然后采取适当的措施，从系统中将已发生的死锁清除掉。

* 死锁解除：与死锁检测相配套的一种措施。当检测到系统中已发生死锁，需将进程从死锁状态中解脱出来。常用方法：撤销或挂起一些进程，以便回收一些资源，再将这些资源分配给已处于阻塞状态的进程。死锁检测盒解除有可能使系统获得较好的资源利用率和吞吐量，但在实现上难度也最大。



#### 死锁预防：

破坏死锁的四个条件中的一个或几个。

* (1)互斥：它是设备的固有属性所决定的，不仅不能改变，还应该加以保证。

* (2)占有且等待：为预防占有且等待条件，可以要求进程一次性的请求所有需要的资源，并且阻塞这个进程直到所有请求都同时满足。这个方法比较低效。

* (3)不可抢占：预防这个条件的方法：

    * 如果占有某些资源的一个进程进行进一步资源请求时被拒绝，则该进程必须释放它最初占有的资源。

    * 如果一个进程请求当前被另一个进程占有的一个资源，则操作系统可以抢占另外一个进程，要求它释放资源。

* (4)循环等待：通过定义资源类型的线性顺序来预防。如果一个进程已经分配了R类资源，那么接下来请求的资源只能是那些排在R类型之后的资源类型。该方法比较低效。



####  死锁避免：

两种死锁避免算法：

* 进程启动拒绝：如果一个进程的请求会导致死锁，则不启动该进程。

* 资源分配拒绝：如果一个进程增加的资源请求会导致死锁，则不允许此分配(银行家算法)。



### Java中因为Intger缓存引起的死锁

很多时候在初入职场，或者面试公司的时候，有的企业会做一些基础的面试题，特别以校招居多，例如让学生实现一个死锁。

以下代码为一种简单的死锁实现：

```JAVA
import java.util.concurrent.Callable;
import java.util.concurrent.ExecutionException;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.Future;

public class DeadLockExample {

    public static void main(String[] args) {
        for (int i = 0; i < 10; i++) {
            new Thread(new IntegerLock("bob", 1, 2)).start();
            new Thread(new IntegerLock("frank", 2, 1)).start();
        }
    }

    private static class IntegerLock implements Runnable {
        private String name;
        private int i, j;
        public IntegerLock(String name, int i, int j) {
            this.name = name;
            this.i = i;
            this.j = j;
        }

        @Override
        public void run() {
            synchronized (Integer.valueOf(i)) {
                synchronized (Integer.valueOf(j)) {
                    System.out.println(name + ":running");
                }
            }
        }
    }
}
```

可以看到这个例子里面的实现有一个循环10次，因为本例子基于的是概率，10次只是增大这个概率的发生。

具体的原理是在于`Integer.valueOf`这个函数，可以看到源码部分：

```java
public static Integer valueOf(int i) {
        if (i >= IntegerCache.low && i <= IntegerCache.high)
            return IntegerCache.cache[i + (-IntegerCache.low)];
        return new Integer(i);
}
```

如果int i的值在[-128, 127]之间，那么直接返回IntegerCache对象中cache的值，也就是说定义如下变量：

```java
Integer i = Integer.valueOf(1);
Integer j = Integer.valueOf(1);
```

实际上`i`和`j`是同一个对象，因此当我们在获取i的监视权后再去申请j，则会造成死锁，和java中的字符串池有一定的类似。

死锁的实现也是思路多多，无论何种实现，理解其原理才是主要的。
