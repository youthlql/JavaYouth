---
title: 'Java并发体系-第四阶段-AQS源码解读-[1]'
tags:
  - Java并发
  - AQS源码
categories:
  - Java并发
  - 原理
keywords: Java并发，AQS源码
description: '万字系列长文讲解-Java并发体系-第四阶段-AQS源码解读-[1]。'
cover: 'https://npm.elemecdn.com/lql_static@latest/logo/Java_concurrency.png'
abbrlink: 92c4503d
date: 2020-10-26 17:59:42
---



# 可重入锁



```java

/**
 * @Author: youthlql-吕
 * @Date: 2020/10/22 21:12
 * <p>
 * 可重入锁:
 * 1、可重复可递归调用的锁，在外层使用锁之后，在内层仍然可以使用，并且不发生死锁，这样的锁就叫做可重入锁。
 * 2、是指在同一个线程在外层方法获取锁的时候，再进入该线程的内层方法会自动获取锁(前提，锁对象得是同一个
 * 对象)，不会因为之前已经获取过还没释放而阻塞
 */
public class ReEnterLockDemo {

    static Object objectLockA = new Object();

    public static void m1(){
        new Thread(() -> {
            synchronized (objectLockA){
                System.out.println(Thread.currentThread().getName()+"\t"+"------外层调用");
                synchronized (objectLockA){
                    System.out.println(Thread.currentThread().getName()+"\t"+"------中层调用");
                    synchronized (objectLockA)
                    {
                        System.out.println(Thread.currentThread().getName()+"\t"+"------内层调用");
                    }
                }
            }
        },"t1").start();

    }

    public static void main(String[] args) {
        m1();
    }
}
```



```java
public class ReEnterLockDemo {

    public synchronized void m1(){
        System.out.println("=====外层");
        m2();
    }

    public synchronized void m2() {
        System.out.println("=====中层");
        m3();
    }

    public synchronized void m3(){
        System.out.println("=====内层");
    }


    public static void main(String[] args) {
        new ReEnterLockDemo().m1();
    }
}
 
```





# LockSupport

## 是什么？

> 官方说明：https://www.apiref.com/java11-zh/java.base/java/util/concurrent/locks/LockSupport.html



LockSupport中的park()和unpark()的作用分别是阻塞线程和解除阻塞线程，相当于线程等待和唤醒机制的加强版。



## 3种让线程等待和唤醒的方法

- 方式1:  使用Object中的wait()方法让线程等待， 使用Object中的notify()方法唤醒线程
- 方式2:  使用JUC包中Condition的await()方法让线程等待，使用signal()方法唤醒线程 
- 方式3:  LockSupport类可以阻塞当前线程以及唤醒指定被阻塞的线程



## Object类提供的等待唤醒机制的缺点

### 正常情况下

```java
public class LockSupportDemo1 {
    static Object objectLock = new Object();
    public static void main(String[] args) {
        new Thread(() -> {
            synchronized (objectLock){
                System.out.println(Thread.currentThread().getName()+"\t"+"------come in");
                try {
                    objectLock.wait();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println(Thread.currentThread().getName()+"\t"+"------被唤醒");
            }
        },"A").start();

        new Thread(() -> {
            synchronized (objectLock)
            {
                objectLock.notify();
                System.out.println(Thread.currentThread().getName()+"\t"+"------通知");
            }
        },"B").start();
    }

}
```

结果：

```
A	------come in
B	------通知
A	------被唤醒

Process finished with exit code 0
```



### 异常情况1

去掉同步代码块

```java
public class LockSupportDemo1 {
    static Object objectLock = new Object();
    public static void main(String[] args) {
        new Thread(() -> {
//            synchronized (objectLock){
                System.out.println(Thread.currentThread().getName()+"\t"+"------come in");
                try {
                    objectLock.wait();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println(Thread.currentThread().getName()+"\t"+"------被唤醒");
//            }
        },"A").start();

        new Thread(() -> {
//            synchronized (objectLock){
                objectLock.notify();
                System.out.println(Thread.currentThread().getName()+"\t"+"------通知");
//            }
        },"B").start();
    }
}    
```

结果：

```
A	------come in
Exception in thread "A" Exception in thread "B" java.lang.IllegalMonitorStateException
	at java.lang.Object.wait(Native Method)
	at java.lang.Object.wait(Object.java:502)
	at com.youth.guiguthirdquarter.AQS.LockSupportDemo1.lambda$main$0(LockSupportDemo1.java:16)
	at java.lang.Thread.run(Thread.java:748)
java.lang.IllegalMonitorStateException
	at java.lang.Object.notify(Native Method)
	at com.youth.guiguthirdquarter.AQS.LockSupportDemo1.lambda$main$1(LockSupportDemo1.java:26)
	at java.lang.Thread.run(Thread.java:748)

Process finished with exit code 0
```

报错了。



### 异常情况2

先唤醒，再等待。

