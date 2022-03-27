---
title: 'Java并发体系-第三阶段-JUC并发包-[1]'
tags:
  - Java并发
  - 原理
  - 源码
categories:
  - Java并发
  - 原理
keywords: Java并发，原理，源码
description: 万字系列长文讲解-Java并发体系-第三阶段-JUC并发包。JUC在高并发编程中使用频率非常高，这里会详细介绍其用法。
cover: 'https://npm.elemecdn.com/lql_static@latest/logo/Java_concurrency.png'
abbrlink: 5be45d9e
date: 2020-10-09 22:13:58
---



# AtomicXXXFieldUpdater

> 算是一个小补充

## 简介

```java
public class AtomicIntegerFieldUpdaterTest {

    public static void main(String[] args) {
        AtomicIntegerFieldUpdater<Test> updater = AtomicIntegerFieldUpdater.newUpdater(Test.class, "value");
        Test ts = new Test();

        IntStream.rangeClosed(0, 2).forEach(item -> {            
        	new Thread(() -> {
                int value = updater.getAndIncrement(ts);
                System.out.println("oldV: " + value);
            }).start();
        });
    }
 }

class Test {
    volatile int value;
}

```

1、以`AtomicIntegerFieldUpdater`为例，看上面代码。Test类的value属性被volatile修饰了，但是volatile只能保证可见性和有序性。在以往的文章里我们讲过是可以通过volatile+CAS同时解决可见性，有序性，原子性。

2、JUC提供了一种新的功能来保证原子性，AtomicXXXFieldUpdater修饰的类对应的字段，在进行更新时同样可以保证原子性。

## 使用场景

- 想让类的属性操作具备原子性，

- 但是不想使用锁。

- 大量需要原子类型修饰的对象，相比较比较耗费内存

举个例子：

如果你想要保证原子性，一般是使用`AtomicStampedReference<Node>`来包装Node对象。

```Java
import org.apache.lucene.util.RamUsageEstimator;

import java.util.concurrent.atomic.AtomicStampedReference;

/**
 * @Author: youthlql-吕
 * @Date: 2020/10/11 16:51
 * <p>
 * 功能描述: 计算对象内存大小
 * https://blog.csdn.net/yunqiinsight/article/details/80431831
 * https://www.cnblogs.com/libin6505/p/10648091.html
 */
public class Calculate_Java_Object_Size {

    public static void main(String[] args) {
        Node node = new Node();
        //计算指定对象本身在堆空间的大小，单位字节
        long b = RamUsageEstimator.shallowSizeOf(node);
        System.out.println(b);

        AtomicStampedReference<Node> nodeAtomicStampedReference = new AtomicStampedReference<Node>(node,0);
        long c = RamUsageEstimator.shallowSizeOf(nodeAtomicStampedReference);
        System.out.println(c + b);
    }

    /**
     * 功能描述: Api说明
     */
    public static void Api(Object o){
        //下面三个方法参数都是 Object类型

        //计算指定对象及其引用树上的所有对象的综合大小，单位字节
        long a = RamUsageEstimator.sizeOf(o);
        System.out.println(a);

        //计算指定对象本身在堆空间的大小，单位字节
        long b = RamUsageEstimator.shallowSizeOf(o);
        System.out.println(b);

        //计算指定对象及其引用树上的所有对象的综合大小，返回可读的结果，如：2KB
        String c = RamUsageEstimator.humanSizeOf(o);
        System.out.println(c);
    }
}

class Node {
    Node pre;
    Node next;
    Integer value;
}

```

> 输出结果：
>
> 24
>
> 40

可以看出如果用了`AtomicStampedReference<Node>`，会多出16个字节。如果对象有10000个，那么会多出很多字节。生产过程中的内存都是很贵的。为了减少内存消耗，同时可以保证原子性，就可以使用`AtomicXXXFieldUpdater`。



# CountDownLatch

## 简介

* countDownLatch这个类使一个线程等待其他线程各自执行完毕后再执行。
* 是通过一个 state（相当于计数器）的东西来实现的，计数器的初始值是 **线程的数量或者任务的数量**。
* 每当一个线程执行完毕后，计数器的值就-1，当计数器的值为0时，表示所有线程都执行完毕，然后在闭锁上等待的线程就可以恢复工作了。
* CountDownLatch的方便之处在于，你可以在一个线程中使用， **也可以在多个线程上使用，一切只依据状态值**，这样便不会受限于任何的场景。

## 使用场景一

**需求**

* 可能刚从数据库读取了一批数据
* 利用并发处理这批数据
* 当所有的数据处理完成后，再去执行后面的操作

**解决方案**

* **第一种**：可以利用 join 的方法，但是在线程池中，比较麻烦。
* **第二种**：利用线程池的awaitTermination，阻塞一段时间。
  - 当使用awaitTermination时，主线程会处于一种等待的状态，等待线程池中所有的线程都运行完毕后才继续运行。
* **第三种**：利用CountDownLatch，每当任务完成一个，就计数器减一。

```java
package _05_AQS;

import java.util.concurrent.CountDownLatch;

/**
 * @Author: youthlql-吕
 * @Date: 2019/9/26 10:05
 * <p>
 * 功能描述:
 */
public class Video32 {

    public static void main(String[] args) throws InterruptedException {
        CountDownLatch countDownLatch = new CountDownLatch(6);

        for (int i = 0; i < 6; i++) {
             new Thread(() ->{
                System.out.println("\t\t" + Thread.currentThread().getName() + "处理完毕~~~");
                countDownLatch.countDown();
                System.out.println("非调用者线程-" + Thread.currentThread().getName() + "-还可以干点其他事");
             }, Country.forEach_Country(i + 1).getCountryName()).start();
        }

        countDownLatch.await();
        System.out.println("-----------------------------");
        System.out.println("\t 所有任务都已经处理完毕，可以往后执行了！");

    }
}

enum Country{

    ONE(1,"1号任务"),
    TWO(2,"2号任务"),
    THREE(3,"3号任务"),
    FOUR(4,"4号任务"),
    FIVE(5,"5号任务"),
    SIX(6,"6号任务");

    private Integer index;
    private String countryName;


    public static Country forEach_Country(Integer index){
        Country[] values = Country.values();
        for (Country c: values) {
            if(c.getIndex() == index){
                return c;
            }
        }
        return null;
    }

    Country(Integer index, String countryName) {
        this.index = index;
        this.countryName = countryName;
    }

    public Integer getIndex() {
        return index;
    }

    public String getCountryName() {
        return countryName;
    }

}

```

