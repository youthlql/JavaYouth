---
title: 'Java并发体系-第二阶段-锁与同步-[1]'
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
abbrlink: 230c5bb3
date: 2020-10-06 22:09:58
---




> - 本阶段文章讲的略微深入，一些基础性问题不会讲解，如有基础性问题不懂，可自行查看我前面的文章，或者自行学习。
> - 本篇文章比较适合校招和社招的面试，笔者在2020年面试的过程中，也确实被问到了下面的一些问题。

# 并发编程中的三个问题

> 由于这个东西，和这篇文章比较配。所以虽然在第一阶段写过了，这里再回顾一遍。

## 可见性



### 可见性概念

可见性（Visibility）：是指一个线程对共享变量进行修改，另一个线程立即得到修改后的新值。

### 可见性演示

```java
/* 笔记
 * 1.当没有加Volatile的时候,while循环会一直在里面循环转圈
 * 2.当加了之后Volatile,由于可见性,一旦num改了之后,就会通知其他线程
 * 3.还有注意不能用if,if不会重新拉回来再判断一次。(也叫做虚假唤醒)
 * 4.案例演示:一个线程对共享变量的修改,另一个线程不能立即得到新值
 * */
public class Video04_01 {

    public static void main(String[] args) {
        MyData myData = new MyData();

        new Thread(() ->{
            System.out.println(Thread.currentThread().getName() + "\t come in ");
            try {
                TimeUnit.SECONDS.sleep(3);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            //睡3秒之后再修改num,防止A线程先修改了num,那么到while循环的时候就会直接跳出去了
            myData.addTo60();
            System.out.println(Thread.currentThread().getName() + "\t come out");
        },"A").start();


        while(myData.num == 0){
            //只有当num不等于0的时候,才会跳出循环
        }
    }
}

class MyData{
    int num = 0;

    public void addTo60(){
        this.num = 60;
    }
}
```

由上面代码可以看出，并发编程时，会出现可见性问题，当一个线程对共享变量进行了修改，另外的线程并没有立即看到修改后的最新值。

## 原子性



### 原子性概念

原子性（Atomicity）：在一次或多次操作中，要么所有的操作都成功执行并且不会受其他因素干扰而中 断，要么所有的操作都不执行或全部执行失败。不会出现中间状态

### 原子性演示

案例演示:5个线程各执行1000次 i++;

```java
/**
 * @Author: 吕
 * @Date: 2019/9/23 15:50
 * <p>
 * 功能描述: volatile不保证原子性的代码验证
 */
public class Video05_01 {

    public static void main(String[] args) {
        MyData03 myData03 = new MyData03();

        for (int i = 0; i < 20; i++) {
             new Thread(() ->{
                 for (int j = 0; j < 1000; j++) {
                     myData03.increment();
                 }
             },"线程" + String.valueOf(i)).start();
        }

        //需要等待上面的20个线程计算完之后再查看计算结果
        while(Thread.activeCount() > 2){
            Thread.yield();
        }

        System.out.println("20个线程执行完之后num:\t" + myData03.num);
    }
}


class MyData03{
    static int num = 0;

    public void increment(){
        num++;
    }

}
```



1、控制台输出：（由于并发不安全，每次执行的结果都可能不一样）

> 20个线程执行完之后num:	19706

正常来说，如果保证原子性的话，20个线程执行完，结果应该是20000。控制台输出的值却不是这个，说明出现了原子性的问题。

2、使用javap反汇编class文件，对于num++可以得到下面的字节码指令：

```java
9: getstatic     #12                 // Field number:I   取值操作
12: iconst_1 
13: iadd 
14: putstatic     #12                 // Field number:I  赋值操作
```

由此可见num++是由多条语句组成，以上多条指令在一个线程的情况下是不会出问题的，但是在多线程情况下就可能会出现问题。

比如num刚开始值是7。A线程在执行13: iadd时得到num值是8，B线程又执行9: getstatic得到前一个值是7。马上A线程就把8赋值给了num变量。但是B线程已经拿到了之前的值7，B线程是在A线程真正赋值前拿到的num值。即使A线程最终把值真正的赋给了num变量，但是B线程已经走过了getstaitc取值的这一步，B线程会继续在7的基础上进行++操作，最终的结果依然是8。本来两个线程对7进行分别进行++操作，得到的值应该是9，因为并发问题，导致结果是8。

3、并发编程时，会出现原子性问题，当一个线程对共享变量操作到一半时，另外的线程也有可能来操作共 享变量，干扰了前一个线程的操作。



## 有序性

### 有序性概念

有序性（Ordering）：是指程序中代码的执行顺序，Java在编译时和运行时会对代码进行优化（重排序）来加快速度，会导致程序终的执行顺序不一定就是我们编写代码时的顺序

