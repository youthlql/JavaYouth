---
title: 'Java并发体系-第二阶段-锁与同步-[3]'
tags:
  - Java并发
  - 原理
  - 源码
categories:
  - Java并发
  - 原理
keywords: Java并发，原理，源码
description: '万字系列长文讲解-Java并发体系-第二阶段,从C++和硬件方面讲解。'
cover: 'https://npm.elemecdn.com/lql_static@latest/logo/Java_concurrency.png'
abbrlink: 113a3931
date: 2020-10-08 22:10:58
---



# synchronized保证三大特性

**synchronized保证原子性的原理**

对num++;增加同步代码块后，保证同一时间只有一个线程操作num++;。就不会出现安全问题。



**synchronized保证可见性的原理**

synchronized保证可见性的原理，执行synchronized时，会对应lock原子操作会刷新工作内存中共享变 量的值。



**synchronized保证有序性的原理**

我们加synchronized后，依然会发生重排序，只不过我们有同步 代码块，可以保证只有一个线程执行同步代码中的代码。保证有序性。



# synchronized的特性

## 可重入特性

意思就是一个线程可以多次执行synchronized,重复获取同一把锁。

```java
/*
    目标:演示synchronized可重入
        1.自定义一个线程类
        2.在线程类的run方法中使用嵌套的同步代码块
        3.使用两个线程来执行
 */
public class Demo01 {
    public static void main(String[] args) {
        new MyThread().start();
        new MyThread().start();
    }

    public static void test01() {
        synchronized (MyThread.class) {
            String name = Thread.currentThread().getName();
            System.out.println(name + "进入了同步代码块2");
        }
    }
}

// 1.自定义一个线程类
class MyThread extends Thread {
    @Override
    public void run() {
        synchronized (MyThread.class) {
            System.out.println(getName() + "进入了同步代码块1");

            Demo01.test01();
        }
    }
}
```

**可重入原理**

synchronized的锁对象中有一个计数器（recursions变量）会记录线程获得几次锁.。在执行完同步代码块时，计数器的数量会-1，直到计数器的数量为0，就释放这个锁。可重入的好处

1. 可以避免死锁
2. 可以让我们更好的来封装代码



## 不可中断特性

**什么是不可中断**

一个线程获得锁后，另一个线程想要获得锁，必须处于阻塞或等待状态，如果第一个线程不释放锁，第 二个线程会一直阻塞或等待，不可被中断。

### synchronized不可中断演示

```java
public class Test {
    private static Object obj = new Object();
    public static void main(String[] args) throws InterruptedException {
        // 1.定义一个Runnable
        Runnable run = () -> {
            // 2.在Runnable定义同步代码块
            synchronized (obj) {
                String name = Thread.currentThread().getName();
                System.out.println(name + "进入同步代码块");
                // 保证不退出同步代码块
                try {
                    Thread.sleep(888888);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        };

        // 3.先开启一个线程来执行同步代码块
        Thread t1 = new Thread(run);
        t1.start();
        Thread.sleep(1000);
        // 4.后开启一个线程来执行同步代码块(阻塞状态)
        Thread t2 = new Thread(run);
        t2.start();

        // 5.停止第二个线程
        System.out.println("停止线程前");
        t2.interrupt();
        System.out.println("停止线程后");

        System.out.println(t1.getState());
        System.out.println(t2.getState());
    }
}
```

**输出结果：**

> Thread-0进入同步代码块
> 停止线程前
> 停止线程后
> TIMED_WAITING
> BLOCKED

### ReentrantLock可中断演示

```java
public class Test {
    private static Lock lock = new ReentrantLock();
    public static void main(String[] args) throws InterruptedException {
//        test01();
        test02();
    }

    // 演示Lock可中断
    public static void test02() throws InterruptedException {
        Runnable run = () -> {
            String name = Thread.currentThread().getName();
            boolean b = false;
            try {
                b = lock.tryLock(3, TimeUnit.SECONDS);
                if (b) {
                    System.out.println(name + "获得锁,进入锁执行");
                    Thread.sleep(88888);
                } else {
                    System.out.println(name + "在指定时间没有得到锁做其他操作");
                }
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                if (b) {
                    lock.unlock();
                    System.out.println(name + "释放锁");
                }
            }
        };

        Thread t1 = new Thread(run);
        t1.start();
        Thread.sleep(1000);
        Thread t2 = new Thread(run);
        t2.start();

        System.out.println("停止t2线程前");
        t2.interrupt();
        System.out.println("停止t2线程后");

        Thread.sleep(4000);
        System.out.println(t1.getState());
        System.out.println(t2.getState());
    }

    // 演示Lock不可中断
    public static void test01() throws InterruptedException {
        Runnable run = () -> {
            String name = Thread.currentThread().getName();
            try {
                lock.lock();
                System.out.println(name + "获得锁,进入锁执行");
                Thread.sleep(88888);
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                lock.unlock();
                System.out.println(name + "释放锁");
            }
        };

        Thread t1 = new Thread(run);
        t1.start();
        Thread.sleep(1000);
        Thread t2 = new Thread(run);
        t2.start();

        System.out.println("停止t2线程前");
        t2.interrupt();
        System.out.println("停止t2线程后");

        Thread.sleep(1000);
        System.out.println(t1.getState());
        System.out.println(t2.getState());
    }
}
```



控制台输出：

> Thread-0获得锁,进入锁执行
> 停止t2线程前
> 停止t2线程后
> java.lang.InterruptedException 
>
> atjava.util.concurrent.locks.AbstractQueuedSynchronizer.tryAcquireNanos(AbstractQueuedSynchronizer.java:1245)
> 	at java.util.concurrent.locks.ReentrantLock.tryLock(ReentrantLock.java:442)
> 	at Test.lambda$test02$0(Test.java:24)
> 	at java.lang.Thread.run(Thread.java:748)
> TIMED_WAITING
> TERMINATED

关于ReentranLock锁中断的原理，在AQS里讲。



# synchronized简单原理

> 我相信这个原理大部分人应该都知道，很多资料都讲过，我这里简单描述一下。



```java
public class SyncTest {
    public void syncBlock(){
        synchronized (this){
            System.out.println("hello block");
        }
    }
    public synchronized void syncMethod(){
        System.out.println("hello method");
    }
}
```

使用javap对其进行反汇编，部分信息如下

```java
{
  public void syncBlock();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=3, args_size=1
         0: aload_0
         1: dup
         2: astore_1
         3: monitorenter				 	  // monitorenter指令进入同步块
         4: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
         7: ldc           #3                  // String hello block
         9: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
        12: aload_1
        13: monitorexit						  // monitorexit指令退出同步块
        14: goto          22
        17: astore_2
        18: aload_1
        19: monitorexit						  // monitorexit指令退出同步块
        20: aload_2
        21: athrow
        22: return
      Exception table:
         from    to  target type
             4    14    17   any
            17    20    17   any
 

  public synchronized void syncMethod();
    descriptor: ()V
    flags: ACC_PUBLIC, ACC_SYNCHRONIZED      //添加了ACC_SYNCHRONIZED标记
    Code:
      stack=2, locals=1, args_size=1
         0: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
         3: ldc           #5                  // String hello method
         5: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
         8: return
 
}
```



## synchronized修饰代码块时

#### monitorenter

首先我们来看一下JVM规范中对于monitorenter的描述： https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-6.html#jvms-6.5.monitorenter