**结果**

```java
		1号任务处理完毕~~~
		5号任务处理完毕~~~
非调用者线程-5号任务-还可以干点其他事
		6号任务处理完毕~~~
		3号任务处理完毕~~~
非调用者线程-3号任务-还可以干点其他事
		4号任务处理完毕~~~
		2号任务处理完毕~~~
非调用者线程-4号任务-还可以干点其他事
非调用者线程-6号任务-还可以干点其他事
非调用者线程-1号任务-还可以干点其他事
-----------------------------
	 所有任务都已经处理完毕，可以往后执行了！
非调用者线程-2号任务-还可以干点其他事
```

因为countdown只会阻塞调用者，其它线程干完任务就可以干其他事。这里的调用者线程就是main线程。

## 使用场景二

**需求**

* 多个线程协同工作
* 多个线程需要等待其他线程的工作之后，再进行其后续工作。
* 被唤醒后继续执行其他操作

```java
public class CountDownLatchExample {

    public static void main(String[] args) throws InterruptedException {
        final CountDownLatch latch = new CountDownLatch(1);
        new Thread(() -> {
            System.out.println(Thread.currentThread().getName() + " Do some initial working.");
            try {
                Thread.sleep(1000);
                latch.await();
                System.out.println(Thread.currentThread().getName() + " Do other working.");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }).start();

        new Thread(() -> {
            System.out.println(Thread.currentThread().getName() + " Do some initial working.");
            try {
                Thread.sleep(1000);
                latch.await();
                System.out.println(Thread.currentThread().getName() + " Do other working.");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }).start();

        new Thread(() -> {
            System.out.println("asyn prepare for some data.");
            try {
                Thread.sleep(2000);
                System.out.println("Data prepare for done.");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }finally {
                latch.countDown();
            }
        }).start();

    }

}
```

**结果**

```java
Thread-0 Do some initial working.
Thread-1 Do some initial working.
asyn prepare for some data.
Data prepare for done.
Thread-0 Do other working.
Thread-1 Do other working.
```

总体来说意思都差不多



## 常用API

**构造方法只有一个**

* `CountDownLatch(int count)` ：构造一个以给定计数

**实例方法**

* `public void await()`
  - **当前线程等到锁存器计数到零**
  - 可以被 **打断**
* `public boolean await(long timeout,TimeUnit unit)`
  - 等待一段时间
  - **timeout** - 等待的最长时间 ， **unit** - timeout参数的时间单位
  - 如果 **指定的等待时间过去**，则返回值false
  - 如果 **计数达到零**，则方法返回值为true
* `public void countDown()`
  - 减少锁存器的计数， **如果计数达到零，释放所有等待的线程**。
* `public long getCount()`
  - 返回当前计数

## 给离散的平行任务增加逻辑层次关系

**需求**

* 并发的从很多的数据库读取大量数据
* 在读取数据的过程中，某个表可能会出现： 数据丢失、数据精度丢失、数据大小不匹配。
* 需要进行对数据的各个情况进行检测，这个检测是并发的完成的
* 所以需要控制如果一个表所有的情况检测完成，再进行后续的操作

**解决**

* 利用 `CountDownLatch`的计数器
* 每当一个检测完成，计数器减一
* 如果计数为0，执行后面操作

```java
import java.util.ArrayList;
import java.util.List;
import java.util.Random;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

/**
 * @Author: youthlql-吕
 * @Date: 2020/10/11 21:05
 * <p>
 * 功能描述:
 */
public class test {
    private static final Random RANDOM = new Random();

    public static void main(String[] args) throws Exception {
        //2个事件请求，这里只演示校验数据行，和数据schema
        Event[] events = {new Event(1), new Event(2)};

        ExecutorService service = Executors.newFixedThreadPool(5);

        for (Event event : events) {
            //2个事件请求中，可能涉及多个表，可以再分成多个表
            List<Table> tables = capture(event);
            for (Table table : tables) {
                TaskBatch taskBatch = new TaskBatch(2);
                TrustSourceColumns sourceColumns = new TrustSourceColumns(table, taskBatch);
                TrustSourceRecordCount recordCount = new TrustSourceRecordCount(table, taskBatch);
                service.submit(sourceColumns);
                service.submit(recordCount);
            }
        }
    }

    static class Event {
        private int id;

        Event(int id) {
            this.id = id;
        }
    }

    interface Watcher {
        void done(Table table);
    }

    static class TaskBatch implements Watcher {

        private final CountDownLatch latch;

        TaskBatch(int size) {
            this.latch = new CountDownLatch(size);
        }

        @Override
        public void done(Table table) {
            latch.countDown();
            if (latch.getCount() == 0) {
                System.out.println("The table " + table.tableName + " finished work , " + table.toString());
            }
        }
    }

    static class Table {
        String tableName;
        long sourceRecordCount;
        long targetCount;
        String columnSchema = "columnXXXType = varchar";

        String targetColumnSchema = "";

        public Table(String tableName, long sourceRecordCount) {
            this.tableName = tableName;
            this.sourceRecordCount = sourceRecordCount;

        }

        @Override
        public String toString() {
            return "Table{" +
                    "tableName='" + tableName + '\'' +
                    ", sourceRecordCount=" + sourceRecordCount +
                    ", targetCount=" + targetCount +
                    ", columnSchema='" + columnSchema + '\'' +
                    ", targetColumnSchema='" + targetColumnSchema + '\'' +
                    '}';
        }
    }

    private static List<Table> capture(Event event) {
        List<Table> list = new ArrayList<>();
        for (int i = 0; i < 4; i++) {
            list.add(new Table("table-" + event.id + "-" + i, i * 1000));
        }
        return list;
    }

    //校验数据行数是否一致
    static class TrustSourceRecordCount implements Runnable {

        private final Table table;

        private final TaskBatch taskBatch;

        TrustSourceRecordCount(Table table, TaskBatch taskBatch) {
            this.table = table;
            this.taskBatch = taskBatch;
        }

        @Override
        public void run() {
            try {
                Thread.sleep(RANDOM.nextInt(100));
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            table.targetCount = table.sourceRecordCount;

            taskBatch.done(table);

        }

    }
    //校验数据列属性以及对应的表
    static class TrustSourceColumns implements Runnable {

        private final Table table;

        private final TaskBatch taskBatch;

        TrustSourceColumns(Table table, TaskBatch taskBatch) {
            this.table = table;
            this.taskBatch = taskBatch;
        }

        @Override
        public void run() {
            try {
                Thread.sleep(RANDOM.nextInt(100));
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            table.targetColumnSchema = table.columnSchema;

            taskBatch.done(table);
        }

    }
}

```