```java
 instance = new SingletonDemo() 是被分成以下 3 步完成
	 memory = allocate();     分配对象内存空间
	 instance(memory);        初始化对象
	 instance = memory;	   设置 instance 指向刚分配的内存地址，此时 instance != null
```

步骤2 和 步骤3 不存在数据依赖关系，重排与否的执行结果单线程中是一样的。这种指令重排是被 Java 允许的。当 3 在前时，instance 不为 null，但实际上初始化工作还没完成，会变成一个返回 null 的getInstance。这时候数据就出现了问题。



### 有序性演示

jcstress是java并发压测工具。https://wiki.openjdk.java.net/display/CodeTools/jcstress 修改pom文件，添加依赖：

```Java
<dependency>   
 <groupId>org.openjdk.jcstress</groupId>    
<artifactId>jcstress-core</artifactId>    
<version>${jcstress.version}</version> 
</dependency>
```



```java
import org.openjdk.jcstress.annotations.*;
import org.openjdk.jcstress.infra.results.I_Result;

 @JCStressTest
 // @Outcome: 如果输出结果是1或4，我们是接受的(ACCEPTABLE)，并打印ok
 @Outcome(id = {"1", "4"}, expect = Expect.ACCEPTABLE, desc = "ok")
 //如果输出结果是0，我们是接受的并且感兴趣的，并打印danger
 @Outcome(id = "0", expect = Expect.ACCEPTABLE_INTERESTING, desc = "danger")
 @State
public class Test03Ordering {

    int num = 0;
    boolean ready = false;
    // 线程1执行的代码
    @Actor //@Actor：表示会有多个线程来执行这个方法
    public void actor1(I_Result r) {
        if (ready) {
            r.r1 = num + num;
        } else {
            r.r1 = 1;
        }
    }

    // 线程2执行的代码
    // @Actor
    public void actor2(I_Result r) {
        num = 2;
        ready = true;
    }
}
```

1、实际上上面两个方法会有很多线程来执行，为了讲解方便，我们只提出线程1和线程2来讲解。

2、I_Result 是一个保存int类型数据的对象，有一个属性 r1 用来保存结果，在多线程情况下可能出现几种结果？

情况1：线 程1先执行actor1，这时ready = false，所以进入else分支结果为1。

情况2：线程2执行到actor2，执行了num = 2;和ready = true，线程1执行，这回进入 if 分支，结果为 4。

情况3：线程2先执行actor2，只执行num = 2；但没来得及执行 ready = true，线程1执行，还是进入 else分支，结果为1。 

情况4：0，发生了指令重排

```java
 // 线程2执行的代码
    // @Actor
    public void actor2(I_Result r) {
        num = 2;    //pos_1
        ready = true;//pos_2
    }
   
```

pos_1处代码和pos_2处代码没有什么数据依赖关系，或者说没有因果关系。Java可能对其进行指令重排，排成下面的顺序。

```java
 // 线程2执行的代码
    // @Actor
    public void actor2(I_Result r) {
    	ready = true;//pos_2
        num = 2;    //pos_1
    }
```

此时如果线程2先执行到`ready = true;`还没来得及执行 `num = 2;` 。线程1执行，直接进入if分支，此时num默认值为0。 得到的结果也就是0。



# 指令重排

计算机在执行程序时，为了提高性能，编译器和处理器常常会对指令做重排。

## 为什么指令重排序可以提高性能？

简单地说，每一个指令都会包含多个步骤，每个步骤可能使用不同的硬件。因此，**流水线技术**产生了，它的原理是指令1还没有执行完，就可以开始执行指令2，而不用等到指令1执行结束之后再执行指令2，这样就大大提高了效率。

但是，流水线技术最害怕**中断**，恢复中断的代价是比较大的，所以我们要想尽办法不让流水线中断。指令重排就是减少中断的一种技术。

我们分析一下下面这个代码的执行情况：

```java
a = b + c;
d = e - f ;
```

先加载b、c（**注意，即有可能先加载b，也有可能先加载c**），但是在执行add(b,c)的时候，需要等待b、c装载结束才能继续执行，也就是增加了停顿，那么后面的指令也会依次有停顿,这降低了计算机的执行效率。

为了减少这个停顿，我们可以先加载e和f,然后再去加载add(b,c),这样做对程序（串行）是没有影响的,但却减少了停顿。既然add(b,c)需要停顿，那还不如去做一些有意义的事情。

综上所述，**指令重排对于提高CPU处理性能十分必要。虽然由此带来了乱序的问题，但是这点牺牲是值得的。**

指令重排一般分为以下三种：

* **编译器优化重排**

  编译器在**不改变单线程程序语义**的前提下，可以重新安排语句的执行顺序。