```java
public class LockSupportDemo1 {
    static Object objectLock = new Object();
    public static void main(String[] args) {
        new Thread(() -> {
            try {
                TimeUnit.SECONDS.sleep(2);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            synchronized (objectLock){
                System.out.println(Thread.currentThread().getName()+"\t"+"------come in");
                try {
                    objectLock.wait();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println(Thread.currentThread().getName()+"\t"+"------被唤醒");
            }
        },"A").start();

        new Thread(() -> {
            synchronized (objectLock)
            {
                objectLock.notify();
                System.out.println(Thread.currentThread().getName()+"\t"+"------通知");
            }
        },"B").start();
    }
}    
```

结果：

```java
B	------通知
A	------come in

Process finished with exit code -1
```

死循环，A无法被唤醒了。



这两点我们之前也说过，Object类提供的wait和notify

1、只能在synchronized同步代码块里使用

2、只能先等待（wait），再唤醒（notify）。顺序一旦错了，那个等待线程就无法被唤醒了。





## Condion类提供的等待唤醒机制的缺点

缺点和Object类里的wait，notify一样。

1、只能在lock同步代码块里使用，不然就报错

2、只能先等待（await），再唤醒（signal）。顺序一旦错了，那个等待线程就无法被唤醒了。

但相对于wait，notify改进的一点是，可以绑定lock进行定向唤醒。



## LockSupport的优点

有的时候我不需要进入同步代码块，我只是需要让线程阻塞，这个时候LockSupport就发挥作用了。并且还解决了之前的第二个问题，也就是等待必须在唤醒的前面。

```java
static void park()  //除非许可证可用，否则禁用当前线程以进行线程调度。 
static void	unpark(Thread thread)	//如果给定线程尚不可用，则为其提供许可。
```

- LockSupport是用来创建锁和其他同步类的基本线程阻塞原语。