结果

```
The table table-1-1 finished work , Table{tableName='table-1-1', sourceRecordCount=1000, targetCount=1000, columnSchema='columnXXXType = varchar', targetColumnSchema='columnXXXType = varchar'}
The table table-1-0 finished work , Table{tableName='table-1-0', sourceRecordCount=0, targetCount=0, columnSchema='columnXXXType = varchar', targetColumnSchema='columnXXXType = varchar'}
The table table-2-0 finished work , Table{tableName='table-2-0', sourceRecordCount=0, targetCount=0, columnSchema='columnXXXType = varchar', targetColumnSchema='columnXXXType = varchar'}
The table table-1-2 finished work , Table{tableName='table-1-2', sourceRecordCount=2000, targetCount=2000, columnSchema='columnXXXType = varchar', targetColumnSchema='columnXXXType = varchar'}
The table table-1-3 finished work , Table{tableName='table-1-3', sourceRecordCount=3000, targetCount=3000, columnSchema='columnXXXType = varchar', targetColumnSchema='columnXXXType = varchar'}
The table table-2-1 finished work , Table{tableName='table-2-1', sourceRecordCount=1000, targetCount=1000, columnSchema='columnXXXType = varchar', targetColumnSchema='columnXXXType = varchar'}
The table table-2-2 finished work , Table{tableName='table-2-2', sourceRecordCount=2000, targetCount=2000, columnSchema='columnXXXType = varchar', targetColumnSchema='columnXXXType = varchar'}
The table table-2-3 finished work , Table{tableName='table-2-3', sourceRecordCount=3000, targetCount=3000, columnSchema='columnXXXType = varchar', targetColumnSchema='columnXXXType = varchar'}

```



## 利用CountDownLatch实现回调函数

**实现：**

* 在每个线程使计数器减一的时候，利用getCount判断，当前是否所有线程任务执行完成

```java
public class CyclicBarrierTest {
    static class MyCountDownLatch extends CountDownLatch{
        private final Runnable runnable;

        public MyCountDownLatch(int count,Runnable runnable) {
            super(count);
            this.runnable = runnable;
        }

        @Override
        public void countDown() {
            super.countDown();
            if (getCount()==0){
                this.runnable.run();
            }
        }

    }

    public static void main(String[] args) {

        final MyCountDownLatch latch = new MyCountDownLatch(2, ()->{
            System.out.println("All of work finish done. This is call back.");
        });

        new Thread(){
            @Override
            public void run() {
                try {
                    Thread.sleep(1000);
                    latch.countDown();
                    System.out.println(getName() +  " finished.");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }.start();

        new Thread(){
            @Override
            public void run() {
                try {
                    Thread.sleep(1000);
                    latch.countDown();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println(getName() +  " finished.");
            }
        }.start();

    }
}
```





# CyclicBarrier

## 引出

- 栅栏类似于闭锁（CountDownLatch），它能阻塞一组线程直到某个事件的发生。栅栏与闭锁的关键区别在于， **所有的线程必须同时到达栅栏位置，才能继续执行**。闭锁用于等待事件，而 **栅栏用于等待其他线程**。

- CyclicBarrier可以使一定数量的线程反复地在栅栏位置处汇集。 **当线程到达栅栏位置时将调用await方法，这个方法将阻塞直到所有线程都到达栅栏位置。** 如果所有线程都到达栅栏位置，那么栅栏将打开，此时所有的线程都将被释放，而栅栏将被重置以便下次使用。

```java
package _05_AQS;

import java.util.concurrent.BrokenBarrierException;
import java.util.concurrent.CyclicBarrier;

/**
 * @Author: youthlql-吕
 * @Date: 2019/9/26 10:34
 * <p>
 * 功能描述:
 */
public class Video33 {

    public static void main(String[] args) {
        CyclicBarrier cyclicBarrier = new CyclicBarrier(7,() -> System.out.println("收集到7颗龙珠,召唤神龙"));

        for (int i = 0; i < 7; i++) {
             final int temp = i + 1;
             new Thread(() ->{
                System.out.println(Thread.currentThread().getName() + "\t收集到第" + temp + "颗龙珠");
                 try {
                     int await = cyclicBarrier.await();
                     System.out.println("还剩几个:" + await);
                 } catch (InterruptedException e) {
                     e.printStackTrace();
                 } catch (BrokenBarrierException e) {
                     e.printStackTrace();
                 }
             },"线程" + String.valueOf(i)).start();
        }
    }
}

```

**结果：**

```
线程0	收集到第1颗龙珠
线程2	收集到第3颗龙珠
线程3	收集到第4颗龙珠
线程1	收集到第2颗龙珠
线程4	收集到第5颗龙珠
线程5	收集到第6颗龙珠
线程6	收集到第7颗龙珠
收集到7颗龙珠,召唤神龙
还剩几个:0
还剩几个:6
还剩几个:3
还剩几个:4
还剩几个:5
还剩几个:1
还剩几个:2
```

可以明显的看到cyclicBarrier会阻塞所有线程，和countdownlatch不一样。

## API使用

### 构造方法

```java
public CyclicBarrier(int parties)
public CyclicBarrier(int parties, Runnable barrierAction)
```

* `parties` 是参与线程的个数
* 第二个构造方法有一个 `Runnable` 参数，这个参数的意思是 **最后一个到达线程要做的任务*

### 重要方法

```java
public int await() throws InterruptedException, BrokenBarrierException
public int await(long timeout, TimeUnit unit) throws InterruptedException, BrokenBarrierException, TimeoutException
```

* 线程调用 await() **表示自己已经到达栅栏**
* BrokenBarrierException 表示栅栏已经被破坏，破坏的原因可能是其中一个线程 await() 时被中断或者超时