* **指令并行重排**

  现代处理器采用了指令级并行技术来将多条指令重叠执行。如果**不存在数据依赖性**(即后一个执行的语句无需依赖前面执行的语句的结果)，处理器可以改变语句对应的机器指令的执行顺序。

* **内存系统重排**

  由于处理器使用缓存和读写缓存冲区，这使得加载(load)和存储(store)操作看上去可能是在乱序执行，因为三级缓存的存在，导致内存与缓存的数据同步存在时间差。

<img src="https://npm.elemecdn.com/youthlql@1.0.8/Java_concurrency/Source_code/Second_stage/0001.png">



**指令重排可以保证串行语义一致，但是没有义务保证多线程间的语义也一致**。所以在多线程下，指令重排序可能会导致一些问题。

## as-if-serial语义

as-if-serial语义的意思是：不管编译器和CPU如何重排序，必须保证在单线程情况下程序的结果是正确的。 以下数据有依赖关系，不能重排序。

写后读：

```
int a = 1; 
int b = a;
```

写后写：

```
int a = 1; 
int a = 2;
```

读后写：

```
int a = 1; 
int b = a; 
int a = 2;
```

编译器和处理器不会对存在数据依赖关系的操作做重排序，因为这种重排序会改变执行结果。但是，如果操作之间不存在数据依赖关系，这些操作就可能被编译器和处理器重排序。

```
int a = 1; 
int b = 2; 
int c = a + b;
```



# Java内存模型(JMM)

在介绍Java内存模型之前，先来看一下到底什么是计算机内存模型。

## 计算机结构

### 计算机结构简介

冯诺依曼，提出计算机由五大组成部分，输入设备，输出设备存储器，控制器，运算器。

<img src="https://npm.elemecdn.com/youthlql@1.0.8/Java_concurrency/Source_code/Second_stage/0002.png">

输入设备：鼠标，键盘等等

输出设备：显示器，打印机等等

存储器：内存条

运算器和控制器组成CPU



### CPU

中央处理器，是计算机的控制和运算的核心，我们的程序终都会变成指令让CPU去执行，处理程序中 的数据。

### 内存

我们的程序都是在内存中运行的，内存会保存程序运行时的数据，供CPU处理。

### 缓存

CPU的运算速度和内存的访问速度相差比较大。这就导致CPU每次操作内存都要耗费很多等待时间。内 存的读写速度成为了计算机运行的瓶颈。于是就有了在CPU和主内存之间增加缓存的设计。靠近CPU 的缓存称为L1，然后依次是 L2，L3和主内存，CPU缓存模型如图下图所示。

<img src="https://npm.elemecdn.com/youthlql@1.0.8/Java_concurrency/Source_code/Second_stage/0003.png">

CPU Cache分成了三个级别: L1， L2， L3。级别越小越接近CPU，速度也更快，同时也代表着容量越小。速度越快的价格越贵。

1、L1是接近CPU的，它容量小，例如32K，速度快，每个核上都有一个L1 Cache。

2、L2 Cache 更大一些，例如256K，速度要慢一些，一般情况下每个核上都有一个独立的L2 Cache。

3、L3 Cache是三级缓存中大的一级，例如12MB，同时也是缓存中慢的一级，在同一个CPU插槽 之间的核共享一个L3 Cache。

<img src="https://npm.elemecdn.com/youthlql@1.0.8/Java_concurrency/Source_code/Second_stage/0004.png">

上面的图中有一个Latency指标。比如Memory这个指标为59.4ns，表示CPU在操作内存的时候有59.4ns的延迟，一级缓存最快只有1.2ns。

**CPU处理数据的流程**

Cache的出现是为了解决CPU直接访问内存效率低下问题的。

1、程序在运行的过程中，CPU接收到指令 后，它会先向CPU中的一级缓存（L1 Cache）去寻找相关的数据，如果命中缓存，CPU进行计算时就可以直接对CPU Cache中的数据进行读取和写人，当运算结束之后，再将CPUCache中的新数据刷新 到主内存当中，CPU通过直接访问Cache的方式替代直接访问主存的方式极大地提高了CPU 的吞吐能 力。

2、但是由于一级缓存（L1 Cache）容量较小，所以不可能每次都命中。这时CPU会继续向下一级的二 级缓存（L2 Cache）寻找，同样的道理，当所需要的数据在二级缓存中也没有的话，会继续转向L3 Cache、内存(主存)和硬盘。



## Java内存模型

1、Java Memory Molde (Java内存模型/JMM)，千万不要和Java内存结构（JVM划分的那个堆，栈，方法区）混淆。关于“Java内存模型”的权威解释，参考 https://download.oracle.com/otn-pub/jcp/memory_model1.0-pfd-spec-oth-JSpec/memory_model-1_0-pfd-spec.pdf。