> Each object is associated with a monitor. A monitor is locked if and only if it has an owner. The thread that executes monitorenter attempts to gain ownership of the monitor associated with objectref， as follows: • If the entry count of the monitor associated with objectref is zero， the thread enters the monitor and sets its entry count to one. The thread is then the owner of the monitor. • If the thread already owns the monitor associated with objectref， it reenters the monitor， incrementing its entry count. • If another thread already owns the monitor associated with objectref， the thread blocks until the monitor's entry count is zero， then tries again to gain ownership.

翻译过来： 每一个对象都会和一个监视器monitor关联。监视器被占用时会被锁住，其他线程无法来获 取该monitor。 当JVM执行某个线程的某个方法内部的monitorenter时，它会尝试去获取当前对象对应 的monitor的所有权。其过程如下：

1. 若monior的进入数为0，线程可以进入monitor，并将monitor的进入数置为1。当前线程成为 monitor的owner（所有者）
2. 若线程已拥有monitor的所有权，允许它重入monitor，则进入monitor的进入数加1
3. 若其他线程已经占有monitor的所有权，那么当前尝试获取monitor的所有权的线程会被阻塞，直 到monitor的进入数变为0，才能重新尝试获取monitor的所有权。

monitorenter小结: synchronized的锁对象会关联一个monitor,这个monitor不是我们主动创建的,是JVM的线程执行到这个 同步代码块,发现锁对象没有monitor就会创建monitor,monitor内部有两个重要的成员变量owner:拥有 这把锁的线程,recursions会记录线程拥有锁的次数,当一个线程拥有monitor后其他线程只能等待

#### monitorexit

首先我们来看一下JVM规范中对于monitorexit的描述： https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-6.html#jvms-6.5.monitorexit

> The thread that executes monitorexit must be the owner of the monitor associated with the instance referenced by objectref. The thread decrements the entry count of the monitor associated with objectref. If as a result the value of the entry count is zero， the thread exits the monitor and is no longer its owner. Other threads that are blocking to enter the monitor are allowed to attempt to do so.

翻译过来：

1.能执行monitorexit指令的线程一定是拥有当前对象的monitor的所有权的线程。

1. 执行monitorexit时会将monitor的进入数减1。当monitor的进入数减为0时，当前线程退出 monitor，不再拥有monitor的所有权，此时其他被这个monitor阻塞的线程可以尝试去获取这个 monitor的所有权

monitorexit释放锁。 monitorexit插入在方法结束处和异常处，JVM保证每个monitorenter必须有对应的monitorexit。



**总结**：synchronized在修饰代码块时，是通过`monitorenter` 和 `monitorexit`来保证并发安全。



## synchronized 修饰方法的的情况

`synchronized` 修饰的方法并没有 `monitorenter` 指令和 `monitorexit` 指令，取得代之的确实是 `ACC_SYNCHRONIZED` 标识，该标识指明了该方法是一个同步方法。JVM 通过该 `ACC_SYNCHRONIZED` 访问标志来辨别一个方法是否声明为同步方法，从而执行相应的同步调用。

**不过两者的本质都是对对象监视器 monitor 的获取。**





# Java对象的布局（C++代码层面）

> 在学习synchronized最底层的C++源码级别前，我们需要先了解这个知识点，不然后面的可能看不懂

术语参考: http://openjdk.java.net/groups/hotspot/docs/HotSpotGlossary.html 在JVM中，对象在内存中的布局分为三块区域：对象头、实例数据和对齐填充。如下图所示：

<img src="https://npm.elemecdn.com/youthlql@1.0.8/Java_concurrency/Source_code/Second_stage/0021.png">

## 对象头

当一个线程尝试访问synchronized修饰的代码块时，它首先要获得锁，那么这个锁到底存在哪里呢？是 存在锁对象的对象头中的。 HotSpot采用instanceOopDesc和arrayOopDesc来描述对象头，arrayOopDesc对象用来描述数组类型。instanceOopDesc的定义的在Hotspot源码的 instanceOop.hpp 文件中，另外，arrayOopDesc 的定义对应 arrayOop.hpp 

<img src="https://npm.elemecdn.com/youthlql@1.0.8/Java_concurrency/Source_code/Second_stage/0022.png">

从instanceOopDesc代码中可以看到 instanceOopDesc继承自oopDesc，oopDesc的定义载Hotspot 源码中的 oop.hpp 文件中。

<img src="https://npm.elemecdn.com/youthlql@1.0.8/Java_concurrency/Source_code/Second_stage/0023.png">

- 在普通实例对象中，oopDesc的定义包含两个成员，分别是 _mark 和 _metadata

- _mark 表示对象标记、属于markOop类型，也就是接下来要讲解的Mark World，它记录了对象和锁有关的信息

- _metadata 表示类元信息，类元信息存储的是对象指向它的类元数据(Klass)的首地址，其中Klass表示 普通指针、 _compressed_klass 表示压缩类指针。

- 对象头由两部分组成，一部分用于存储自身的运行时数据，称之为 Mark Word，另外一部分是类型指 针，及对象指向它的类元数据的指针。

> 关于Klass，Class这些JVM的东西，推荐去看《深入理解Java虚拟机：JVM高级特性与最佳实践（第3版）》。强调一点，是第三版。第三版相比第二版改了太多东西了，我看第二版的时候总感觉讲的不够深入缺点什么东西，不过第三版补齐了很多东西。

### Mark Word

Mark Word用于存储对象自身的运行时数据，如哈希码（HashCode）、GC分代年龄、锁状态标志、 线程持有的锁、偏向线程ID、偏向时间戳等等，占用内存大小与虚拟机位长一致。Mark Word对应的类 型是 markOop 。源码位于 markOop.hpp 中。

<img src="https://npm.elemecdn.com/youthlql@1.0.8/Java_concurrency/Source_code/Second_stage/0024.png">

<img src="https://npm.elemecdn.com/youthlql@1.0.8/Java_concurrency/Source_code/Second_stage/0025.png">

在64位虚拟机下，Mark Word是64bit大小的，其存储结构如下：

<img src="https://npm.elemecdn.com/youthlql@1.0.8/Java_concurrency/Source_code/Second_stage/0026.png">

在32位虚拟机下，Mark Word是32bit大小的，其存储结构如下：

<img src="https://npm.elemecdn.com/youthlql@1.0.8/Java_concurrency/Source_code/Second_stage/0027.png">

再加一个图对比一下，有一丁点的补充

<img src="https://npm.elemecdn.com/youthlql@1.0.8/Java_concurrency/Source_code/Second_stage/0028.png">





### klass pointer

这一部分用于存储对象的类型指针，该指针指向它的类元数据，JVM通过这个指针确定对象是哪个类的 实例。该指针的位长度为JVM的一个字大小，即32位的JVM为32位，64位的JVM为64位。 如果应用的对 象过多，使用64位的指针将浪费大量内存，统计而言，64位的JVM将会比32位的JVM多耗费50%的内 存。为了节约内存可以使用选项**-XX:+UseCompressedOops** 开启指针压缩，其中，oop即ordinary object pointer普通对象指针。开启该选项后，下列指针将压缩至32位：

1. 每个Class的属性指针（即静态变量）
2. 每个对象的属性指针（即对象变量）
3. 普通对象数组的每个元素指针