- LockSupport类使用了一种名为Permit(许可）的概念来做到阻塞和唤醒线程的功能，每个线程都有一个许可(permit),
  permit只有两个值1和零，默认是零。可以把许可看成是一种(0,1)信号量(Semaphore），但与Semaphore不同的是，许可的累加上限是1。

```java
public static void park() {
        UNSAFE.park(false, 0L);
    }
```

LockSupport底层还是UNSAFE（前面讲过）。

- permit默认是0，所以一开始调用park()方法，当前线程就会阻塞，直到别的线程将当前线程的permit设置为1时,park方法会被唤醒，然后会将permit再次设置为0并返回。

- 调用unpark(thread)方法后，就会将thread线程的许可permit设置成1(注意多次调用unpark方法，不会累加，permit值还是1)会自动唤醒thread线程，即之前阻塞中的LockSupport.park()方法会立即返回。

- LockSupport和每个使用它的线程都有一个许可(permit)关联。permit相当于1，0的开关，默认是0，
  调用一次unpark就将0变成1，
  调用一次park会消费permit，也就是将1变成o，同时park立即返回。
  如再次调用park会变成阻塞(因为permit为零了会阻塞在这里，一直到permit变为1)，这时调用unpark会把permit置为1。
  每个线程都有一个相关的permit, permit最多只有一个，重复调用unpark也不会积累凭证。

- 形象的理解
  线程阻塞需要消耗凭证(permit)，这个凭证最多只有1个。
  当调用park方法时
      如果有凭证，则会直接消耗掉这个凭证然后正常退出;
      如果无凭证，就必须阻塞等待凭证可用;
  而unpark则相反，它会增加一个凭证，但凭证最多只能有1个，累加无效。

我们用LockSupport来测试下之前的异常场景

### 异常情况1

无同步代码块

```java
public class LockSupportDemo3 {
    public static void main(String[] args) {
        /**
         LockSupport：俗称 锁中断
         LockSupport它的解决的痛点
         1。LockSupport不用持有锁块，不用加锁，程序性能好，
         2。不需要等待和唤醒的先后顺序，不容易导致卡死
         */
        Thread t1 = new Thread(() -> {

            System.out.println(Thread.currentThread().getName() + "\t ----begin-时间：" + System.currentTimeMillis());
            LockSupport.park();//阻塞当前线程
            System.out.println(Thread.currentThread().getName() + "\t ----被唤醒-时间：" + System.currentTimeMillis());
        }, "t1");
        t1.start();
        LockSupport.unpark(t1);
        System.out.println(Thread.currentThread().getName() + "\t 通知t1...");


    }
}
```

结果：

```java
t1	 ----begin-时间：1603376148147
t1	 ----被唤醒-时间：1603376148147
main	 通知t1...

Process finished with exit code 0
```

没有问题



### 异常情况2

先唤醒，再阻塞（等待）。

```java
public static void main(String[] args) {
        Thread t1 = new Thread(() -> {
            try {
                TimeUnit.SECONDS.sleep(2);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println(Thread.currentThread().getName() + "\t ----begin-时间：" + System.currentTimeMillis());
            LockSupport.park();//阻塞当前线程
            System.out.println(Thread.currentThread().getName() + "\t ----被唤醒-时间：" + System.currentTimeMillis());
        }, "t1");
        t1.start();
        LockSupport.unpark(t1);
        System.out.println(Thread.currentThread().getName() + "\t 通知t1...");


    }
```

结果：

```
main	 通知t1...
t1	 ----begin-时间：1603376257183
t1	 ----被唤醒-时间：1603376257183

Process finished with exit code 0
```

可以看到，如果你先唤醒了。那么后面的`LockSupport.park();`就相当于瞬间被唤醒了，不会和之前一样程序卡死。为什么呢？结合之前分析的流程

1、先执行unpark，将许可证由0变为1

2、然后park来了发现许可证此时为0（也就是有许可证），那么他就不会阻塞，马上就往后执行。同时消耗许可证（也就是将1又变为0）。





# AQS



## AQS是什么？

**字面意思：**抽象的队列同步器

**技术翻译：**是用来构建锁或者其它同步器组件的重量级基础框架及整个JUC体系的基石， 通过内置的FIFO队列来完成资源获取线程的排队工作，并通过一个int类变量`state`表示持有锁的状态。

<img src="https://npm.elemecdn.com/youthlql@1.0.8/Java_concurrency/Source_code/Fourth_stage/0001.png">



<img src="https://npm.elemecdn.com/youthlql@1.0.8/Java_concurrency/Source_code/Fourth_stage/0011.png">

AbstractOwnableSynchronizer
AbstractQueuedLongSynchronizer
AbstractQueuedSynchronizer    

上面几个都是AQS，但是通常地: AbstractQueuedSynchronizer简称为AQS。



AQS是一个抽象的父类，可以将其理解为一个框架。基于AQS这个框架，我们可以实现多种同步器，比如下方图中的几个Java内置的同步器。同时我们也可以基于AQS框架实现我们自己的同步器以满足不同的业务场景需求。

<img src="https://npm.elemecdn.com/youthlql@1.0.8/Java_concurrency/Source_code/Fourth_stage/0002.png">



## AQS能干嘛？

加锁会导致阻塞：有阻塞就需要排队，实现排队必然需要有某种形式的队列来进行管理

<img src="https://npm.elemecdn.com/youthlql@1.0.8/Java_concurrency/Source_code/Fourth_stage/0003.png">

1、抢到资源的线程直接使用办理业务，抢占不到资源的线程的必然涉及一种**排队等候机制**，抢占资源失败的线程继续去等待(类似办理窗口都满了，暂时没有受理窗口的顾客只能去候客区排队等候)，仍然保留获取锁的可能且获取锁流程仍在继续(候客区的顾客也在等着叫号，轮到了再去受理窗口办理业务）。

2、既然说到了**排队等候机制**，那么就一定 会有某种队列形成，这样的队列是什么数据结构呢?

3、如果共享资源被占用，就需要一定的阻塞等待唤醒机制来保证锁分配。这个机制主要用的是CLH队列的变体实现的，将暂时获取不到锁的线程加入到队列中，这个队列就是AQS的抽象表现。它将请求共享资源的线程封装成队列的结点(Node) ，通过CAS、自旋以及LockSuport.park()的方式，维护state变量的状态，使并发达到同步的效果。



# AQS独占模式（以ReentrantLock 源码为例）

## AQS结构

```Java
// 头结点，你直接把它当做当前持有锁的线程 可能是最好理解的。实际上可能略有出入，往下看分析即可
private transient volatile Node head;

// 阻塞的尾节点，每个新的节点进来，都插入到最后，也就形成了一个链表
private transient volatile Node tail;

// 这个是最重要的，代表当前锁的状态，0代表没有被占用，大于 0 代表有线程持有当前锁
// 这个值可以大于 1，是因为锁可以重入，每次重入都加上 1
private volatile int state;

// 代表当前持有独占锁的线程，举个最重要的使用例子，因为锁可以重入
// reentrantLock.lock()可以嵌套调用多次，所以每次用这个来判断当前线程是否已经拥有了锁
// if (currentThread == getExclusiveOwnerThread()) {state++}
private transient Thread exclusiveOwnerThread; //继承自AbstractOwnableSynchronizer
```



## Node类结构

```java
static final class Node {
    // 标识节点当前在共享模式下
    static final Node SHARED = new Node();
    // 标识节点当前在独占模式下
    static final Node EXCLUSIVE = null;

    // ======== 下面的几个int常量是给waitStatus用的 ===========
    /** waitStatus value to indicate thread has cancelled */
    // 代码此线程取消了争抢这个锁
    static final int CANCELLED =  1;
    /** waitStatus value to indicate successor's thread needs unparking */
    // 官方的描述是，其表示当前node的后继节点对应的线程需要被唤醒
    static final int SIGNAL    = -1;
    /** waitStatus value to indicate thread is waiting on condition */
    // 等待condition唤醒
    static final int CONDITION = -2;
    /**
     * waitStatus value to indicate the next acquireShared should
     * unconditionally propagate
     */
    // 共享模式同步状态获取讲会无条件的传播下去（共享模式下，该字段才会使用）
    static final int PROPAGATE = -3;
    // ===============-2和-3用的不多，暂时不分析======================================


    // 取值为上面的1、-1、-2、-3，或者0(以后会讲到，waitStatus初始值为0)
    // 这么理解，暂时只需要知道如果这个值 大于0 代表此线程取消了等待，
    //    ps: 半天抢不到锁，不抢了，ReentrantLock是可以指定timeouot的
    volatile int waitStatus;
    // 前驱节点的引用
    volatile Node prev;
    // 后继节点的引用
    volatile Node next;
    // 这个就是线程本尊
    volatile Thread thread;

}
```

Node 的数据结构其实也挺简单的，就是 thread + waitStatus + pre + next 四个属性而已，大家先要有这个概念在心里。



## AQS队列基本结构

<img src="https://npm.elemecdn.com/youthlql@1.0.8/Java_concurrency/Source_code/Fourth_stage/0004.png">

注意排队队列，不包括head（也就是后文要说的哨兵节点）。



## 开始

```java
package com.youth.guiguthirdquarter.AQS;

import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.ReentrantLock;

/**
 * @Author: youthlql-吕
 * @Date: 2020/10/25 21:59
 * <p>
 * 功能描述:
 */
public class AQSDemo {
    public static void main(String[] args) {

        ReentrantLock lock = new ReentrantLock();

        //带入一个银行办理业务的案例来模拟我们的AQS如何进行线程的管理和通知唤醒机制

        //3个线程模拟3个来银行网点，受理窗口办理业务的顾客

        //A顾客就是第一个顾客，此时受理窗口没有任何人，A可以直接去办理
        new Thread(() -> {
            lock.lock();
            try{
                System.out.println("-----A thread come in");

                try { TimeUnit.MINUTES.sleep(20); }catch (Exception e) {e.printStackTrace();}
            }finally {
                lock.unlock();
            }
        },"A").start();

        //第二个顾客，第二个线程---》由于受理业务的窗口只有一个(只能一个线程持有锁)，此时B只能等待，
        //进入候客区
        new Thread(() -> {
            lock.lock();
            try{
                System.out.println("-----B thread come in");
            }finally {
                lock.unlock();
            }
        },"B").start();

        //第三个顾客，第三个线程---》由于受理业务的窗口只有一个(只能一个线程持有锁)，此时C只能等待，
        //进入候客区
        new Thread(() -> {
            lock.lock();
            try{
                System.out.println("-----C thread come in");
            }finally {
                lock.unlock();
            }
        },"C").start();
    }
}
```

以这样的一个实际例子说明。

<img src="https://npm.elemecdn.com/youthlql@1.0.8/Java_concurrency/Source_code/Fourth_stage/0005.png">



## 非公平锁lock()加锁

### lock()

```java

    static final class NonfairSync extends Sync {
        private static final long serialVersionUID = 7316153563782823691L;

        final void lock() {
            /*
            1、非公平锁不公平的第一个原因就出现在这里。刚准备加锁的线程，这里会用CAS抢一下锁（也就是通过
            看state的状态）。如果抢成功了就调用setExclusiveOwnerThread，设置当前持有独占锁的线程为本
            线程。
            */
            if (compareAndSetState(0, 1))
                setExclusiveOwnerThread(Thread.currentThread());
            else
                //如果抢锁失败就走入这个流程，抢锁失败说明当前锁已经被占用了
                acquire(1);
        }

        protected final boolean tryAcquire(int acquires) {
            return nonfairTryAcquire(acquires);
        }
    }
	//相当于只要调用了这个方法，说明线程独占锁成功
	protected final void setExclusiveOwnerThread(Thread thread) {
        exclusiveOwnerThread = thread;
    }
```

> A线程刚进来的时候，AQS的head和tail节点都还没有被初始化，则会被默认初始化为null。并且state默认初始化为0。

1、A线程进去窗口办理业务，此时state == 0，那么CAS就直接成功了，并且把sate改为1。然后调用下`setExclusiveOwnerThread`，就直接结束了。【加锁成功，直接返回】

**B线程**

1、接着B线程去窗口办理业务，因为之前A线程把state变为了1，那么B线程在进行第一个if-CAS判断就会失败。所以就走到了else分支，调用`acquire(1)`方法。



**C线程**

因为A线程占用着锁，C线程执行逻辑和B一样。（后续假设C进行加锁时间在B后面一点）

### acquire()和tryAcquire()

```java

	/*
	1、acquire()方法来自父类AQS，我们看到，这个方法，如果tryAcquire(arg) 返回true, 也就结束了。
    否则，acquireQueued方法会将线程压到队列中。
	*/
    public final void acquire(int arg) { // 此时 arg == 1
        
        /*
        1、首先调用tryAcquire(1)一下，名字上就知道，这个只是试一试。因为有可能直接就成功了呢，也就不需要
        进队列排队了。
        2、有可能成功的情况就是，在走到这一步的时候，前面占锁的线程刚好释放锁
        */
        if (!tryAcquire(arg) &&
            // tryAcquire(arg)没有成功，这个时候需要把当前线程挂起，放到阻塞队列中。
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg)) {
              selfInterrupt();
        }
    }


	/*
	1、上面的tryAcquire里会直接调用ReentrantLock类的nonfairTryAcquire方法，
	2、尝试直接获取锁，返回值是boolean，代表是否获取到锁
    3、有两种情况会返回true：
    	1.没有线程在等待锁
    	2.重入锁，线程本来就持有锁，也就可以理所当然可以直接获取
	*/
	final boolean nonfairTryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
        	/*
        	1、state == 0 此时此刻没有线程持有锁
        	2、前面也说了有可能成功的情况就是，在走到这一步的时候，前面占锁的线程刚好释放锁
        	*/
            if (c == 0) {
                //那就用CAS尝试一下，成功了就获取到锁了。
                if (compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
        	// 会进入这个else if分支，说明是重入锁了，需要操作：state=state+1
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0) // overflow
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }

```

**B线程**

1、B线程最终走进了`nonfairTryAcquire()`方法，但是因为A还在占锁（占着处理窗口state），所以此时state为1，B线程走到else if分支进行判断。

2、B线程发现已经占有锁的线程不是自己，说明不是重入锁，也不会进入else if分支。最终返回fasle，回到`tryAcquire`，准备挂起线程。



**C线程**

因为A线程占用着锁，C线程执行逻辑和B一样





### addWaiter()

```java



    
	/*
	1、假设tryAcquire(arg) 返回false，那么代码将执行：acquireQueued(addWaiter(Node.EXCLUSIVE),
	arg)，这个方法，首先需要执行：addWaiter(Node.EXCLUSIVE)
	2、此方法的作用是把线程包装成node，同时进入到队列中。参数mode此时是Node.EXCLUSIVE，代表独占模式
	3、以下几行代码想把当前node加到链表的最后面去，也就是进到队列的最后
	*/
    private Node addWaiter(Node mode) {
        Node node = new Node(Thread.currentThread(), mode);
        //得到尾节点（head和tail在没有初始化前都是null，没有初始化的时候也说明队列为空）
        Node pred = tail;

        //队列不为空时（即之前已经初始化过了），会进入下面这个分支，此时只需要将新的node加入队尾
        if (pred != null) { 
            // 将当前的队尾节点，设置为自己的前驱 
            node.prev = pred; 
            // 用CAS把自己设置为队尾, 如果成功后，tail == node 了，这个节点成为排队队列新的尾巴
            if (compareAndSetTail(pred, node)) { 
                /*
                1、进到这里说明设置成功，当前node==tail, 将自己与之前的队尾相连，上面已经有
                node.prev = pred，加上下面这句，也就实现了和之前的尾节点双向连接了
                */
                pred.next = node;
                // 线程入队了，可以返回了
                return node;
            }
        }
        
        /*
        1、仔细看看上面的代码，有两种情况会走到这里
           1、pred==null(说明队列是空的) 
           2、CAS设置队尾失败(有线程在竞争入队)
        */
        enq(node);
        return node;
    }

```

> 之前说了A线程刚进来的时候，AQS的head和tail节点都还没有被初始化，则会被默认初始化为null

**B线程**

1、B线程进入`addWaiter()`，发现pred == null，直接进入`enq()`



**C线程**

1、【前面说了C在B后面】，C线程进来后和B不一样，因为B在后面已经设置了tail指针。那么C线程在判断的时候pred 就不是null，就直接进入了if分支

2、C在if逻辑里准备入队，进行相应设置后，变成下面这样。

<img src="https://npm.elemecdn.com/youthlql@1.0.8/Java_concurrency/Source_code/Fourth_stage/0006.png">



### enq()

```java
 
	/*
	1、采用空的for循环，以自旋的方式入队，到这个方法只有两种可能：队列为空，或者有线程竞争入队【上面说过】
    2、自旋在这边的语义是：CAS设置tail过程中，竞争一次竞争不到，我就多次竞争，总会排到的
	*/
    private Node enq(final Node node) {
        for (;;) {
            Node t = tail;
            
           	/*
           	1、进入这个分支，说明是队列为空的这种情况，那么就准备初始化一个空的节点（new Node()）
           	作为排队队列的head。
           	*/
            if (t == null) { // Must initialize
                /*
                1、初始化head节点，前面说过 head 和 tail 初始化的时候都是 null 的。
                2、还是一步CAS，因为可能是很多线程同时进来呢   
                */
                if (compareAndSetHead(new Node()))                    
               /*
                1、注意这里传的参数是new Node()，说明是一个空的节点（并不是我们B线程封装的节点，
                这个空节点只作为占位符，称作傀儡节点或者哨兵节点）。这个时候head节点的waitStatus==0,
                看new Node()构造方法就知道了。注意：new Node()虽然是空节点，但他不是null
                2、这个时候有了head，但是tail还是null，设置一下，把tail指向head，放心，马上就有
                线程要来了，到时候tail就要被抢了
                3、注意：这里只是设置了tail=head，这里可没return哦。所以，设置完了以后，继续for
                循环，下次就到下面的else分支了
                */
                    tail = head;
            } else {             
                /*
                1、下面几行，和上一个方法 addWaiter 是一样的，只是这个套在无限循环里，就是将当前
                线程排到队尾，有线程竞争的话排不上重复排，直到排上了再return 
                【这里看不懂的话就看下面的例子】               
                */
                
                node.prev = t;
                if (compareAndSetTail(t, node)) {
                    t.next = node;
                    return t;
                }
            }
        }
    }

```

**B线程**

**第一轮循环**

1、B线程进入enq()。首先发现t == tail 依然为null，那么就直接进入if分支。

2、进入if分支后，调用`compareAndSetHead(new Node())`准备初始化head节点。注意这里传的参数是`new Node()`，说明是一个空的节点（并不是我们B线程封装的节点，这个空节点只作为占位符，**称作傀儡节点或者哨兵节点**），然后将head赋值给tail。

>  补充：双向链表中，第一个节点为虚节点(也叫哨兵节点)，其实并不存储任何信息，只是占位。 真正的第一个有数据的节点，是从第二个节点开始的。

此时队列变成了下面的样子：

<img src="https://npm.elemecdn.com/youthlql@1.0.8/Java_concurrency/Source_code/Fourth_stage/0007.png">

3、然后if结束之后，继续空的for循环，B线程开始了第二轮循环。



**第二轮循环**

1、第二次循环再过来的时候，t == tail，但此时tail不再为null，所以进入else分支。

2、`node.prev = t`，进入if之后，让B节点的prev指针指向t，然后`compareAndSetTail(t, node)`设置尾节点

<img src="https://npm.elemecdn.com/youthlql@1.0.8/Java_concurrency/Source_code/Fourth_stage/0008.png">

3、CAS设置尾节点成功之后，执行if里的逻辑

<img src="https://npm.elemecdn.com/youthlql@1.0.8/Java_concurrency/Source_code/Fourth_stage/0009.png">





### acquireQueued()

```java

	/*
	1、现在，又回到这段代码了
     if (!tryAcquire(arg) 
            && acquireQueued(addWaiter(Node.EXCLUSIVE), arg)) 
         selfInterrupt();
    2、acquireQueued这个方法，参数node，经过addWaiter(Node.EXCLUSIVE)，此时已经进入排队队列队尾
    3、注意一下：如果acquireQueued(addWaiter(Node.EXCLUSIVE), arg))返回true的话，意味着上面这段
    代码将进入selfInterrupt()
    4、这个方法非常重要，真正的线程挂起，然后被唤醒后去获取锁，都在这个方法里了
	*/
    final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();             
                /*
                1、p == head 说明当前节点虽然进到了排队队列，但是是队列的第一个，因为它的前驱是head
                （或者说是哨兵节点，因为head指向了哨兵节点）
                2、注意，队列不包含head节点，head一般指的是占有锁的线程，head后面的才称为排队队列
                3、所以当前节点可以去试抢一下锁
                4、这里我们说一下，为什么可以去试试：它是排队队列队头，所以作为队头，可以去试一试能不能
                拿到锁，因为可能之前的线程已经释放锁了。如果尝试成功，那它就不需要被挂起，直接拿锁，
                效率会高
                5、tryAcquire已经分析过了, 忘记了请往前看一下，就是简单用CAS试操作一下state
                */
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC，这个后面释放锁的时候会讲
                    failed = false;
                    return interrupted;
                }
                
                /*
                1、到这里，说明上面的if分支没有成功。
                  1、要么当前node本来就不是队头，
                  2、要么就是tryAcquire(arg)没有抢赢别人，继续往下看
                */
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            //tryAcquire()方法抛异常时，failed为true，会取消当前节点的排队。
            if (failed)
                cancelAcquire(node);//取消排队
        }
    }
```

**B线程**

1、进入`acquireQueued()`后，发现也是一个空循环。首先通过`node.predecessor()`得到B节点的前一个节点P，也就是哨兵节点。

2、p == head为true。然后if里再次执行`tryAcquire(arg)`拿一次锁【流程前面已经分析过了，不重复了】。因为A线程任然持有锁，所以最终结果B节点`tryAcquire`失败。准备挂起线程



### shouldParkAfterFailedAcquire()

```java

	/*
	1、会到这里就是没有抢到锁呗，这个方法说的是："当前线程没有抢到锁，是否需要挂起当前线程？"
    第一个参数是前驱节点，第二个参数才是代表当前线程的节点
	*/
    private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        int ws = pred.waitStatus;
        // 前驱节点的 waitStatus == -1 ，说明前驱节点状态正常，当前线程需要挂起，直接可以返回true
        if (ws == Node.SIGNAL)
            return true;

        /*
        1、前驱节点 waitStatus大于0 ，之前说过，大于0说明前驱节点取消了排队。
        2、这里需要知道这点：进入阻塞队列排队的线程会被挂起，而唤醒的操作是由前驱节点完成的。所以
        下面这块代码说的是将当前节点的prev指向waitStatus<=0的节点，简单说，就是为了找个好爹，因为你还
        得依赖它来唤醒呢，如果前驱节点取消了排队，找前驱节点的前驱节点做爹，往前遍历总能找到一个好爹的。
        */
        if (ws > 0) {
            /*
             * Predecessor was cancelled. Skip over predecessors and
             * indicate retry.
             */
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
            pred.next = node;
        } else {
            /*
             * waitStatus must be 0 or PROPAGATE.  Indicate that we
             * need a signal, but don't park yet.  Caller will need to
             * retry to make sure it cannot acquire before parking.
             */
            /*
            1、如果进入到这个分支意味着什么，前驱节点的waitStatus不等于-1和1，那也就是只可能是0，-2，-3
            在我们前面的源码中，都没有看到有设置waitStatus的，所以每个新的node入队时，waitStatu都是0
            2、正常情况下，前驱节点是之前的 tail，那么它的 waitStatus 应该是 0，用CAS将前驱节点
            的waitStatus设置为Node.SIGNAL(也就是-1)，表示我后面有节点需要被唤醒。
            3、这里可以简单说下 waitStatus 中 SIGNAL(-1) 状态的意思，Doug Lea 注释的是：代表后继
            节点需要被唤醒。也就是说这个 waitStatus 其实代表的不是自己的状态，而是后继节点的状态，
            我们知道，每个node 在入队的时候，都会把前驱节点的状态改为 SIGNAL，然后阻塞，等待被前驱唤醒。
            这里涉及的是两个问 题：有线程取消了排队、唤醒操作。其实本质是一样的，读者也可以顺着 
            “waitStatus代表后继节点的状态”这种思路去看一遍源码。
            */
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
        // 这个方法返回 false，那么会再走一次 for 循序，然后再次进来此方法，此时会从第一个分支返回 true
        return false;
    }


    
	/*
	1、private static boolean shouldParkAfterFailedAcquire(Node pred, Node node)
    这个方法结束根据返回值我们简单分析下：
       1、如果返回true, 说明前驱节点的waitStatus==-1，是正常情况，那么当前线程需要被挂起，等待以后被唤醒
       我们也说过，以后是被前驱节点唤醒，就等着前驱节点拿到锁，然后释放锁的时候叫你好了
       2、如果返回false, 说明当前不需要被挂起，为什么呢？往后看
       
    需要跳回到前面这个方法
    if (shouldParkAfterFailedAcquire(p, node) &&
                   parkAndCheckInterrupt())
                   interrupted = true;
	*/


```

**B线程**

**第一次循环**

1、B线程的前驱节点是哨兵节点（ws == 0），  所以最终走了else分支，执行了  `compareAndSetWaitStatus(pred, ws, Node.SIGNAL)`方法。将哨兵节点的`compareAndSetWaitStatus`值变为了-1

2、返回false，返回到`acquireQueued()`进行第二次循环【不再赘述】。



**第二次循环**

1、此时B线程的前驱节点--哨兵节点的ws == -1。那么此方法返回true，准备执行parkAndCheckInterrupt



### parkAndCheckInterrupt()

```java

	/*
	1、如果shouldParkAfterFailedAcquire(p, node)返回true，那么需要执行parkAndCheckInterrupt():
    这个方法很简单，因为前面返回true，所以需要挂起线程，这个方法就是负责挂起线程的，
    2、这里用了LockSupport.park(this)来挂起线程，然后就停在这里了，等待被唤醒=======
	*/
    private final boolean parkAndCheckInterrupt() {
        LockSupport.park(this);
        return Thread.interrupted();
    }
	/*
	1、接下来说说如果shouldParkAfterFailedAcquire(p, node)返回false的情况
    2、仔细看shouldParkAfterFailedAcquire(p, node)，我们可以发现，其实第一次进来的时候，一般都不会
    返回true的，原因很简单，前驱节点的waitStatus=-1是依赖于后继节点设置的。也就是说，我都还没给前驱
    设置-1呢，怎么可能是true呢，但是要看到，这个方法是套在循环里的，所以第二次进来的时候状态就是-1了。

    3、解释下为什么shouldParkAfterFailedAcquire(p, node)返回false的时候不直接挂起线程：
    主要是为了应对在经过这个方法后，node已经是head的直接后继节点了。
    4、假设返回fasle的时候，node已经是head的直接后继节点了，但是你直接挂起了线程，就要走别人唤醒你的那
    几步代码。那这里完全可以重新走一遍for循环，直接尝试下获取锁，可能会更快。注意是可能，不代表一定，因为
    你也无法确定unparkSuccessor释放锁，通知后继节点这个方法执行的快慢。但是你多尝试一次获取锁，总归是快的。
    	for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return;
            }
	*/

}
```

到这一步，B线程才算真正的入队坐稳了。B线程在这里阻塞，或者说挂起。



## 非公平锁lock()解锁

然后，就是还需要介绍下唤醒的动作了。我们知道，正常情况下，如果线程没获取到锁，线程会被 `LockSupport.park(this);` 挂起停止，等待被唤醒。



### release()和tryRelease()

```java
// 唤醒的代码还是比较简单的，你如果上面加锁的都看懂了，下面都不需要看就知道怎么回事了
public void unlock() {
    sync.release(1);
}

//AQS类的方法
public final boolean release(int arg) {
    if (tryRelease(arg)) {
        Node h = head;
        //h是哨兵节点
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}

// 回到ReentrantLock看tryRelease方法
protected final boolean tryRelease(int releases) {
    int c = getState() - releases;
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
    // 是否完全释放锁
    boolean free = false;
    // 其实就是重入的问题，如果c==0，也就是说没有嵌套锁了，可以释放了，否则还不能释放掉
    if (c == 0) {
        free = true;
        setExclusiveOwnerThread(null);
    }
    setState(c);
    return free;
}

```





### unparkSuccessor()

```java
// 唤醒后继节点，从上面调用处知道，参数node是head头结点（或者说是哨兵节点，因为本身head就指向了哨兵节点）
private void unparkSuccessor(Node node) {

    int ws = node.waitStatus;
    // 如果head节点当前waitStatus<0, 将其修改为0
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0);
    /*
    1、下面的代码就是唤醒后继节点，但是有可能后继节点取消了等待（waitStatus==1）从队尾往前找，
    找到waitStatus<=0的所有节点中排在最前面的
    */
    Node s = node.next;
    if (s == null || s.waitStatus > 0) {
        s = null;
        // 从后往前找，仔细看代码，不必担心中间有节点取消(waitStatus==1)的情况
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    if (s != null)
        // 唤醒线程
        LockSupport.unpark(s.thread);
}
```

**B线程**

1、哨兵节点的后一个节点就是B节点，B节点的waitStatus == 0，所以就直接走唤醒线程那一步了。



### 唤醒之后

唤醒线程以后，被唤醒的线程将从以下代码中继续往前走：

```java
private final boolean parkAndCheckInterrupt() {
    LockSupport.park(this); // 刚刚线程被挂起在这里了
    return Thread.interrupted();
}
// 又回到这个方法了：acquireQueued(final Node node, int arg)，这个时候，node的前驱是head了
```

返回这个方法进行第三次循环

```java
//node还是B节点
final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();
                //A线程走了，B就可以tryacquire成功
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC，
                    failed = false;
                    return interrupted;
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```

1、B线程`tryAcquire()`成功之后就占有了state，也就是拿到了锁。

```java
 final boolean nonfairTryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                if (compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0) // overflow
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }

protected final void setExclusiveOwnerThread(Thread thread) {
        exclusiveOwnerThread = thread;
    }
```

2、此时state那里有B线程的引用`exclusiveOwnerThread`，队列里也有B线程的引用，需要把队列里的多余引用给GC掉。

3、AQS采用的是将head指向B节点成为新的哨兵节点，旧的哨兵节点因为没有任何引用指向了，慢慢就会被GC掉。



<img src="https://npm.elemecdn.com/youthlql@1.0.8/Java_concurrency/Source_code/Fourth_stage/0010.png">





# 公平锁和非公平锁

看了上面的源码，这个知识点应该是可以很轻松理解的。公平锁和非公平锁在源码层次只有几处不一样。



## 构造

ReentrantLock 默认采用非公平锁，除非你在构造方法中传入参数 true 。

```java
public ReentrantLock() {
    // 默认非公平锁
    sync = new NonfairSync();
}
public ReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
}
```

## 非公平锁的 lock 方法

```java
static final class NonfairSync extends Sync {
    final void lock() {
        // 1、和公平锁相比，这里会直接先进行一次CAS，成功就返回了。这是第一处不一样
        if (compareAndSetState(0, 1))
            setExclusiveOwnerThread(Thread.currentThread());
        else
            acquire(1);
    }
    // AbstractQueuedSynchronizer类的acquire(int arg)方法
    public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
    protected final boolean tryAcquire(int acquires) {
        return nonfairTryAcquire(acquires);
    }
}

final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        // 这里没有对队列进行判断，直接CAS抢，这是第二点不一样【对比请看下方公平锁的lock】
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```





## 公平锁的 lock 方法

```java
static final class FairSync extends Sync {
    final void lock() {
        acquire(1);
    }
    // AbstractQueuedSynchronizer类的acquire(int arg)方法
    public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
    protected final boolean tryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        if (c == 0) {
            // 2、和非公平锁相比，这里多了一个判断：是否有线程在队列列等待，有我就不抢了
            if (!hasQueuedPredecessors() &&
                compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        else if (current == getExclusiveOwnerThread()) {
            int nextc = c + acquires;
            if (nextc < 0)
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        return false;
    }
}
```



# 推荐

[CLH队列](https://coderbee.net/index.php/concurrent/20131115/577)