2、 Java内存模型，是Java虚拟机规范中所定义的一种内存模型，Java内存模型是标准化的，屏蔽掉了底层不同计算机的区别。 Java内存模型是一套规范，描述了Java程序中各种变量(线程共享变量)的访问规则，以及在JVM中将变量存储到内存和从内存中读取变量这样的底层细节，具体如下。

3、Java内存模型根据官方的解释，主要是在说两个关键字，一个是`volatile`，一个是`synchronized`。



**主内存**

主内存是所有线程都共享的，都能访问的。所有的共享变量都存储于主内存。

**工作内存**

每一个线程有自己的工作内存，工作内存只存储该线程对共享变量的副本。线程对变量的所有的操 作(读，取)都必须在工作内存中完成，而不能直接读写主内存中的变量，不同线程之间也不能直接 访问对方工作内存中的变量。

<img src="https://npm.elemecdn.com/youthlql@1.0.8/Java_concurrency/Source_code/Second_stage/0005.png">

Java的线程不能直接在主内存中操作共享变量。而是首先将主内存中的共享变量赋值到自己的工作内存中，再进行操作，操作完成之后，刷回主内存。

**Java内存模型的作用**

Java内存模型是一套在多线程读写共享数据时，对共享数据的可见性、有序性、和原子性的规则和保障。 synchronized,volatile

## CPU缓存，内存与Java内存模型的关系

- 通过对前面的CPU硬件内存架构、Java内存模型以及Java多线程的实现原理的了解，我们应该已经意识到，多线程的执行终都会映射到硬件处理器上进行执行。 但Java内存模型和硬件内存架构并不完全一致。
- 对于硬件内存来说只有寄存器、缓存内存、主内存的概念，并没有工作内存和主内存之分，也就是说Java内存模型对内存的划分对硬件内存并没有任何影响， 因为JMM只是一种抽象的概念，是一组规则，不管是工作内存的数据还是主内存的数据，对于计算机硬 件来说都会存储在计算机主内存中，当然也有可能存储到CPU缓存或者寄存器中，因此总体上来说， Java内存模型和计算机硬件内存架构是一个相互交叉的关系，是一种抽象概念划分与真实物理硬件的交叉。

JMM内存模型与CPU硬件内存架构的关系：

<img src="https://npm.elemecdn.com/youthlql@1.0.8/Java_concurrency/Source_code/Second_stage/0006.png">

工作内存：可能对应CPU寄存器，也可能对应CPU缓存，也可能对应内存。

- Java内存模型是一套规范，描述了Java程序中各种变量(线程共享变量)的访问规则，以及在JVM中将变量 存储到内存和从内存中读取变量这样的底层细节，Java内存模型是对共享数据的可见性、有序性、和原子性的规则和保障。



## 再谈可见性

<img src="https://npm.elemecdn.com/youthlql@1.0.8/Java_concurrency/Source_code/Second_stage/0007.png">

<img src="https://npm.elemecdn.com/youthlql@1.0.8/Java_concurrency/Source_code/Second_stage/0008.png">

1、图中所示是 个双核 CPU 系统架构 ，每个核有自己的控制器和运算器，其中控制器包含一组寄存器和操作控制器，运算器执行算术逻辅运算。每个核都有自己的1级缓存，在有些架构里面还有1个所有 CPU 共享的2级缓存。 那么 Java 内存模型里面的工作内存，就对应这里的 Ll 或者 L2 存或者 CPU 寄存器。

2、一个线程操作共享变量时，它首先从主内存复制共享变量到自己的工作内存，然后对工作内存里的变量进行处理，处理完后将变量值更新到主内存。 

3、那么假如线程A和线程B同时处理一个共享变量，会出现什么情况?我们使用图所示CPU架构，假设线程A和线程B使用不同CPU执行，并且当前两级Cache都为空，那么这时候由于Cache的存在，将会导致内存不可见问题，具体看下面的分析。

-  线程A首先获取共享变量X的值，由于两级Cache都没有命中，所以加载主内存中X的值，假如为0。然后把X=0的值缓存到两级缓存，线程A修改X的值为1，然后将其写入两级Cache，并且刷新到主内存。线程A操作完毕后，线程A所在的CPU的两级Cache 内和主内存里面的X的值都是1。
-  线程B获取X的值，首先一级缓存没有命中，然后看二级缓存，二级缓存命中了，所以返回X=1;到这里一切都是正常的，因为这时候主内存中也是X=1。然后线程B修改X的值为2，并将其存放到线程2所在的一级Cache和共享二级Cache中，最后更新主内存中X 的值为2;到这里一切都是好的。
-  线程A 这次又需要修改X的值，获取时一级缓存命中，并且X=1，到这里问题就出现了，明明线程B已经把X的值修改为了2，为何线程A获取的还是1呢?这就是共享变量的内存不可见问题，也就是线程B写入的值对线程A不可见。那么如何解决共享变量内存不可见问题?使用Java中的volatile和synchronized关键字就可以解决这个问题，下面会有讲解。