当然，也不是所有的指针都会压缩，一些特殊类型的指针JVM不会优化，比如指向PermGen的Class对 象指针(JDK8中指向元空间的Class对象指针)、本地变量、堆栈元素、入参、返回值和NULL指针等。 对象头 = Mark Word + 类型指针（未开启指针压缩的情况下） 在32位系统中，Mark Word = 4 bytes，类型指针 = 4bytes，对象头 = 8 bytes = 64 bits； 在64位系统中，Mark Word = 8 bytes，类型指针 = 8bytes，对象头 = 16 bytes = 128bits；

## 实例数据

就是类中定义的成员变量。

## 对齐填充

对齐填充并不是必然存在的，也没有什么特别的意义，他仅仅起着占位符的作用，由于HotSpot VM的 自动内存管理系统要求对象起始地址必须是8字节的整数倍，换句话说，就是对象的大小必须是**8字节的 整数倍**。而对象头正好是8字节的倍数，因此，当对象实例数据部分没有对齐时，就需要通过对齐填充 来补全。



## 查看Java对象布局的方法

```java
<dependency>    
	<groupId>org.openjdk.jol</groupId>    
	<artifactId>jol-core</artifactId>    
	<version>0.9</version> 
</dependency>
```





# Lock Record

字面意思就是锁记录。通过对Java对象头的介绍可以看到锁信息也是存在于对象的`mark word`中的。当对象状态为偏向锁（biasable）时，`mark word`存储的是偏向的线程ID；当状态为轻量级锁（lightweight locked）时，`mark word`存储的是指向线程栈中`Lock Record`的指针；当状态为重量级锁（inflated）时，为指向堆中的monitor对象的指针。



## Lock Record的结构

线程在执行同步块之前，JVM会先在当前的线程的栈帧中创建一个`Lock Record`，其包括一个用于存储对象头中的 `mark word`（官方称之为`Displaced Mark Word`）以及一个指向对象的指针。下图右边的部分就是一个`Lock Record`。

<img src="https://npm.elemecdn.com/youthlql@1.0.8/Java_concurrency/Source_code/Second_stage/0029.png">





# synchronized偏向锁原理（C++源码层面）

## 概述

1、偏向锁是JDK 6中的重要引进，因为HotSpot作者经过研究实践发现，在大多数情况下，锁不仅不存在多线程竞争，而且总是由同一线程多次获得，为了让线程获得锁的代价更低，引进了偏向锁。 偏向锁的“偏”，就是偏心的“偏”、偏袒的“偏”，它的意思是这个锁会偏向于第一个获得它的线程，会在对 象头存储锁偏向的线程ID，以后该线程进入和退出同步块时只需要检查是否为偏向锁、锁标志位以及 ThreadID即可。

## 举例

Java是支持多线程的语言，因此在很多基础库中为了保证代码在多线程的情况下也能正常运行，也就是我们常说的线程安全，都会加入如`synchronized`这样的同步语义。但是在应用在实际运行时，很可能只有一个线程会调用相关同步方法。比如下面这个demo：

```java
public class Demo{
	private List<String> list = new ArrayList<>();
	
    public static void main(String[] args) {
        Demo Demo = new Demo();
        for (int i = 0; i < 50; i++) {
            Demo.add("Demo--->" + i);
        }
    }

    public synchronized void add(String s) {
        list.add(s);
    }

}
```

在这个demo中为了保证对list操纵时线程安全，对add方法加了`synchronized`的修饰，但实际使用时却只有一个线程调用到该方法。对于轻量级锁而言，每次调用add时，加锁解锁都有一个CAS操作；对于重量级锁而言，加锁也会有一个或多个CAS操作（这里的一个和多个 只是针对该demo，并不适用于所有场景）。



## 对象的Mark Word

- 偏向锁在Java 6之后是默认启用的，但在应用程序启动几秒钟之后才激活，可以使用-XX:BiasedLockingStartupDelay=0 参数关闭延迟，如果确定应用程序中所有锁通常情况下处于竞争 状态，可以通过 XX:-UseBiasedLocking=false 参数关闭偏向锁。
- 当JVM启用了偏向锁模式（1.6以上默认开启），当新创建一个对象的时候，如果该对象所属的class没有关闭偏向锁模式（什么时候会关闭一个class的偏向模式下文会说，默认所有class的偏向模式都是是开启的），那新创建对象的`mark word`将是可偏向状态，此时`mark word中`的thread id（参见上文偏向状态下的`mark word`格式）为0，表示未偏向任何线程，也叫做匿名偏向(anonymously biased)。



## 加锁过程

> 偏向锁加锁的C++代码过于复杂，这里只是用文字描述了几种情况。正常面试的时候，也不会让你说C++代码执行过程。

加锁过程分为几种情况（case），注意下面的不是顺序，只是加锁的几种情况。

case 1：当该对象第一次被线程获得锁的时候，发现是匿名偏向状态，则会用CAS指令，将`mark word`中的thread id由0改成当前线程Id。如果成功，则代表获得了偏向锁，继续执行同步块中的代码。否则，将偏向锁撤销，升级为轻量级锁。

case 2：当被偏向的线程再次进入同步块时，发现锁对象偏向的就是当前线程，在通过一些额外的检查后，会往当前线程的栈中添加一条`Displaced Mark Word`为空的`Lock Record`中，然后继续执行同步块的代码，因为操纵的是线程私有的栈，因此不需要用到CAS指令；由此可见偏向锁模式下，当被偏向的线程再次尝试获得锁时，仅仅进行几个简单的操作就可以了，在这种情况下，`synchronized`关键字带来的性能开销基本可以忽略。

case 3：当其他线程进入同步块时，发现已经有偏向的线程了，则会进入到**撤销偏向锁**的逻辑里，一般来说，会在`safepoint`中去查看偏向的线程是否还存活，如果存活且还在同步块中则将锁升级为轻量级锁，原偏向的线程继续拥有锁，当前线程则走入到锁升级的逻辑里；如果偏向的线程已经不存活或者不在同步块中，则将对象头的`mark word`改为无锁状态（unlocked），之后再升级为轻量级锁。

## 解锁过程

当有其他线程尝试获得锁时，是根据遍历偏向线程的`lock record`来确定该线程是否还在执行同步块中的代码。因此偏向锁的解锁很简单，仅仅将栈中的**最近一条**`lock record`的`obj`字段设置为null。偏向锁的解锁步骤中**并不会修改对象头中的thread id。**



## 偏向锁升级时机

一般来说（批量重偏向除外），偏向锁升级的时机为：当锁已经发生偏向后，只要有另一个线程尝试获得偏向锁，则该偏向锁就会升级成轻量级锁。



## 偏向锁撤销过程

这里说的撤销是指在获取偏向锁的过程因为不满足条件导致要将锁对象改为非偏向锁状态；释放是指退出同步块时的过程。

> 撤销逻辑有很多，我们只分析最常见的情况：假设锁已经偏向线程A，这时B线程尝试获得锁。