### 其他方法

```java
public void reset()
```

* 将屏障重置为初始状态。 如果任何一方正在等待屏障，他们将返回 **BrokenBarrierException** 。
* 这样就可以重复利用这个屏障

## CyclicBarrier 与 CountDownLatch 区别

* `CountDownLatch` 是一次性的。 `CyclicBarrier` 是可循环利用的
* `CountDownLatch` 参与的线程的职责是不一样的，有的在倒计时，有的在等待倒计时结束。 `CyclicBarrier` 参与的线程职责是一样的

> 牛客-京东2019春招Java开发笔试卷-T27

> https://blog.csdn.net/liangyihuai/article/details/83106584

从jdk作者设计的目的来看，javadoc是这么描述它们的：

- CountDownLatch: A synchronization aid that allows one or more threads to wait until a set of operations being performed in other threads completes. 
- CyclicBarrier : A synchronization aid that allows a set of threads to all wait for each other to reach a common barrier point.  
- 从javadoc的描述可以得出： CountDownLatch：一个或者多个线程，等待其他多个线程完成某件事情之后才能执行； CyclicBarrier：多个线程互相等待，直到到达同一个同步点，再继续一起执行。 
- 对于CountDownLatch来说，重点是“一个线程（多个线程）等待”，而其他的N个线程在完成“某件事情”之后，可以终止，也可以等待。 而对于CyclicBarrier，重点是多个线程，在任意一个线程没有完成，所有的线程都必须互相等待，然后继续一起执行。 CountDownLatch是计数器，线程完成一个记录一个，只不过计数不是递增而是递减，而CyclicBarrier更像是一个阀门，需要所有线程都到达，阀门才能打开，然后继续执行。 按照这个题目的描述等所有线程都到达了这一个阀门处，再一起执行，此题强调的是，一起继续执行，我认为 选B 比较合理！  



- 像上文中CountDownLatch的例子，main线程在等待，其余6个线程任务做完之后，main线程才苏醒干后面的事。因为countdown只会阻塞调用者，其它线程干完任务就可以干其他事。这里的调用者线程就是main线程。

- 像上文中CyclicBarrier的例子，是7个线程互相等待对方，7个任务都完成后，执行注册的回调任务。

# Exchanger

## 简介

* 用于两个工作线程之间交换数据的封装工具类
* 简单说就是一个线程在完成一定的事务后想与另一个线程交换数据，则 **第一个先拿出数据的线程会一直等待第二个线程，直到第二个线程拿着数据到来时才能彼此交换对应数据*

`Exchanger<v></v>` 泛型类型，其中 **V 表示可交换的数据类型**

## 简单的应用

```java
import java.util.concurrent.*;

/**
 * @Author: youthlql-吕
 * @Date: 2020/10/11 21:05
 * <p>
 * 功能描述:
 */
public class ExchangerTest {

    public static void main(String[] args) {
        final Exchanger<String> exchanger = new Exchanger<>();


        new Thread(()->{
            System.out.println(Thread.currentThread().getName() + " start . ");
            try {

                /**
                 * 如果这里睡200ms的话，应该是B线程先拿出数据，然后B线程等待A线程。因为是B先给的数据，
                 * 所以最后A线程会先拿到B给的数据，也就是先打印
                 */
                TimeUnit.MILLISECONDS.sleep(200);
                String exchange = exchanger.exchange("I am come from T-A");
                System.out.println(Thread.currentThread().getName() + " get value : " + exchange);
                System.out.println(Thread.currentThread().getName() + " end . ");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }

        },"A").start();


        new Thread(()->{
            System.out.println(Thread.currentThread().getName() + " start . ");
            try {
                String exchange = exchanger.exchange("I am come from T-B");
                System.out.println(Thread.currentThread().getName() + " get value : " + exchange);
                System.out.println(Thread.currentThread().getName() + " end . ");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }

        },"B").start();
    }
}
```

**结果：**

```java
A start . 
B start . 
A get value : I am come from T-B
B get value : I am come from T-A
B end . 
A end . 
```



## 方法介绍

```java
public V exchange(V x) throws InterruptedException
```

* 等待另一个线程到达此交换点，然后将给定对象传输给它，接收其对象作为回报。
* 可以被打断
* 如果已经有个线程正在等待了，则直接交换数据

## 数据的分析

```java
import java.util.concurrent.*;


/**
 * @Author: youthlql-吕
 * @Date: 2020/10/11 21:05
 * <p>
 * 功能描述:
 */
public class test {
    public static void main(String[] args) {


        final Exchanger<Object> exchanger = new Exchanger<>();

        new Thread(() -> {
            Object Aobj = new Object();
            System.out.println("A将会发送：" + Aobj);
            try {
                Object Robj = exchanger.exchange(Aobj);
                System.out.println("A接收的：" + Robj);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }

        }, "A").start();

        new Thread(() -> {
            Object Bobj = new Object();
            System.out.println("B将会发送：" + Bobj);
            try {
                Object Robj = exchanger.exchange(Bobj);
                System.out.println("B接收的：" + Robj);

            } catch (InterruptedException e) {
                e.printStackTrace();
            }

        }, "B").start();
    }
}

```

**结果**

```
A将会发送：java.lang.Object@7be2d776
B将会发送：java.lang.Object@5eca8d3a
B接收的：java.lang.Object@7be2d776
A接收的：java.lang.Object@5eca8d3a
```



**从这个例子可以看出一个很严重的问题**：发送的对象和接收的对象是同一个对象，可能会用严重的线程安全问题。

# ReentrantLock

## 简介

当多个线程需要访问某个公共资源的时候，我们知道需要通过加锁来保证资源的访问不会出问题。java提供了两种方式来加锁：

* 一种是关键字： `synchronized`，一种是concurrent包下的基于API实现的。
* synchronized是 JVM底层支持的，而concurrent包则是 `jdk`实现。



## 公平锁和非公平锁

* 当有线程竞争锁时，当前线程会首先尝试获得锁而不是在队列中进行排队等候，这对于那些已经在队列中排队的线程来说显得不公平，这也是非公平锁的由来

* 默认情况下为非公平锁。

  