# 主内存与工作内存之间的交互

为了保证数据交互时数据的正确性，Java内存模型中定义了8种操作来完成这个交互过程，这8种操作本身都是原子性的。虚拟机实现时必须保证下面 提及的每一种操作都是原子的、不可再分的。

<img src="https://npm.elemecdn.com/youthlql@1.0.8/Java_concurrency/Source_code/Second_stage/0009.png">

> (1)lock:作用于主内存的变量，它把一个变量标识为一条线程独占的状态。
>
> (2)unlock:作用于主内存的变量，它把一个处于锁定状态的变量释放出来，释放后的变量才可以被其它线程锁定。
>
> (3)read:作用于主内存的变量，它把一个变量的值从主内存传输到线程的工作内存中，以便随后的load动作使用。
>
> (4)load:作用于工作内存的变量，它把read操作从主内存中得到的变量值放入工作内存的变量副本中。
>
> (5)use:作用于工作内存的变量，它把工作内存中一个变量的值传递给执行引擎，每当虚拟机遇到一个需要使用到变量的值的字节码指令时都会执行这个操作。
>
> (6)assign:作用于工作内存的变量，它把一个从执行引擎接收到的值赋给工作内存的变量，每当虚拟机遇到一个给变量赋值的字节码指令时执行这个操作。
>
> (7)store:作用于工作内存的变量，它把工作内存中一个变量的值传送到主内存中，以便随后的write使用。
>
> (8)write:作用于主内存的变量，它把store操作从工作内存中得到的变量的值放入主内存的变量中。

注意:

1. 如果对一个变量执行lock操作，将会清空工作内存中此变量的值
2. 对一个变量执行unlock操作之前，必须先把此变量同步到主内存中
3. lock和unlock操作只有加锁才会有。synchronized就是通过这样来保证可见性的。

如果没有synchronized，那就是下面这样的

<img src="https://npm.elemecdn.com/youthlql@1.0.8/Java_concurrency/Source_code/Second_stage/0010.png">



# happens-before

## 什么是happens-before?

一方面，程序员需要JMM提供一个强的内存模型来编写代码；另一方面，编译器和处理器希望JMM对它们的束缚越少越好，这样它们就可以最可能多的做优化来提高性能，希望的是一个弱的内存模型。

JMM考虑了这两种需求，并且找到了平衡点，对编译器和处理器来说，**只要不改变程序的执行结果（单线程程序和正确同步了的多线程程序），编译器和处理器怎么优化都行。**

而对于程序员，JMM提供了**happens-before规则**（JSR-133规范），满足了程序员的需求——**简单易懂，并且提供了足够强的内存可见性保证。**换言之，程序员只要遵循happens-before规则，那他写的程序就能保证在JMM中具有强的内存可见性。

JMM使用happens-before的概念来定制两个操作之间的执行顺序。这两个操作可以在一个线程以内，也可以是不同的线程之间。因此，JMM可以通过happens-before关系向程序员提供跨线程的内存可见性保证。

happens-before关系的定义如下：

1. 如果一个操作happens-before另一个操作，那么第一个操作的执行结果将对第二个操作可见，而且第一个操作的执行顺序排在第二个操作之前。
2. **两个操作之间存在happens-before关系，并不意味着Java平台的具体实现必须要按照happens-before关系指定的顺序来执行。如果重排序之后的执行结果，与按happens-before关系来执行的结果一致，那么JMM也允许这样的重排序。**

happens-before关系本质上和as-if-serial语义是一回事。

as-if-serial语义保证单线程内重排序后的执行结果和程序代码本身应有的结果是一致的，happens-before关系保证正确同步的多线程程序的执行结果不被重排序改变。

总之，**如果操作A happens-before操作B，那么操作A在内存上所做的操作对操作B都是可见的，不管它们在不在一个线程。**

## 天然的happens-before关系

在Java中，有以下天然的happens-before关系：

* 1、程序次序规则：一个线程内，按照代码顺序，书写在前面的操作先行发生于书写在后面的操作

* 2、锁定规则：一个unLock操作先行发生于后面对同一个锁的lock操作，比如说在代码里有先对一个lock.lock()，lock.unlock()，lock.lock()

* 3、volatile变量规则：对一个volatile变量的写操作先行发生于后面对这个volatile变量的读操作，volatile变量写，再是读，必须保证是先写，再读