1. 查看偏向的线程是否存活，如果已经不存活了，则直接撤销偏向锁。JVM维护了一个集合存放所有存活的线程，通过遍历该集合判断某个线程是否存活。
2. 偏向的线程是否还在同步块中，如果不在了，则撤销偏向锁。我们回顾一下偏向锁的加锁流程：每次进入同步块（即执行`monitorenter`）的时候都会以从高往低的顺序在栈中找到第一个可用的`Lock Record`，将其obj字段指向锁对象。每次解锁（即执行`monitorexit`）的时候都会将最低的一个相关`Lock Record`移除掉。所以可以通过遍历线程栈中的`Lock Record`来判断线程是否还在同步块中。
3. 将偏向线程所有相关`Lock Record`的`Displaced Mark Word`设置为null，然后将最高位的`Lock Record`的`Displaced Mark Word` 设置为无锁状态，最高位的`Lock Record`也就是第一次获得锁时的`Lock Record`（这里的第一次是指重入获取锁时的第一次），然后将对象头指向最高位的`Lock Record`，这里不需要用CAS指令，因为是在`safepoint`。 执行完后，就升级成了轻量级锁。原偏向线程的所有Lock Record都已经变成轻量级锁的状态。【轻量级锁加锁过程会在下文讲到，不要慌】



触发时机：
释放：对应的就是synchronized方法的退出或synchronized块的结束。
撤销：笼统的说就是多个线程竞争导致不能再使用偏向模式的时候。



# synchronized轻量级锁原理（C++源码层面）

## 加锁过程

1.在线程栈中创建一个`Lock Record`，将其`obj`（即Object reference）字段指向锁对象。

2.会把锁的Mark Word复制到自己的Lock Record的Displaced Mark Word里面。然后线程尝试直接通过CAS指令将`Lock Record`的地址存储在对象头的`mark word`中，如果对象处于无锁状态则修改成功，代表该线程获得了轻量级锁。如果失败，进入到步骤3。

3.如果是当前线程已经持有该锁了，代表这是一次锁重入。设置`Lock Record`第一部分（`Displaced Mark Word`）为null，起到了一个重入计数器的作用。然后结束。

4.如果都失败，表示Mark Word已经被替换成了其他线程的锁记录，说明在与其它线程竞争锁，需要膨胀为重量级锁。【这就是轻量级锁升级为重量级锁的时机】



## 解锁过程

1.遍历线程栈,找到所有`obj`字段等于当前锁对象的`Lock Record`。

2.如果`Lock Record`的`Displaced Mark Word`为null，代表这是一次重入，将`obj`设置为null后continue。

3.如果`Lock Record`的`Displaced Mark Word`不为null，则利用CAS指令将对象头的`mark word`恢复成为`Displaced Mark Word`。如果成功，则continue，否则膨胀为重量级锁。



## 轻量级锁重入示例图

我们看个demo，在该demo中重复3次获得锁。

```java
synchronized(obj){
    synchronized(obj){
    	synchronized(obj){
    	}
    }
}
```

<img src="https://npm.elemecdn.com/youthlql@1.0.8/Java_concurrency/Source_code/Second_stage/0030.png">

## 轻量级锁什么时候升级为重量级锁？

> 其实在加锁的时候已经说过了，这里再以一个具体场景说下

- 线程1获取轻量级锁时会把锁的Mark Word复制到自己的Lock Record的Displaced Mark Word里面。然后线程尝试直接通过CAS指令将`Lock Record`的地址存储在对象头的`mark word`中

- 如果在线程1复制对象头的同时（在线程1CAS之前），线程2也准备获取锁，复制了对象头到线程2的锁记录空间中，但是在线程2在CAS的时候，发现线程1已经把对象头换了，**线程2的CAS失败**。那么此时就代表发生了锁竞争，准备升级为重量级锁





## 轻量级锁CAS的问题

1、**结论：**没有自旋这回事，只有重量级锁获取失败才会自旋，网上的文章好多都是错的，我个人认为轻量级锁的意义就是在没有线程争用锁时不用创建monitor。**【源码得到的结论，实践才是硬道理】**

2、**轻量级锁和偏向锁区别：**只要存在竞争就会升级重量级。**轻量级锁的存在就是用于线程之间交替获取锁的场景**，但是和偏向锁是有区别的啊。一个线程获取偏向锁之后，那么这个锁自然而然就属于这个线程（就算该线程释放了偏向锁也不会改变这把锁偏向这个线程的【也就是之前说的不会修改Thread ID】，这个前提是没有发生过批量重偏向使锁的epoch与其对应class类的epoch不相等）。所以说偏向锁的场景是用于一个线程不断的获取锁，如果把它放在轻量级锁的场景下线程之间交替获取的话会发生偏向锁的撤销的。也就是说在偏向锁的情况下，线程1之前释放了锁，线程2再获取锁，即使此时没有**同时锁竞争**的情况，依然是要升级为轻量级锁的。而轻量级锁只要没有同时去获取锁，就可以不升级为重量级锁，也就代表你可以不同线程交替获取这个锁。
3、效率上来看偏向锁只有在获取的时候进行一次CAS，以后的释放和获取只需要简单的一些判断操作。而轻量级锁的获取和释放都要都要CAS，单纯的看效率还是偏向锁效率高。



# synchronized重量级锁原理（C++源码层面）

> 重量级锁面试可能问的多，就多写了点C++代码

当出现多个线程**同时竞争**锁时，如果不是**同时竞争**，轻量级锁依然可以实现线程交替运行。



## Monitor监视器锁

- 重量级锁通过对象的监视器（monitor）实现，其中monitor的本质是依赖于底层操作系统的互斥量（mutex） 实现的实现，操作系统实现线程之间的切换需要从用户态到内核态的切换，切换成本非常高。这也是为什么重量级锁效率不高的原因。

- 重量级锁的状态下，对象的`mark word`为指向一个堆中monitor对象的指针。一个monitor对象包括这么几个关键字段：cxq（下图中的ContentionList），EntryList ，WaitSet，owner。其中cxq ，EntryList ，WaitSet都是由ObjectWaiter的链表结构，owner指向持有锁的线程。

<img src="https://npm.elemecdn.com/youthlql@1.0.8/Java_concurrency/Source_code/Second_stage/0031.png">

在HotSpot虚拟机中，monitor是由ObjectMonitor实现的。其源码是用c++来实现的，位于HotSpot虚 拟机源码ObjectMonitor.hpp文件中(src/share/vm/runtime/objectMonitor.hpp)。ObjectMonitor主 要数据结构如下：

```C++
ObjectMonitor() {    
	_header       = NULL;    
	_count        = 0;   
    _waiters      = 0，    
    _recursions   = 0;  // 线程的重入次数
    _object       = NULL; // 存储该monitor的对象    
    _owner        = NULL; // 标识拥有该monitor的线程    
    _WaitSet      = NULL; // 处于wait状态的线程，会被加入到_WaitSet    
    _WaitSetLock  = 0 ;    
    _Responsible  = NULL;    
    _succ         = NULL;    
    _cxq          = NULL; // 多线程竞争锁时的单向列表    
    FreeNext      = NULL;    
    _EntryList    = NULL; // 处于等待锁block状态的线程，会被加入到该列表   
    _SpinFreq     = 0;   
    _SpinClock    = 0;    
    OwnerIsThread = 0; 
}
```

> Contention List：所有请求锁的线程将被首先放置到该竞争队列
> Entry List：Contention List中那些有资格成为候选人的线程被移到Entry List
> Wait Set：那些调用wait方法被阻塞的线程被放置到Wait Set
> OnDeck：任何时刻最多只能有一个线程正在竞争锁，该线程称为OnDeck
> Owner：获得锁的线程称为Owner
> !Owner：释放锁的线程