* 锁的存储结构就两个东西:"双向链表" + "int类型状态"。ReenTrantLock的实现是一种自旋锁， 通过循环调用CAS操作来实现加锁。它的性能比较好也是因为避免了使线程进入内核态的阻塞状态。想尽办法避免线程进入内核的阻塞状态是我们去分析和理解锁设计的关键钥匙。





## 构造方法

```java
public ReentrantLock()
public ReentrantLock(boolean fair)
```

* `fair`：该参数为true时，会尽力维持公平

## 获得锁

```java
public void lock()
public void lockInterruptibly()  throws InterruptedException
public boolean tryLock()
public boolean tryLock(long timeout, TimeUnit unit) throws InterruptedException
```

**lock**

* 正常的获取锁，如果没有获得到锁，就会被阻塞

**lockInterruptibly**

* 获取锁，如果没有获得到锁，就会被阻塞
* 可以被打断

**tryLock**

* 如果获得到锁，返回true
* 如果没有获得到锁，返回false
* `timeout`：表示等待的时间
* `tryLock()`在获取的锁的时候，不会考虑此时是否有其他线程在等待，会破坏公平。
* 如果你希望遵守公平设置此锁，然后用 `tryLock(0, TimeUnit.SECONDS)` 这几乎是等效的（它也检测中断）。

## 释放锁

```java
public void unlock()
```

* 尝试释放此锁。
* 必须是锁的持有者才能释放锁

## 锁的调试

```java
protected Thread getOwner()
public final boolean hasQueuedThreads()
public final int getQueueLength()
protected Collection<Thread> getQueuedThreads()
```

**getOwner**

* 返回持有锁的线程

**hasQueuedThreads**

* 是否有线程在等待获取锁

**getQueueLength**

* 获取等待锁的线程数目

**getQueuedThreads**

* 返回正在等待的线程集合

## Lock和synchronized的区别

**底层实现**：

* Lock基于 `AQS`实现，通过state和一个CLH队列来维护锁的获取与释放
* synchronized需要通过 `monitor`，经历一个从用户态到内核态的转变过程，更加耗时

**其他区别**

| synchronized                | Lock                                                         |
| --------------------------- | ------------------------------------------------------------ |
| 是java内置关键字，在jvm层面 | 是个java类                                                   |
| 无法判断是否获取锁的状态    | 可以判断是否获取到锁                                         |
| 会自动释放锁                | 需在finally中手工释放锁（unlock()方法释放锁），否则容易造成线程死锁 |
| 线程会一直等待下去          | 如果尝试获取不到锁，线程可以不用一直等待就结束               |

**总结来说**
synchronized的锁可重入、不可中断、非公平。而Lock锁可重入、可判断、可公平（两者皆可）。



# Semaphore

## 简介

- Semaphore是一种在多线程环境下使用的设施，该设施负责协调各个线程，以**保证它们能够正确、合理的使用公共资源的设施**，也是操作系统中用于控制进程同步互斥的量。

- Semaphore是一种计数信号量，用于管理一组资源，内部是基于AQS的共享模式。**它相当于给线程规定一个量从而控制允许活动的线程数**。

- Semaphore用于限制可以访问某些资源（物理或逻辑的）的线程数目，他维护了一个许可证集合，有多少资源需要限制就维护多少许可证集合，假如这里有N个资源，那就对应于N个许可证，同一时刻也只能有N个线程访问。

- 一个线程获取许可证就调用acquire方法，用完了释放资源就调用release方法。



除了JDK定义的哪些锁，Semaphore也可以定义锁。Semaphore可以做的功能相当的多，比如秒杀限流

## 用Semaphore自定义Lock

```java
public class SemaphoreTest {

    public static void main(String[] args) {
        final SemaphoreLock lock = new SemaphoreLock();
        for (int i = 0; i < 2; i++) {
            new Thread(()->{
                try {
                    lock.lock();
                    System.out.println(Thread.currentThread().getName() + " get the lock.");
                    Thread.sleep(3000);
                }catch (Exception e){
                    e.printStackTrace();
                }finally {
                    System.out.println(Thread.currentThread().getName() + " release the lock.");
                    lock.unlock();
                }
            }).start();
        }

    }
        static class SemaphoreLock{
        private final Semaphore semaphore = new Semaphore(1);

        public void lock() throws InterruptedException {
            semaphore.acquire();
        }

        public void unlock()  {
            semaphore.release();
        }
    }
}
```


​    

**结果**

> Thread-0 get the lock.  
> Thread-0 release the lock.  
> Thread-1 get the lock.  
> Thread-1 release the lock.

**跟synchronized的区别**

*   可以看出最大的区别就是可以控制多个线程访问多份资源，而不是只用一个线程访问一份资源



## 用Semaphore做限流

Semaphore可以控制多个线程访问多份资源，而不是只用一个线程访问一份资源，所以在限流方面也有应用。

```java
package _05_AQS;

import java.util.concurrent.Semaphore;
import java.util.concurrent.TimeUnit;

/**
 * @Author: youthlql-吕
 * @Date: 2019/9/26 10:58
 * <p>
 * 功能描述:
 */
public class Video34 {

    public static void main(String[] args) {
        Semaphore semaphore = new Semaphore(3);

        for (int i = 0; i < 10; i++) {
             new Thread(() ->{
                 try {
                     semaphore.acquire();
                     System.out.println(Thread.currentThread().getName() + "\t 进入抢购秒杀页面，准备抢小米9");
                     //停3秒后离开
                     try {
                         TimeUnit.SECONDS.sleep(3);
                     } catch (InterruptedException e) {
                         e.printStackTrace();
                     }
                     System.out.println(Thread.currentThread().getName() + "\t 离开抢购秒杀页面，成功抢到小米9");
                 } catch (InterruptedException e) {
                     e.printStackTrace();
                 }finally {
                     semaphore.release();
                 }
             },"用户" + String.valueOf(i)).start();
        }
    }
}

```

**结果：**