* 4、传递规则：如果操作A先行发生于操作B，而操作B又先行发生于操作C，则可以得出操作A先行发生于操作C

* 5、线程启动规则：Thread对象的start()方法先行发生于此线程的每个一个动作，thread.start()，thread.interrupt()

* 6、线程中断规则：对线程interrupt()方法的调用先行发生于被中断线程的代码检测到中断事件的发生

* 7、线程终结规则：线程中所有的操作都先行发生于线程的终止检测，我们可以通过Thread.join()方法结束、Thread.isAlive()的返回值手段检测到线程已经终止执行

* 8、对象终结规则：一个对象的初始化完成先行发生于他的finalize()方法的开始


上面这8条原则的意思很显而易见，就是程序中的代码如果满足这个条件，就一定会按照这个规则来保证指令的顺序。

**举例1：**

```java
int a = 1; // A操作
int b = 2; // B操作
int sum = a + b;// C 操作
System.out.println(sum);
```

根据以上介绍的happens-before规则，假如只有一个线程，那么不难得出：

```
1> A happens-before B 
2> B happens-before C 
3> A happens-before C
```

注意，真正在执行指令的时候，其实JVM有可能对操作A & B进行重排序，因为无论先执行A还是B，他们都对对方是可见的，并且不影响执行结果。

如果这里发生了重排序，这在视觉上违背了happens-before原则，但是JMM是允许这样的重排序的。

所以，我们只关心happens-before规则，不用关心JVM到底是怎样执行的。只要确定操作A happens-before操作B就行了。

重排序有两类，JMM对这两类重排序有不同的策略：

* 会改变程序执行结果的重排序，比如 A -> C，JMM要求编译器和处理器都禁止这种重排序。
* 不会改变程序执行结果的重排序，比如 A -> B，JMM对编译器和处理器不做要求，允许这种重排序。



**举例2：**

```Java
//伪代码
volatile boolean flag = false;
    //线程1
    prepare();

    flag = false;

    //线程2
    while(!flag){
        sleep();
    }

    //基于准备好的资源进行操作
    execute();
```

这8条原则是避免说出现乱七八糟扰乱秩序的指令重排，要求是这几个重要的场景下，比如是按照顺序来，但是8条规则之外，可以随意重排指令。

比如这个例子，如果用volatile来修饰flag变量，一定可以让prepare()指令在flag = true之前先执行，这就禁止了指令重排。

因为volatile要求的是，volatile前面的代码一定不能指令重排到volatile变量操作后面，volatile后面的代码也不能指令重排到volatile前面。

# volatile

volatile不保证原子性，只保证可见性和禁止指令重排

## CPU术语介绍

<img src="https://npm.elemecdn.com/youthlql@1.0.8/Java_concurrency/Source_code/Second_stage/0011.png">





```java
private static volatile SingletonDemo instance = null;

    private SingletonDemo() {
        System.out.println(Thread.currentThread().getName() + "\t 执行单例构造函数");
    }

    public static SingletonDemo getInstance(){

        if(instance == null){
            synchronized (SingletonDemo.class){
                if(instance == null){
                    instance = new SingletonDemo(); //pos_1
                }
            }
        }
        return instance;
    }
```

**pos_1处的代码转换成汇编代码如下**

```shell
0x01a3de1d: movb $0×0,0×1104800(%esi);
0x01a3de24: lock addl $0×0,(%esp);
```

## volatile保证可见性原理

有volatile变量修饰的共享变量进行写操作的时候会多出第二行汇编代码，通过查IA-32架 构软件开发者手册可知，Lock前缀的指令在多核处理器下会引发了两件事情。 

1）将当前处理器缓存行的数据写回到系统内存。 

2）这个写回内存的操作会使在其他CPU里缓存了该内存地址的数据无效。 

​	为了提高处理速度，处理器不直接和主内存进行通信，而是先将系统内存的数据读到内部缓存（L1，L2或其他）后再进行操作，但操作完不知道何时会写到内存。如果对声明了volatile的 变量进行写操作，JVM就会向处理器发送一条Lock前缀的指令，将这个变量所在缓存行的数据写回到系统内存。但是，就算写回到内存，如果其他处理器缓存的值还是旧的，再执行计算操作就会有问题。所以，在多处理器下，为了保证各个处理器的缓存是一致的，就会实现MESI缓存一致性协议，每个处理器通过嗅探在总线上传播的数据来检查自己缓存的值是不是过期了，当处理器发现自己缓存行对应的内存地址被修改，就会将当前处理器的缓存行设置成无效状态，当处理器对这个数据进行修改操作的时候，会重新从系统内存中把数据读到处理器缓存里。 

> 注意：lock前缀指令是同时保证可见性和有序性（也就是禁止指令重排）的