1、当一个线程尝试获得锁时，如果该锁已经被占用，则会将该线程封装成一个`ObjectWaiter`对象插入到Contention List的队列的队首，然后调用`park`函数挂起当前线程。

2、当线程释放锁时，会从Contention List或EntryList中挑选一个线程唤醒，被选中的线程叫做`Heir presumptive`即假定继承人，假定继承人【Ready线程】被唤醒后会尝试获得锁，但`synchronized`是非公平的，所以假定继承人不一定能获得锁。这是因为对于重量级锁，线程先自旋尝试获得锁，这样做的目的是为了减少执行操作系统同步操作带来的开销。如果自旋不成功再进入等待队列。这对那些已经在等待队列中的线程来说，稍微显得不公平。

3、如果线程获得锁后调用`Object.wait`方法，则会将线程加入到WaitSet中，当被`Object.notify`唤醒后，会将线程从WaitSet移动到Contention List或EntryList中去。需要注意的是，当调用一个锁对象的`wait`或`notify`方法时，**如当前锁的状态是偏向锁或轻量级锁则会先膨胀成重量级锁**。

4、**关于Contention List（cxq）和EntryList的区别：**cxq是单向链表，指的是如果已经有t1线程获取到monitor对象拿到锁后，t2和t3没有竞争到，t2、t3线程会进行到cxq队列，先自己尝试竞争锁，如果竞争不到则自旋再去挣扎一下获取锁，当t1执行完同步代码块，释放锁后，由t1、t2、t3再去争抢锁，如果t1再次抢到锁，那么t2、t3会进行到EntryList阻塞队列，如果此时又有t4、t5线程过来会被放到cxq队列，t2,t3,t4,t5，通过自旋尝试获取锁，如果还是没有获取到锁，则通过park将当 前线程挂起，等待被唤醒。如果t1被释放， 根据不同的策略（由QMode指定），从cxq或EntryList中获取头节点，通过 ObjectMonitor::ExitEpilog 方法唤醒该节点封装的线程，唤醒操作终由unpark完成，被唤醒的线程，继续执行monitor 的竞争。当获取锁的线程释放后，EntryList中的线程和WaitSet中的线程被唤醒都可能去获取锁变成owner的拥有者。



- 每一个Java对象都可以与一个监视器monitor关联，我们可以把它理解成为一把锁，当一个线程想要执行一段被synchronized圈起来的同步方法或者代码块时，该线程得先获取到synchronized修饰的对象 对应的monitor。 我们的Java代码里不会显示地去创造这么一个monitor对象，我们也无需创建，事实上可以这么理解： monitor并不是随着对象创建而创建的。我们是通过synchronized修饰符告诉JVM需要为我们的某个对 象创建关联的monitor对象。每个线程都存在两个ObjectMonitor对象列表，分别为free和used列表。 同时JVM中也维护着global locklist。当线程需要ObjectMonitor对象时，首先从线程自身的free表中申请，若存在则使用，若不存在则从global list中分配一批`monitor`到free中。

- free对应C++代码：omFreeList
- global locklist对应C++代码：gFreeList



## monitor竞争

1、执行monitorenter时，会调用InterpreterRuntime.cpp (位于：src/share/vm/interpreter/interpreterRuntime.cpp) 的 InterpreterRuntime::monitorenter函 数。具体代码可参见HotSpot源码。

<img src="https://npm.elemecdn.com/youthlql@1.0.8/Java_concurrency/Source_code/Second_stage/0032.png">



2、对于重量级锁，monitorenter函数中会调用 ObjectSynchronizer::slow_enter 3.终调用 ObjectMonitor::enter（位于：src/share/vm/runtime/objectMonitor.cpp），源码如下：



```C++
void ATTR ObjectMonitor::enter(TRAPS) {
   
  Thread * const Self = THREAD ;
  void * cur ;
  // owner为null代表无锁状态，如果能CAS设置成功，则当前线程直接获得锁
  cur = Atomic::cmpxchg_ptr (Self, &_owner, NULL) ;
  if (cur == NULL) {
     ...
     return ;
  }
  // 如果是重入的情况，_recursions++
  if (cur == Self) {
     // TODO-FIXME: check for integer overflow!  BUGID 6557169.
     _recursions ++ ;
     return ;
  }
  /* 
  1、当前线程是之前持有轻量级锁的线程。由轻量级锁膨胀且第一次调用enter方法，那cur是指向Lock Record的指针。
  */
  if (Self->is_lock_owned ((address)cur)) {
    assert (_recursions == 0, "internal state error");
    // 重入计数重置为1
    _recursions = 1 ;
    // 设置owner字段为当前线程（之前owner是指向Lock Record的指针）
    _owner = Self ;
    OwnerIsThread = 1 ;
    return ;
  }

  ...

  // 在调用系统的同步操作之前，先尝试自旋获得锁
  if (Knob_SpinEarly && TrySpin (Self) > 0) {
     ...
     //自旋的过程中获得了锁，则直接返回
     Self->_Stalled = 0 ;
     return ;
  }

  ...

  { 
    ...

    for (;;) {
      jt->set_suspend_equivalent();
      // 在该方法中调用系统同步操作。也就是获得锁或阻塞
      EnterI (THREAD) ;
      ...
    }
    Self->set_current_pending_monitor(NULL);
    
  }

  ...

}
```



## Monitor等待或获取锁

```C++
void ATTR ObjectMonitor::EnterI (TRAPS) {
    Thread * Self = THREAD ;
    ...
    // 尝试获得锁
    if (TryLock (Self) > 0) {
        ...
        return ;
    }

    DeferredInitialize () ;
 
	// 自旋
    if (TrySpin (Self) > 0) {
        ...
        return ;
    }
    
    ...
	
    // 当前线程被封装成ObjectWaiter对象node
    ObjectWaiter node(Self) ;
    Self->_ParkEvent->reset() ;
    node._prev   = (ObjectWaiter *) 0xBAD ;
    node.TState  = ObjectWaiter::TS_CXQ ;

    // 通过CAS将node节点插入到_cxq队列的头部，cxq是一个单向链表
    ObjectWaiter * nxt ;
    for (;;) {
        node._next = nxt = _cxq ;
        if (Atomic::cmpxchg_ptr (&node, &_cxq, nxt) == nxt) break ;

        // CAS失败的话 再尝试获得锁，这样可以降低插入到_cxq队列的频率
        if (TryLock (Self) > 0) {
            ...
            return ;
        }
    }

	// SyncFlags默认为0，如果没有其他等待的线程，则将_Responsible设置为自己
    if ((SyncFlags & 16) == 0 && nxt == NULL && _EntryList == NULL) {
        Atomic::cmpxchg_ptr (Self, &_Responsible, NULL) ;
    }


    TEVENT (Inflated enter - Contention) ;
    int nWakeups = 0 ;
    int RecheckInterval = 1 ;

    for (;;) {
		//线程在被挂起前再做一下挣扎，看能不能获取到锁
        if (TryLock (Self) > 0) break ;
        assert (_owner != Self, "invariant") ;

        ...

        // park self
        if (_Responsible == Self || (SyncFlags & 1)) {
            // 当前线程是_Responsible时，调用的是带时间参数的park
            TEVENT (Inflated enter - park TIMED) ;
            Self->_ParkEvent->park ((jlong) RecheckInterval) ;
            // Increase the RecheckInterval, but clamp the value.
            RecheckInterval *= 8 ;
            if (RecheckInterval > 1000) RecheckInterval = 1000 ;
        } else {
            //否则直接调用park挂起当前线程
            TEVENT (Inflated enter - park UNTIMED) ;
            Self->_ParkEvent->park() ;
        }

        if (TryLock(Self) > 0) break ;

        ...
        
        if ((Knob_SpinAfterFutile & 1) && TrySpin (Self) > 0) break ;

       	...
        // 在释放锁时，_succ会被设置为EntryList或_cxq中的一个线程
        if (_succ == Self) _succ = NULL ;

        // Invariant: after clearing _succ a thread *must* retry _owner before parking.
        OrderAccess::fence() ;
    }

   // 走到这里说明已经获得锁了

    assert (_owner == Self      , "invariant") ;
    assert (object() != NULL    , "invariant") ;
  
	// 将当前线程的node从cxq或EntryList中移除
    UnlinkAfterAcquire (Self, &node) ;
    if (_succ == Self) _succ = NULL ;
	if (_Responsible == Self) {
        _Responsible = NULL ;
        OrderAccess::fence();
    }
    ...
    return ;
}
```