```
用户0	 进入抢购秒杀页面，准备抢小米9
用户2	 进入抢购秒杀页面，准备抢小米9
用户1	 进入抢购秒杀页面，准备抢小米9
用户1	 离开抢购秒杀页面，成功抢到小米9
用户2	 离开抢购秒杀页面，成功抢到小米9
用户0	 离开抢购秒杀页面，成功抢到小米9
用户4	 进入抢购秒杀页面，准备抢小米9
用户3	 进入抢购秒杀页面，准备抢小米9
用户5	 进入抢购秒杀页面，准备抢小米9
用户3	 离开抢购秒杀页面，成功抢到小米9
用户4	 离开抢购秒杀页面，成功抢到小米9
用户6	 进入抢购秒杀页面，准备抢小米9
用户8	 进入抢购秒杀页面，准备抢小米9
用户5	 离开抢购秒杀页面，成功抢到小米9
用户7	 进入抢购秒杀页面，准备抢小米9
```

代码里限制了资源为3个，所以同一时间段只能有3个线程进来抢购小米9手机。



## 常用API

### 构造方法

```java
public Semaphore(int permits)
public Semaphore(int permits , boolean fair)
```


*   `permits`：同一时间可以访问资源的线程数目
*   `fair`：尽可能的保证公平

### 重要方法

```java
public void acquire() throws InterruptedException
public void release()
```


* `acquire()`：**获取一个许可证**，如果许可证用完了，则陷入阻塞。可以被打断。

* `release()`：**释放一个许可证**

  `acquire(int permits)` 
  `public void release(int permits)`

**acquire多个时，如果没有足够的许可证可用，那么当前线程将被禁用以进行线程调度**，并且处于休眠状态，直到发生两件事情之一：

*   一些其他线程调用此信号量的一个release方法，当前线程旁边将分配许可证，并且可用许可证的数量满足此请求;
*   要么一些其他线程interrupts当前线程。

**release多个时，会使许可证增多，最终可能超过初始值**

```java
public boolean tryAcquire(int permits)
public boolean tryAcquire(int permits,
                          long timeout,
                          TimeUnit unit)
                   throws InterruptedException
```


*   尝试去拿，**拿到返回true**

### 其他方法

```java
public int availablePermits()
```


* 返回此信号量中当前可用的许可数。

  `protected Collection<Thread> getQueuedThreads()`
  public final int getQueueLength()

* `getQueuedThreads`返回正在阻塞的线程集合

* `getQueueLength`返回阻塞获取的线程数

  `public void acquireUninterruptibly()`
  `public void acquireUninterruptibly(int permits)`

* 可以`不被打断`获取许可证

  `public int drainPermits()`

* 获取当前全部的许可证目标



# Condition

## 简介

- Condition主要是用于线程通信的，也就是和Object类的wait，notify有同样的功能。不过Condition的功能更加多样，Conditon可以绑定锁，实现选择性唤醒。
- Condition和Lock搭配使用; condition必须使用`lock.newCondition();`来创建condition。必须存放在Lock中。否则抛出异常。

```Java
package _07_LockAndCondition;

import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

/**
 * @Author: youthlql-吕
 * @Date: 2019/9/26 15:41
 * <p>
 * 功能描述: A线程打印3次,B线程打印6次,C线程9次，持续两轮
 */
public class ConditionDemo {

    public static void main(String[] args) {
        ShareResource shareResource = new ShareResource();
        new Thread(() ->{
            for (int i = 0; i < 2; i++) {
                shareResource.print3();
            }
        },"A").start();

        new Thread(() ->{
            for (int i = 0; i < 2; i++) {
                shareResource.print6();
            }
        },"B").start();

        new Thread(() ->{
            for (int i = 0; i < 2; i++) {
                shareResource.print9();
            }
        },"C").start();


    }
}

//以前是一个lock只有一个钥匙,现在是一个lock多个钥匙
class ShareResource{
    //标志位,可取值为1,2,3
    private volatile Integer flag = 1;
    private Lock lock = new ReentrantLock();
    private Condition c1 = lock.newCondition();
    private Condition c2 = lock.newCondition();
    private Condition c3 = lock.newCondition();

    public void print3(){
        lock.lock();
        try {
            while(flag != 1){
                c1.await();
            }
            for (int i = 0; i < 3 ; i++) {
                System.out.println(Thread.currentThread().getName() + "\t" + (i+1));
            }
            flag = 2;
            c2.signal();//唤醒2号线程
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
    }

    public void print6(){
        lock.lock();
        try {
            while(flag != 2){
                c2.await();
            }
            for (int i = 0; i < 6 ; i++) {
                System.out.println(Thread.currentThread().getName() + "\t" + (i+1));
            }
            flag = 3;
            c3.signal();//唤醒3号线程
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
    }


    public void print9(){
        lock.lock();
        try {
            while(flag != 3){
                c3.await();
            }
            for (int i = 0; i < 9 ; i++) {
                System.out.println(Thread.currentThread().getName() + "\t" + (i+1));
            }
            flag = 1;
            c1.signal();//唤醒1号线程
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
    }


}
```

**结果：**

```
A	1
A	2
A	3
B	1
B	2
B	3
B	4
B	5
B	6
C	1
C	2
C	3
C	4
C	5
C	6
C	7
C	8
C	9
A	1
A	2
A	3
B	1
B	2
B	3
B	4
B	5
B	6
C	1
C	2
C	3
C	4
C	5
C	6
C	7
C	8
C	9
```



## Condition版生产者消费者

```java
/**
 * @Author: youthlql-吕
 * @Date: 2019/9/26 15:03
 * <p>
 * 功能描述: 使用Lock和Condition实现
 */
public class Producer_04 {
    public static void main(String[] args) {
        Consumer4 consumer = new Consumer4();

        //生产者线程A
        new Thread(() ->{
            for (int i = 0; i < 10; i++) {
                try {
                    consumer.increment();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        },"生产者A").start();

        new Thread(() ->{
            for (int i = 0; i < 10; i++) {
                try {
                    consumer.decrement();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        },"消费者B").start();

        new Thread(() ->{
            for (int i = 0; i < 10; i++) {
                try {
                    consumer.increment();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        },"生产者C").start();

        new Thread(() ->{
            for (int i = 0; i < 10; i++) {
                try {
                    consumer.decrement();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        },"消费者D").start();
    }
}
class Consumer4{
    private volatile Integer num = 0;
    private Lock lock = new ReentrantLock();
    private Condition condition = lock.newCondition();

    public  void increment() throws InterruptedException {
        lock.lock();
        try {
            while(num != 0){
                condition.await();
            }
            num++;
            System.out.println(Thread.currentThread().getName() + "\t" + num);
            condition.signalAll();
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }

    }

    public  void decrement() throws InterruptedException {
        lock.lock();
        try {
            while(num == 0){
                condition.await();
            }
            num--;
            System.out.println(Thread.currentThread().getName() + "\t" + num);
            condition.signalAll();
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
    }

}
```