> 注意：lock前缀指令相当于一个内存屏障【后文讲】









## volatile禁止指令重排的原理

```java
public class VolatileExample {
    int a = 0;
    volatile boolean flag = false;

    public void writer() {
        a = 1; // step 1
        flag = true; // step 2
    }

    public void reader() {
        if (flag) { // step 3
            System.out.println(a); // step 4
        }
    }
}
```

在JSR-133之前的旧的Java内存模型中，是允许volatile变量与普通变量重排序的。那上面的案例中，可能就会被重排序成下列时序来执行：

1. 线程A写volatile变量，step 2，设置flag为true；
2. 线程B读同一个volatile，step 3，读取到flag为true；
3. 线程B读普通变量，step 4，读取到 a = 0；
4. 线程A修改普通变量，step 1，设置 a = 1；

可见，如果volatile变量与普通变量发生了重排序，虽然volatile变量能保证内存可见性，也可能导致普通变量读取错误。

所以在旧的内存模型中，volatile的写-读就不能与锁的释放-获取具有相同的内存语义了。为了提供一种比锁更轻量级的**线程间的通信机制**，**JSR-133**专家组决定增强volatile的内存语义：严格限制编译器和处理器对volatile变量与普通变量的重排序。

编译器还好说，JVM是怎么还能限制处理器的重排序的呢？它是通过**内存屏障**来实现的。

什么是内存屏障？硬件层面，内存屏障分两种：读屏障（Load Barrier）和写屏障（Store Barrier）。内存屏障有两个作用：

1. 阻止屏障两侧的指令重排序；
2. 强制把写缓冲区/高速缓存中的脏数据等写回主内存，或者让缓存中相应的数据失效。

> 注意这里的缓存主要指的是上文说的CPU缓存，如L1，L2等

### 保守策略下



- 在每个volatile写操作的前面插入一个StoreStore屏障。 

- 在每个volatile写操作的后面插入一个StoreLoad屏障。 

- 在每个volatile读操作的前面插入一个LoadLoad屏障。 

- 在每个volatile读操作的后面插入一个LoadStore屏障。

编译器在**生成字节码时**，会在指令序列中插入内存屏障来禁止特定类型的处理器重排序。编译器选择了一个**比较保守的JMM内存屏障插入策略**，但它可以保证在任意处理器平台，任意的程序中都能 得到正确的volatile内存语义。 

> 再逐个解释一下这几个屏障。注：下述Load代表读操作，Store代表写操作
>
> **LoadLoad屏障**：对于这样的语句Load1; LoadLoad; Load2，在Load2及后续读取操作要读取的数据被访问前，保证Load1要读取的数据被读取完毕。
> **StoreStore屏障**：对于这样的语句Store1; StoreStore; Store2，在Store2及后续写入操作执行前，这个屏障会吧Store1强制刷新到内存，保证Store1的写入操作对其它处理器可见。
> **LoadStore屏障**：对于这样的语句Load1; LoadStore; Store2，在Store2及后续写入操作被刷出前，保证Load1要读取的数据被读取完毕。
> **StoreLoad屏障**：对于这样的语句Store1; StoreLoad; Load2，在Load2及后续所有读取操作执行前，保证Store1的写入对所有处理器可见。它的开销是四种屏障中最大的（冲刷写缓冲器，清空无效化队列）。在大多数处理器的实现中，这个屏障是个万能屏障，兼具其它三种内存屏障的功能

对于连续多个volatile变量读或者连续多个volatile变量写，编译器做了一定的优化来提高性能，比如：

> 第一个volatile读;
>
> LoadLoad屏障；
>
> 第二个volatile读；
>
> LoadStore屏障

**1、下面是保守策略下，volatile写插入内存屏障后生成的指令序列示意图**

<img src="https://npm.elemecdn.com/youthlql@1.0.8/Java_concurrency/Source_code/Second_stage/0012.png">

> 图中的StoreStore屏障可以保证在volatile写之前，其前面的所有普通写操作已经对任 意处理器可见了。这是因为StoreStore屏障将保障上面所有的普通写在volatile写之前刷新到主内存。这里比较有意思的是，volatile写后面的StoreLoad屏障。此屏障的作用是避免volatile写与 后面可能有的volatile读/写操作重排序。因为编译器常常无法准确判断在一个volatile写的后面 是否需要插入一个StoreLoad屏障（比如，一个volatile写之后方法立即return）。为了保证能正确 实现volatile的内存语义，JMM在采取了保守策略：在每个volatile写的后面，或者在每个volatile 读的前面插入一个StoreLoad屏障。从整体执行效率的角度考虑，JMM最终选择了在每个volatile写的后面插入一个StoreLoad屏障。因为volatile写-读内存语义的常见使用模式是：一个 写线程写volatile变量，多个读线程读同一个volatile变量。当读线程的数量大大超过写线程时， 选择在volatile写之后插入StoreLoad屏障将带来可观的执行效率的提升。从这里可以看到JMM在实现上的一个特点：首先确保正确性，然后再去追求执行效率