当该线程被唤醒时，会从挂起的点继续执行，通过 ObjectMonitor::TryLock 尝试获取锁，TryLock方 法实现如下：

```C++
int ObjectMonitor::TryLock (Thread * Self) {
	for (;;) {
		void * own = _owner ;
		if (own != NULL) return 0 ;
		if (Atomic::cmpxchg_ptr (Self， &_owner， NULL) == NULL) {
			// Either guarantee _recursions == 0 or set _recursions = 0.		
			assert (_recursions == 0， "invariant") ;
			assert (_owner == Self， "invariant") ;
			// CONSIDER: set or assert that OwnerIsThread == 1
			return 1 ;
		}
		// The lock had been free momentarily， but we lost the race to the lock.
		// Interference -- the CAS failed.
		// We can either return -1 or retry.
		// Retry doesn't make as much sense because the lock was just acquired.
		if (true) return -1 ;
	}
}
```

以上代码的具体流程概括如下：

1. 当前线程被封装成ObjectWaiter对象node，状态设置成ObjectWaiter::TS_CXQ。
2. 在for循环中，通过CAS把node节点push到_cxq列表中，同一时刻可能有多个线程把自己的node 节点push到_cxq列表中。
3. node节点push到_cxq列表之后，通过自旋尝试获取锁，如果还是没有获取到锁，则通过park将当 前线程挂起，等待被唤醒。
4. 当该线程被唤醒时，会从挂起的点继续执行，通过 ObjectMonitor::TryLock 尝试获取锁。



## monitor释放

当某个持有锁的线程执行完同步代码块时，会进行锁的释放，给其它线程机会执行同步代码，在 HotSpot中，通过退出monitor的方式实现锁的释放，并通知被阻塞的线程，具体实现位于 ObjectMonitor的exit方法中。（位于：src/share/vm/runtime/objectMonitor.cpp），源码如下所 示：

````C++
void ATTR ObjectMonitor::exit(bool not_suspended, TRAPS) {
   Thread * Self = THREAD ;
   // 如果_owner不是当前线程
   if (THREAD != _owner) {
     // 当前线程是之前持有轻量级锁的线程。由轻量级锁膨胀后还没调用过enter方法，_owner会是指向Lock Record的指针。
     if (THREAD->is_lock_owned((address) _owner)) {
       assert (_recursions == 0, "invariant") ;
       _owner = THREAD ;
       _recursions = 0 ;
       OwnerIsThread = 1 ;
     } else {
       // 异常情况:当前不是持有锁的线程
       TEVENT (Exit - Throw IMSX) ;
       assert(false, "Non-balanced monitor enter/exit!");
       if (false) {
          THROW(vmSymbols::java_lang_IllegalMonitorStateException());
       }
       return;
     }
   }
   // 重入计数器还不为0，则计数器-1后返回
   if (_recursions != 0) {
     _recursions--;        // this is simple recursive enter
     TEVENT (Inflated exit - recursive) ;
     return ;
   }

   // _Responsible设置为null
   if ((SyncFlags & 4) == 0) {
      _Responsible = NULL ;
   }

   ...

   for (;;) {
      assert (THREAD == _owner, "invariant") ;

      // Knob_ExitPolicy默认为0
      if (Knob_ExitPolicy == 0) {
         // code 1：先释放锁，这时如果有其他线程进入同步块则能获得锁
         OrderAccess::release_store_ptr (&_owner, NULL) ;   // drop the lock
         OrderAccess::storeload() ;                         // See if we need to wake a successor
         // code 2：如果没有等待的线程或已经有假定继承人
         if ((intptr_t(_EntryList)|intptr_t(_cxq)) == 0 || _succ != NULL) {
            TEVENT (Inflated exit - simple egress) ;
            return ;
         }
         TEVENT (Inflated exit - complex egress) ;

         // code 3：要执行之后的操作需要重新获得锁，即设置_owner为当前线程
         if (Atomic::cmpxchg_ptr (THREAD, &_owner, NULL) != NULL) {
            return ;
         }
         TEVENT (Exit - Reacquired) ;
      } 
      ...

      ObjectWaiter * w = NULL ;
      // code 4：根据QMode的不同会有不同的唤醒策略，默认为0
      int QMode = Knob_QMode ;
	  // QMode == 2 : cxq中的线程有更高优先级，直接绕过EntryList队列，唤醒cxq的队首线程
      if (QMode == 2 && _cxq != NULL) {
         
          w = _cxq ;
          assert (w != NULL, "invariant") ;
          assert (w->TState == ObjectWaiter::TS_CXQ, "Invariant") ;
          ExitEpilog (Self, w) ;
          return ;
      }
	 // QMode == 3 将cxq中的元素插入到EntryList的末尾
      if (QMode == 3 && _cxq != NULL) {
          
          w = _cxq ;
          for (;;) {
             assert (w != NULL, "Invariant") ;
             ObjectWaiter * u = (ObjectWaiter *) Atomic::cmpxchg_ptr (NULL, &_cxq, w) ;
             if (u == w) break ;
             w = u ;
          }
          assert (w != NULL              , "invariant") ;

          ObjectWaiter * q = NULL ;
          ObjectWaiter * p ;
          for (p = w ; p != NULL ; p = p->_next) {
              guarantee (p->TState == ObjectWaiter::TS_CXQ, "Invariant") ;
              p->TState = ObjectWaiter::TS_ENTER ;
              p->_prev = q ;
              q = p ;
          }

          // Append the RATs to the EntryList
          // TODO: organize EntryList as a CDLL so we can locate the tail in constant-time.
          ObjectWaiter * Tail ;
          for (Tail = _EntryList ; Tail != NULL && Tail->_next != NULL ; Tail = Tail->_next) ;
          if (Tail == NULL) {
              _EntryList = w ;
          } else {
              Tail->_next = w ;
              w->_prev = Tail ;
          }

          // Fall thru into code that tries to wake a successor from EntryList
      }
	  // QMode == 4，将cxq插入到EntryList的队首
      if (QMode == 4 && _cxq != NULL) {
          
          w = _cxq ;
          for (;;) {
             assert (w != NULL, "Invariant") ;
             ObjectWaiter * u = (ObjectWaiter *) Atomic::cmpxchg_ptr (NULL, &_cxq, w) ;
             if (u == w) break ;
             w = u ;
          }
          assert (w != NULL              , "invariant") ;

          ObjectWaiter * q = NULL ;
          ObjectWaiter * p ;
          for (p = w ; p != NULL ; p = p->_next) {
              guarantee (p->TState == ObjectWaiter::TS_CXQ, "Invariant") ;
              p->TState = ObjectWaiter::TS_ENTER ;
              p->_prev = q ;
              q = p ;
          }

          // Prepend the RATs to the EntryList
          if (_EntryList != NULL) {
              q->_next = _EntryList ;
              _EntryList->_prev = q ;
          }
          _EntryList = w ;

          // Fall thru into code that tries to wake a successor from EntryList
      }

      w = _EntryList  ;
      if (w != NULL) {
          // 如果EntryList不为空，则直接唤醒EntryList的队首元素
          assert (w->TState == ObjectWaiter::TS_ENTER, "invariant") ;
          ExitEpilog (Self, w) ;
          return ;
      }

      // EntryList为null，则处理cxq中的元素
      w = _cxq ;
      if (w == NULL) continue ;

      // 因为之后要将cxq的元素移动到EntryList，所以这里将cxq字段设置为null
      for (;;) {
          assert (w != NULL, "Invariant") ;
          ObjectWaiter * u = (ObjectWaiter *) Atomic::cmpxchg_ptr (NULL, &_cxq, w) ;
          if (u == w) break ;
          w = u ;
      }
      TEVENT (Inflated exit - drain cxq into EntryList) ;

      assert (w != NULL              , "invariant") ;
      assert (_EntryList  == NULL    , "invariant") ;


      if (QMode == 1) {
         // QMode == 1 : 将cxq中的元素转移到EntryList，并反转顺序
         ObjectWaiter * s = NULL ;
         ObjectWaiter * t = w ;
         ObjectWaiter * u = NULL ;
         while (t != NULL) {
             guarantee (t->TState == ObjectWaiter::TS_CXQ, "invariant") ;
             t->TState = ObjectWaiter::TS_ENTER ;
             u = t->_next ;
             t->_prev = u ;
             t->_next = s ;
             s = t;
             t = u ;
         }
         _EntryList  = s ;
         assert (s != NULL, "invariant") ;
      } else {
         // QMode == 0 or QMode == 2‘
         // 将cxq中的元素转移到EntryList
         _EntryList = w ;
         ObjectWaiter * q = NULL ;
         ObjectWaiter * p ;
         for (p = w ; p != NULL ; p = p->_next) {
             guarantee (p->TState == ObjectWaiter::TS_CXQ, "Invariant") ;
             p->TState = ObjectWaiter::TS_ENTER ;
             p->_prev = q ;
             q = p ;
         }
      }


      // _succ不为null，说明已经有个继承人了，所以不需要当前线程去唤醒，减少上下文切换的比率
      if (_succ != NULL) continue;

      w = _EntryList  ;
      // 唤醒EntryList第一个元素
      if (w != NULL) {
          guarantee (w->TState == ObjectWaiter::TS_ENTER, "invariant") ;
          ExitEpilog (Self, w) ;
          return ;
      }
   }
}
````