# ReentrantReadWriteLock读写锁

读写锁的目的是为了让读 — 读之间不加锁

| 冲突    | 策略   |
| ------- | ------ |
| 读 — 读 | 并行化 |
| 读 — 写 | 串行化 |
| 写 — 读 | 串行化 |
| 写 — 写 | 串行化 |

```java
package _05_AQS;

import java.util.*;

/**
 * @Author: youthlql-吕
 * @Date: 2019/9/26 10:58
 * <p>
 * 功能描述:
 */
public class Video34 {

    private static final ReentrantReadWriteLock lock = new ReentrantReadWriteLock(true);

    private static final Lock readLock = lock.readLock();
    private static final Lock writeLock = lock.writeLock();

    private static final List<String> data = new ArrayList<>();

    public static void main(String[] args) throws InterruptedException {
        new Thread(()->write(),"write-1").start();

        Thread.sleep(1000);
        new Thread(()->read(),"read-1").start();

        new Thread(()->read(),"read-2").start();

    }

    public static void write(){
        try{
            writeLock.lock();
            data.add("数据XXX");
            Thread.sleep(3000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            System.out.println("写锁释放");
            writeLock.unlock();
        }
    }

    public static void read(){
        try{
            readLock.lock();
            for (String datum : data) {
                System.out.println(Thread.currentThread().getName() + " 读入："+datum);
            }
            Thread.sleep(3000);
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            System.out.println("读锁释放");
            readLock.unlock();
        }
    }
}

```

**结果：**

```
写锁释放
read-1 读入：数据XXX
read-2 读入：数据XXX
读锁释放
读锁释放
```

可以看到读写-串行化，读读可以并发。

# StampedLock读写锁

## ReadWriteLock出现的问题

1、深入分析ReadWriteLock，会发现它有个潜在的问题：如果有线程正在读，写线程需要等待读线程释放锁后才能获取写锁，即读的过程中不允许写线程去抢锁，这是一种悲观的读锁，会出现写饥饿。

2、有100个线程访问某个资源，如果有99线程个需要读锁，1个线程需要写锁，此时，写的线程很难得到执行。



## StampedLock改进

3、StampedLock和ReadWriteLock相比，改进之处在于：`读的过程中也允许获取写锁后写入` 。这样一来，我们读的数据就可能不一致，所以，**需要一点额外的代码来判断读的过程中是否有写入，这种读锁是一种乐观锁**。

4、乐观锁的意思就是乐观地估计读的过程中大概率不会有写入，因此被称为乐观锁。反过来，悲观锁则是读的过程中拒绝有写入，也就是写入必须等待。显然乐观锁的并发效率更高，但一旦有小概率的写入导致读取的数据不一致，需要能检测出来，再读一遍就行。



## 用StampedLock去悲观的读

StampedLock可以完全实现ReadWriteLock的功能。

```java
public class StampedLockDemo1 {


    private static final StampedLock stampedLock = new StampedLock();

    private static Integer DATA = 0;

    public static void write() {
        long stamp = -1;
        try {
            stamp = stampedLock.writeLock();// 获取写锁
            DATA++;
            System.out.println("写-->" + DATA);
        } finally {
            stampedLock.unlockWrite(stamp); // 释放写锁
        }
    }

    public static void read() {
        long stamp = -1;

        try {
            stamp = stampedLock.readLock();// 获取悲观读锁
            System.out.println("读-->" + DATA);
        } finally {
            stampedLock.unlockRead(stamp); // 释放悲观读锁
        }
    }

    public static void main(String[] args) {
        ExecutorService executor = Executors.newFixedThreadPool(10);
        //写任务
        Runnable writeTask = () -> {
            for (; ; ) {
                write();
            }
        };
        //读任务
        Runnable readTask = () -> {
            for (; ; ) {
                read();
            }
        };

        //一个线程写，9个线程读
        executor.submit(readTask);
        executor.submit(readTask);
        executor.submit(readTask);
        executor.submit(readTask);
        executor.submit(readTask);
        executor.submit(readTask);
        executor.submit(readTask);
        executor.submit(readTask);
        executor.submit(readTask);
        executor.submit(writeTask);//写线程要写最后


    }
}
```

独线程一旦先执行，写线程很难再获得锁

> 输出结果：
>
> 读-->0
> 读-->0
> 读-->0
> 读-->0
> 读-->0
> 读-->0
> 读-->0
> 读-->0
> 读-->0
> 读-->0
> 读-->0
> 读-->0
> 写-->1

## 用StampedLock去乐观的读

只需要改一下read()

```java
public class StampedLockDemo2 {
    private static final StampedLock stampedLock = new StampedLock();

    private static Integer DATA = 0;

    public static void write() {
        long stamp = -1;
        try {
            stamp = stampedLock.writeLock();// 获取写锁
            DATA++;
            System.out.println("写-->" + DATA);
        } finally {
            stampedLock.unlockWrite(stamp); // 释放写锁
        }
    }

    public static void read() {
        long stamp = stampedLock.tryOptimisticRead(); // 获得一个乐观读锁

        //在这块可能会有写锁抢锁，修改数据，所以用validate检查乐观读锁后是否有其他写锁发生
        /**
         * 1、在这块可能会有写锁抢锁，修改数据，所以用validate检查乐观读锁后是否有其他写锁发生
         * 判断执行读操作期间,是否存在写操作,如果存在则validate返回false
         * 2、如果有写锁抢锁，修改了数据，那么就要获取悲观锁。因为写锁在修改数据的过程中，你不能直接
         * 去读，只能老老实实拿到读锁再去读，才不会发生线程安全问题
         */
        if (!stampedLock.validate(stamp)) {//检查乐观读锁后是否有其他写锁发生
            try {
                stamp = stampedLock.readLock();// 获取悲观读锁
                System.out.println("悲观读-->" + DATA);
                return;
            } finally {
                stampedLock.unlockRead(stamp); // 释放悲观读锁
            }
        }
        System.out.println("乐观读-->" + DATA);

    }

    public static void main(String[] args) {
        ExecutorService executor = Executors.newFixedThreadPool(10);
        //写任务
        Runnable writeTask = () -> {
            for (; ; ) {
                write();
            }
        };
        //读任务
        Runnable readTask = () -> {
            for (; ; ) {
                read();
            }
        };

        //一个线程写，9个线程读
        executor.submit(readTask);
        executor.submit(readTask);
        executor.submit(readTask);
        executor.submit(readTask);
        executor.submit(readTask);
        executor.submit(readTask);
        executor.submit(readTask);
        executor.submit(readTask);
        executor.submit(readTask);
        executor.submit(writeTask);


    }
}
```