**2、下面是在保守策略下，volatile读插入内存屏障后生成的指令序列示意图**

<img src="https://npm.elemecdn.com/youthlql@1.0.8/Java_concurrency/Source_code/Second_stage/0013.png">

> 图中的LoadLoad屏障用来禁止处理器把上面的volatile读与下面的普通读重排序。 LoadStore屏障用来禁止处理器把上面的volatile读与下面的普通写重排序。 上述volatile写和volatile读的内存屏障插入策略非常保守。在实际执行时，只要不改变volatile写-读的内存语义，编译器可以根据具体情况省略不必要的屏障。



**优化举例：**

```java
class VolatileBarrierExample {
        int a;
        volatile int v1 = 1;
        volatile int v2 = 2;

        void readAndWrite() {
            int i = v1; // 第一个volatile读
            int j = v2; // 第二个volatile读
            a = i + j; // 普通写
            v1 = i + 1; // 第一个volatile写
            v2 = j * 2; // 第二个 volatile写
        }
        // 其他方法 }
    }
```

针对readAndWrite()方法，编译器在生成字节码时可以做如下的优化

<img src="https://npm.elemecdn.com/youthlql@1.0.8/Java_concurrency/Source_code/Second_stage/0014.png">

​	注意，最后的StoreLoad屏障不能省略。因为第二个volatile写之后，方法立即return。此时编译器可能无法准确断定后面是否会有volatile读或写，为了安全起见，编译器通常会在这里插入一个StoreLoad屏障。 

​	上面的优化针对任意处理器平台，由于不同的处理器有不同“松紧度”的处理器内存模型，内存屏障的插入还可以根据具体的处理器内存模型继续优化。以X86处理器为例，图中除最后的StoreLoad屏障外，其他的屏障都会被省略。



### X86处理器优化

前面保守策略下的volatile读和写，在X86处理器平台可以优化成如下图所示。 

X86处理器仅会对写-读操作做重排序。X86不会对读-读、读-写和写-写操作 做重排序，因此在X86处理器中会省略掉这3种操作类型对应的内存屏障。在X86中，JMM仅需在volatile写后面插入一个StoreLoad屏障即可正确实现volatile写-读的内存语义。这意味着在X86处理器中，volatile写的开销比volatile读的开销会大很多（因为执行StoreLoad屏障开销会比较大）。 

<img src="https://npm.elemecdn.com/youthlql@1.0.8/Java_concurrency/Source_code/Second_stage/0015.png">



## volatile的用途

> 下面的代码在前面可能已经写过了，这里总结一下

从volatile的内存语义上来看，volatile可以保证内存可见性且禁止重排序。

在保证内存可见性这一点上，volatile有着与锁相同的内存语义，所以可以作为一个“轻量级”的锁来使用。但由于volatile仅仅保证对单个volatile变量的读/写具有原子性，而锁可以保证整个**临界区代码**的执行具有原子性。所以**在功能上，锁比volatile更强大；在性能上，volatile更有优势**。

在禁止重排序这一点上，volatile也是非常有用的。比如我们熟悉的单例模式，其中有一种实现方式是“双重锁检查”，比如这样的代码：

```java
public class Singleton {

    private static Singleton instance; // 不使用volatile关键字

    // 双重锁检验
    public static Singleton getInstance() {
        if (instance == null) { // 第7行
            synchronized (Singleton.class) {
                if (instance == null) {
                    instance = new Singleton(); // 第10行
                }
            }
        }
        return instance;
    }
}
```

如果这里的变量声明不使用volatile关键字，是可能会发生错误的。它可能会被重排序：

```java
instance = new Singleton(); // 第10行

// 可以分解为以下三个步骤
1 memory=allocate();// 分配内存 相当于c的malloc
2 ctorInstanc(memory) //初始化对象
3 s=memory //设置s指向刚分配的地址

// 上述三个步骤可能会被重排序为 1-3-2，也就是：
1 memory=allocate();// 分配内存 相当于c的malloc
3 s=memory //设置s指向刚分配的地址
2 ctorInstanc(memory) //初始化对象
```

而一旦假设发生了这样的重排序，比如线程A在第10行执行了步骤1和步骤3，但是步骤2还没有执行完。这个时候另一个线程B执行到了第7行，它会判定instance不为空，然后直接返回了一个未初始化完成的instance！

所以JSR-133对volatile做了增强后，volatile的禁止重排序功能还是非常有用的。