在进行必要的锁重入判断以及自旋优化后，进入到主要逻辑：

`code 1` 设置owner为null，即释放锁，这个时刻其他的线程能获取到锁。这里是一个非公平锁的优化；

`code 2` 如果当前没有等待的线程则直接返回就好了，因为不需要唤醒其他线程。或者如果说succ不为null，代表当前已经有个"醒着的"继承人线程，那当前线程不需要唤醒任何线程；

`code 3` 当前线程重新获得锁，因为之后要操作cxq和EntryList队列以及唤醒线程；

`code 4`根据QMode的不同，会执行不同的唤醒策略；

根据QMode的不同，有不同的处理方式：

1. QMode = 2且cxq非空：取cxq队列队首的ObjectWaiter对象，调用ExitEpilog方法，该方法会唤醒ObjectWaiter对象的线程，然后立即返回，后面的代码不会执行了；
2. QMode = 3且cxq非空：把cxq队列插入到EntryList的尾部；
3. QMode = 4且cxq非空：把cxq队列插入到EntryList的头部；
4. QMode = 0：暂时什么都不做，继续往下看；

只有QMode=2的时候会提前返回，等于0、3、4的时候都会继续往下执行：

1.如果EntryList的首元素非空，就取出来调用ExitEpilog方法，该方法会唤醒ObjectWaiter对象的线程，然后立即返回；
2.如果EntryList的首元素为空，就将cxq的所有元素放入到EntryList中，然后再从EntryList中取出来队首元素执行ExitEpilog方法，然后立即返回；



1. 退出同步代码块时会让_recursions减1，当_recursions的值减为0时，说明线程释放了锁。
2. 根据不同的策略（由QMode指定），从cxq或EntryList中获取头节点，通过 ObjectMonitor::ExitEpilog 方法唤醒该节点封装的线程，唤醒操作终由unpark完成，实现 如下：

```Java
void ObjectMonitor::ExitEpilog (Thread * Self， ObjectWaiter * Wakee) {
	assert (_owner == Self， "invariant") ;
	_succ = Knob_SuccEnabled ? Wakee->_thread : NULL ;
	P
	arkEvent * Trigger = Wakee->_event ;
	Wakee = NULL ;
// Drop the lock
	OrderAccess::release_store_ptr (&_owner， NULL) ;
	OrderAccess::fence() ; // ST _owner vs LD in unpark()
	if (SafepointSynchronize::do_call_back()) {
		TEVENT (unpark before SAFEPOINT) ;
	}
	DTRACE_MONITOR_PROBE(contended__exit， this， object()， Self);
	Trigger->unpark() ; // 唤醒之前被pack()挂起的线程.
// Maintain stats and report events to JVMTI
	if (ObjectMonitor::_sync_Parks != NULL) {
		ObjectMonitor::_sync_Parks->inc() ;
	}
}
```



被唤醒的线程，会回到 void ATTR ObjectMonitor::EnterI (TRAPS) 的第600行，继续执行monitor 的竞争。

```C++
// park self
if (_Responsible == Self || (SyncFlags & 1)) {
	TEVENT (Inflated enter - park TIMED) ;
	Self->_ParkEvent->park ((jlong) RecheckInterval) ;
// Increase the RecheckInterval， but clamp the value.
	RecheckInterval *= 8 ;
	if (RecheckInterval > 1000) RecheckInterval = 1000 ;
} else {
	TEVENT (Inflated enter - park UNTIMED) ;
	Self->_ParkEvent->park() ;
}
if (TryLock(Self) > 0) break ;
```

## monitor是重量级锁

可以看到ObjectMonitor的函数调用中会涉及到Atomic::cmpxchg_ptr，Atomic::inc_ptr等内核函数， 执行同步代码块，没有竞争到锁的对象会park()被挂起，竞争到锁的线程会unpark()唤醒。这个时候就 会存在操作系统用户态和内核态的转换，这种切换会消耗大量的系统资源。所以synchronized是Java语 言中是一个重量级(Heavyweight)的操作。 用户态和和内核态是什么东西呢？要想了解用户态和内核态还需要先了解一下Linux系统的体系架构：

<img src="https://npm.elemecdn.com/youthlql@1.0.8/Java_concurrency/Source_code/Second_stage/0033.png">