> 悲观读-->704
> 写-->705
> 写-->706
> 写-->707
> 悲观读-->707
> 写-->708
> 悲观读-->708
> 写-->709
> 写-->710
> 悲观读-->710
> 乐观读-->710
> 乐观读-->690
> 乐观读-->690
> 乐





# ForkJoin框架

思想：分而治之。将一个大任务分割成若干小任务，最终汇总每个小任务的结果得到这个大任务的结果。

**举例说明**

1、我们举个例子：如果要计算一个超大数组的和，最简单的做法是用一个循环在一个线程内完成：

2、还有一种方法，可以把数组拆成两部分，分别计算，最后加起来就是最终结果，这样可以用两个线程并行执行：

3、如果拆成两部分还是很大，我们还可以继续拆，用4个线程并行执行：

这就是Fork/Join任务的原理：判断一个任务是否足够小。如果是，直接计算，否则，就分拆成几个小任务分别计算。这个过程可以反复“裂变”成一系列小任务。



**编码实现**

**整个任务流程如下所示**：

* 首先继承任务，覆写任务的执行方法
* 通过判断阈值，判断该线程是否可以执行
* 如果不能执行，则将任务继续递归分配，利用fork方法，并行执行
* 如果是有返回值的，才需要调用join方法，汇集数据。



**主要的两个类：**

* `RecursiveAction` 一个递归无结果的ForkJoinTask（没有返回值）
* `RecursiveTask` 一个递归有结果的ForkJoinTask（有返回值）

## RecursiveTask

```java
public class ForkJoinRecursiveTask  {
	/* 
	   1、分到哪种程度可以不用分了
	   2、也就是设置一个任务处理最大的阈值
	*/
    private final static int MAX_THRESHOLD = 3;


    public static void main(String[] args) {
        final ForkJoinPool joinPool = new ForkJoinPool();
        ForkJoinTask<Integer> future = joinPool.submit(new CalculatedRecursiveTask(0, 1000));
        try {
            Integer integer = future.get();
            System.out.println("执行结果：" + integer);
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (ExecutionException e) {
            e.printStackTrace();
        }
    }


    private static class CalculatedRecursiveTask extends RecursiveTask<Integer> {


        private final int start;//任务开始的上标
        private final int end;//任务开始的下标

        private CalculatedRecursiveTask(int start, int end) {
            this.start = start;
            this.end = end;
        }

        @Override
        protected Integer compute() {
            if (end - start <= MAX_THRESHOLD) {//如果自己能处理，就自己计算
                return IntStream.rangeClosed(start, end).sum();
            } else {//自己处理不了，拆分任务
                int middle = (end + start) / 2;
                CalculatedRecursiveTask leftTask = new CalculatedRecursiveTask(start, middle);
                CalculatedRecursiveTask rightTask = new CalculatedRecursiveTask(middle + 1, end);
				
                //分别去执行
                leftTask.fork();
                rightTask.fork();
				
                //把返回结果合起来
                return leftTask.join() + rightTask.join();
            }
        }
    }
}

```

这个是有返回值的



## RecursiveAction

这个是没有返回值的

```java
public class ForkJoinRecursiveAction {

    private final static int MAX_THRESHOLD = 3;//设置一个任务处理最大的阈值

    private final static AtomicInteger SUM = new AtomicInteger();


    public static void main(String[] args) throws InterruptedException {
        ForkJoinPool forkJoinPool = new ForkJoinPool();
		
        forkJoinPool.submit(new CalculateRecursiveAction(0,1000));
		
        //任务执行需要事件，这里可以等一下
        forkJoinPool.awaitTermination(10, TimeUnit.SECONDS);

        System.out.println("执行结果为：" + SUM);
    }

    private static class CalculateRecursiveAction extends RecursiveAction{

        private final int start;
        private final int end;

        private CalculateRecursiveAction(int start, int end) {
            this.start = start;
            this.end = end;
        }

        @Override
        protected void compute() {
            if ((end-start)<=MAX_THRESHOLD){
                SUM.addAndGet(IntStream.rangeClosed(start,end).sum());
            }else {
                int middle = (start+end)/2;
                CalculateRecursiveAction leftAction = new CalculateRecursiveAction(start,middle);
                CalculateRecursiveAction rightAction = new CalculateRecursiveAction(middle+1,end);
                leftAction.fork();
                rightAction.fork();
            }
        }
    }
}

```



## 原理

**整个流程需要三个类完成**：

1、**ForkJoinPool**

- 既然任务是被逐渐的细化的，那就需要把这些任务存在一个池子里面，这个池子就是ForkJoinPool。

- 它与其它的ExecutorService区别主要在于它使用**"工作窃取"**，那什么是工作窃取呢？

- 工作窃取：一个大任务会被划分成无数个小任务，这些任务被分配到不同的队列，这些队列有些干活干的块，有些干得慢。于是干得快的，一看自己没任务需要执行了，就去隔壁的队列里面拿去任务执行。

2、**ForkJoinTask**

ForkJoinTask就是ForkJoinPool里面的每一个任务。他主要有两个子类：`RecursiveAction`和`RecursiveTask`。然后通过fork()方法去分配任务执行任务，通过join()方法汇总任务结果。

<img src="https://npm.elemecdn.com/youthlql@1.0.8/Java_concurrency/Source_code/Third_stage/0001.png">

## 小总结

* Fork/Join是一种基于“分治”的算法：通过分解任务，并行执行，最后合并结果得到最终结果。
* ForkJoinPool线程池可以把一个大任务分拆成小任务并行执行，任务类必须继承自RecursiveTask或RecursiveAction。
* 使用Fork/Join模式可以进行并行计算以提高效率。