从上图可以看出，Linux操作系统的体系架构分为：用户空间（应用程序的活动空间）和内核。 内核：本质上可以理解为一种软件，控制计算机的硬件资源，并提供上层应用程序运行的环境。 用户空间：上层应用程序活动的空间。应用程序的执行必须依托于内核提供的资源，包括CPU资源、存 储资源、I/O资源等。 系统调用：为了使上层应用能够访问到这些资源，内核必须为上层应用提供访问的接口：即系统调用。

所有进程初始都运行于用户空间，此时即为用户运行状态（简称：用户态）；但是当它调用系统调用执 行某些操作时，例如 I/O调用，此时需要陷入内核中运行，我们就称进程处于内核运行态（或简称为内 核态）。 系统调用的过程可以简单理解为：

1. 用户态程序将一些数据值放在寄存器中， 或者使用参数创建一个堆栈， 以此表明需要操作系统提供的服务。
2. 用户态程序执行系统调用。
3. CPU切换到内核态，并跳到位于内存指定位置的指令。
4. 系统调用处理器(system call handler)会读取程序放入内存的数据参数，并执行程序请求的服务。
5. 系统调用完成后，操作系统会重置CPU为用户态并返回系统调用的结果。 由此可见用户态切换至内核态需要传递许多变量，同时内核还需要保护好用户态在切换时的一些寄存器值、变量等，以备内核态切换回用户态。这种切换就带来了大量的系统资源消耗，这就是在 synchronized未优化之前，效率低的原因。



# 锁降级的争论

1、先说结论，**在openjdk的hotsopt jdk8u里是有锁降级的机制的**，锁降级是什么时候加入到hotspot的这个我没去关注，所以我只说看过代码的jdk8u版本，另外根据R大的这个[回答](https://www.zhihu.com/question/19882320)，我相信sunj dk也一样。

2、然后再详细说：

- 锁降级的代码在[`deflate_idle_monitors`](http://hg.openjdk.java.net/jdk8u/jdk8u/hotspot/file/9ce27f0a4683/src/share/vm/runtime/synchronizer.cpp#l1503)方法中，其调用点在进入SafePoint的方法`SafepointSynchronize::begin()`中。
  在`deflate_idle_monitors`中会找到已经`idle`的monitor(也就是重量级锁的对象)，然后调用`deflate_monitor`方法将其降级。

- 因为锁降级是发生在safepoint的，所以如果降级时间过长会导致程序一直处于STW的阶段。在[这里](http://openjdk.java.net/jeps/8183909)有篇文章讨论了优化机制。jdk8中本身也有个`MonitorInUseLists`的开关，其影响了寻找`idle monitor`的方式，对该开关的一些讨论看[这里](https://bugs.openjdk.java.net/browse/JDK-8149442)。

- 至于为什么《java并发编程的艺术》中说锁不能降级，我**猜测**可能该书作者看的jdk版本还没有引入降级机制。





# 细节/容易混淆的地方

## java语言规范

1、java语言规范里面，`int i = 0，resource = loadedResoures，flag = true`，各种变量的简单的赋值操作，规定都是原子的包括引用类型的变量的赋值写操作，也是原子的。

2、但是很多复杂的一些操作，i++，先读取i的值，再跟新i的值，i = y + 2，先读取y的值，再更新i的值，这种复杂操作，不是简单赋值写，他是有计算的过程在里面的，此时java语言规范默认是不保证原子性的。

## 32位Java虚拟机中的long和double变量写操作为何不是原子的？

原子性这块，特例，32位虚拟机里的long/double类型的变量的简单赋值写操作，不是原子的，long i = 30，double c = 45.0，在32位虚拟机里就不是原子的，因为long和double是64位的

 ```
0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000
 ```

如果多个线程同时并发的执行long i = 30，long是64位的，就会导致有的线程在修改i的高32位，有的线程在修改i的低32位，多线程并发给long类型的变量进行赋值操作，在32位的虚拟机下，是有问题的

 就可能会导致多线程给long i = 30赋值之后，导致i的值不是30，可能是-3333344429，乱码一样的数字，就是因为高低32位赋值错了，就导致二进制数字转换为十进制之后是一个很奇怪的数字

## volatile保不保证原子性？

1、volatile对原子性保障的语义，在java里很有限的，几乎可以忽略不计。32位的java虚拟机里面，对long/double变量的赋值写是不原子的，此时如果对变量加上了volatile，就可以保证在32位java虚拟机里面，对long/double变量的赋值写是原子的了。（这是一个特列，可以通过volatile来保证原子性）但总体来说volatiel不保证原子性

例子：

`volatile long i;` 多个线程执行：`i = 30`，此时就不要紧了，因为volatile修饰了，就可以保证这个赋值操作是原子的了。

2、`int i = 0`，这种原子性的保证，不是靠volatile，java语言规范本身就规定了这种操作是原子性的。



**结论：volatile不保证原子性**













# wait和notify

## 为什么要出现咋同步代码块中

> 通过前面对monitor的C++源码讲解，答案应该很明显了。

1、如果一个线程在同步块中调用了`Object#wait`方法，会将该线程对应的ObjectWaiter从EntryList移除并加入到WaitSet中，然后释放锁。当wait的线程被notify之后，会将对应的ObjectWaiter从WaitSet移动到EntryList中。

2、注意，如果没有获取到监视器锁，wait 方法是会抛异常的，而且注意这个异常是IllegalMonitorStateException 异常。这是重要知识点，要考。

3、如果线程获得锁后调用`Object.wait`方法，则会将线程加入到WaitSet中。而WaitSet这个结构是在Monitor对象里，如果你没有获取到监视器锁，你就没有Monitor。那就无法加入到WaitSet里。







# 参考：



- 《Java并发编程的艺术》

- 《Java并发编程之美》

- b站一些机构的视频

- https://www.cnblogs.com/yanlong300/p/8986041.html

- https://github.com/farmerjohngit/myblog

- 公总号：儒猿技术窝

  



# 题外话：

1、《Java并发编程的艺术》这本书非常好。不过我第一遍看的时候，就属于云里雾里的，就是感觉抓不到重点。直到我秋招结束之后，再次回顾的时候才开始有点感觉了。这本书从硬件层面，从设计层面讲的一些内容讲的很好，给了我很大的帮助。这本书结合《Java并发编程之美》可能会有更好的阅读体验和理解（仅仅是个人觉得）。读者有时间还是尽量要看下这两本书，我的博客只是总结了面试经常会问到的一些内容，以及一些很难找到的一些资料。

2、相信大家也能明显的感觉到写这篇博客的时候，涉及到了很多**操作系统**的知识。其实越往后学，你就越能发现**操作系统**对你理解很多东西的原理会有很大帮助。比如说这篇博客的内存屏障，MESI缓存一致性协议，cpu指令；还有netty的一些通信原理，还有mysql,redis与操作系统的交互，rokcetmq的零拷贝，消息储存机制等等。希望读者有时间能够好好看一下**计算机网络和操作系统**，真的是很重要，并且校招和社招面试大厂问的都比较多。

3、因为东西写的比较多，略微有点乱，写了3万多字。所以可能目录顺序不是那么的好，敬请读者见谅。

4、感谢各个博主的博客，对我都有很大的帮助。笔者能力有限，总结的博客可能有错误。如果有错误请及时联系我，谢谢~。



