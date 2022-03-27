---
title: JVM系列-第5章-堆
tags:
  - JVM
  - 虚拟机
categories:
  - JVM
  - 1.内存与垃圾回收篇
keywords: JVM，虚拟机。
description: JVM系列-第5章-堆。
cover: 'https://npm.elemecdn.com/lql_static@latest/logo/jvm.png'
abbrlink: 50ac3a1c
date: 2020-11-11 20:38:42
---



# 堆

堆的核心概述
--------

### 堆与进程



1.  堆针对一个JVM进程来说是唯一的。也就是**一个进程只有一个JVM实例**，一个JVM实例中就有一个运行时数据区，一个运行时数据区只有一个堆和一个方法区。
2.  但是**进程包含多个线程，他们是共享同一堆空间的**。

<img src="https://npm.elemecdn.com/youthlql@1.0.8/JVM/chapter_005/0001.png">



1.  一个JVM实例只存在一个堆内存，堆也是Java内存管理的核心区域。
2.  Java堆区在JVM启动的时候即被创建，其空间大小也就确定了，堆是JVM管理的最大一块内存空间，并且堆内存的大小是可以调节的。
3.  《Java虚拟机规范》规定，堆可以处于物理上不连续的内存空间中，但在逻辑上它应该被视为连续的。
4.  所有的线程共享Java堆，在这里还可以划分线程私有的缓冲区（Thread Local Allocation Buffer，**TLAB**）。
5.  《Java虚拟机规范》中对Java堆的描述是：**所有的对象实例以及数组都应当在运行时分配在堆上**。（The heap is the run-time data area from which memory for all class instances and arrays is allocated）
    - 从实际使用角度看：“几乎”所有的对象实例都在堆分配内存，但并非全部。因为还有一些对象是在栈上分配的（逃逸分析，标量替换）
6.  数组和对象可能永远不会存储在栈上（**不一定**），因为栈帧中保存引用，这个引用指向对象或者数组在堆中的位置。
7.  在方法结束后，堆中的对象不会马上被移除，仅仅在垃圾收集的时候才会被移除。

    *   也就是触发了GC的时候，才会进行回收
    *   如果堆中对象马上被回收，那么用户线程就会收到影响，因为有stop the word
8.  堆，是GC（Garbage Collection，垃圾收集器）执行垃圾回收的重点区域。

> 随着JVM的迭代升级，原来一些绝对的事情，在后续版本中也开始有了特例，变的不再那么绝对。

```java
public class SimpleHeap {
    private int id;//属性、成员变量

    public SimpleHeap(int id) {
        this.id = id;
    }

    public void show() {
        System.out.println("My ID is " + id);
    }
    public static void main(String[] args) {
        SimpleHeap sl = new SimpleHeap(1);
        SimpleHeap s2 = new SimpleHeap(2);

        int[] arr = new int[10];

        Object[] arr1 = new Object[10];
    }
}
```

<img src="https://npm.elemecdn.com/youthlql@1.0.8/JVM/chapter_005/0002.png">



### 堆内存细分

现代垃圾收集器大部分都基于分代收集理论设计，堆空间细分为：

1.  Java7 及之前堆内存逻辑上分为三部分：新生区+养老区+永久区
    *   Young Generation Space    新生区      Young/New
        *   又被划分为Eden区和Survivor区
    *   Old generation space    养老区           Old/Tenure
    *   Permanent Space   永久区                   Perm
2.  Java 8及之后堆内存逻辑上分为三部分：新生区+养老区+元空间
    *   Young Generation Space 新生区，又被划分为Eden区和Survivor区
    *   Old generation space 养老区
    *   Meta Space 元空间 Meta

约定：新生区 <–> 新生代 <–> 年轻代 、 养老区 <–> 老年区 <–> 老年代、 永久区 <–\> 永久代

<img src="https://npm.elemecdn.com/youthlql@1.0.8/JVM/chapter_005/0003.png">



2.  堆空间内部结构，JDK1.8之前从永久代 替换成 元空间

<img src="https://npm.elemecdn.com/youthlql@1.0.8/JVM/chapter_005/0004.png">



## JVisualVM可视化查看堆内存

运行下面代码

```java
public class HeapDemo {
    public static void main(String[] args) {
        System.out.println("start...");
        try {
            TimeUnit.MINUTES.sleep(30);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        System.out.println("end...");
    }

}
```



1、双击jdk目录下的这个文件

<img src="https://npm.elemecdn.com/youthlql@1.0.8/JVM/chapter_005/0005.png">



2、工具 -> 插件 -> 安装Visual GC插件

<img src="https://npm.elemecdn.com/youthlql@1.0.8/JVM/chapter_005/0006.png">

3、运行上面的代码

<img src="https://npm.elemecdn.com/youthlql@1.0.8/JVM/chapter_005/0007.png">



设置堆内存大小与 OOM
--------------

### 设置堆内存



1.  Java堆区用于存储Java对象实例，那么堆的大小在JVM启动时就已经设定好了，大家可以通过选项"-Xms"和"-Xmx"来进行设置。
  
    *   **-Xms**用于表示堆区的起始内存，等价于**-XX:InitialHeapSize**
    *   **-Xmx**则用于表示堆区的最大内存，等价于**-XX:MaxHeapSize**
2.  一旦堆区中的内存大小超过“-Xmx"所指定的最大内存时，将会抛出OutofMemoryError异常。
3.  通常会将-Xms和-Xmx两个参数配置相同的值
  - 原因：假设两个不一样，初始内存小，最大内存大。在运行期间如果堆内存不够用了，会一直扩容直到最大内存。如果内存够用且多了，也会不断的缩容释放。频繁的扩容和释放造成不必要的压力，避免在GC之后调整堆内存给服务器带来压力。
  - 如果两个设置一样的就少了频繁扩容和缩容的步骤。内存不够了就直接报OOM
4.  默认情况下:

    *   初始内存大小：物理电脑内存大小/64
    *   最大内存大小：物理电脑内存大小/4



```java
/**
 * 1. 设置堆空间大小的参数
 * -Xms 用来设置堆空间（年轻代+老年代）的初始内存大小
 *      -X 是jvm的运行参数
 *      ms 是memory start
 * -Xmx 用来设置堆空间（年轻代+老年代）的最大内存大小
 *
 * 2. 默认堆空间的大小
 *    初始内存大小：物理电脑内存大小 / 64
 *             最大内存大小：物理电脑内存大小 / 4
 * 3. 手动设置：-Xms600m -Xmx600m
 *     开发中建议将初始堆内存和最大的堆内存设置成相同的值。
 *
 * 4. 查看设置的参数：方式一： jps   /  jstat -gc 进程id
 *                  方式二：-XX:+PrintGCDetails
 */
public class HeapSpaceInitial {
    public static void main(String[] args) {

        //返回Java虚拟机中的堆内存总量
        long initialMemory = Runtime.getRuntime().totalMemory() / 1024 / 1024;
        //返回Java虚拟机试图使用的最大堆内存量
        long maxMemory = Runtime.getRuntime().maxMemory() / 1024 / 1024;

        System.out.println("-Xms : " + initialMemory + "M");
        System.out.println("-Xmx : " + maxMemory + "M");

        System.out.println("系统内存大小为：" + initialMemory * 64.0 / 1024 + "G");
        System.out.println("系统内存大小为：" + maxMemory * 4.0 / 1024 + "G");

        try {
            Thread.sleep(1000000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

输出结果：

```java
-Xms : 123M
-Xmx : 1794M
系统内存大小为：7.6875G
系统内存大小为：7.0078125G
```

1、笔者电脑内存大小是8G，不足8G的原因是操作系统自身还占据了一些。

2、两个不一样的原因待会再说



设置下参数再看

<img src="https://npm.elemecdn.com/youthlql@1.0.8/JVM/chapter_005/0008.png">

```java
public class HeapSpaceInitial {
    public static void main(String[] args) {

        //返回Java虚拟机中的堆内存总量
        long initialMemory = Runtime.getRuntime().totalMemory() / 1024 / 1024;
        //返回Java虚拟机试图使用的最大堆内存量
        long maxMemory = Runtime.getRuntime().maxMemory() / 1024 / 1024;

        System.out.println("-Xms : " + initialMemory + "M");
        System.out.println("-Xmx : " + maxMemory + "M");


        try {
            Thread.sleep(1000000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

输出结果：

```java
-Xms : 575M
-Xmx : 575M
```

为什么会少25M

**方式一： jps   /  jstat -gc 进程id**

<img src="https://npm.elemecdn.com/youthlql@1.0.8/JVM/chapter_005/0009.png">

> jps：查看java进程
>
> jstat：查看某进程内存使用情况

```java
SOC: S0区总共容量
S1C: S1区总共容量
S0U: S0区使用的量
S1U: S1区使用的量
EC: 伊甸园区总共容量
EU: 伊甸园区使用的量
OC: 老年代总共容量
OU: 老年代使用的量
```

1、

25600+25600+153600+409600 = 614400K

614400 /1024 = 600M

2、

25600+153600+409600 = 588800K

588800 /1024 = 575M

3、

并非巧合，S0区和S1区两个只有一个能使用，另一个用不了（后面会详解）

 **方式二：-XX:+PrintGCDetails**

<img src="https://npm.elemecdn.com/youthlql@1.0.8/JVM/chapter_005/0010.png">



### OOM

```java
public class OOMTest {
    public static void main(String[] args) {
        ArrayList<Picture> list = new ArrayList<>();
        while(true){
            try {
                Thread.sleep(20);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            list.add(new Picture(new Random().nextInt(1024 * 1024)));
        }
    }
}

class Picture{
    private byte[] pixels;

    public Picture(int length) {
        this.pixels = new byte[length];
    }
}
```

1、设置虚拟机参数

`-Xms600m -Xmx600m`

最终输出结果：

```java
Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
	at com.atguigu.java.Picture.<init>(OOMTest.java:29)
	at com.atguigu.java.OOMTest.main(OOMTest.java:20)

Process finished with exit code 1
```



2、堆内存变化图

<img src="https://npm.elemecdn.com/youthlql@1.0.8/JVM/chapter_005/0011.png">

3、原因：大对象导致堆内存溢出

<img src="https://npm.elemecdn.com/youthlql@1.0.8/JVM/chapter_005/0012.png">



年轻代与老年代
---------





1、存储在JVM中的Java对象可以被划分为两类：

	- 一类是生命周期较短的瞬时对象，这类对象的创建和消亡都非常迅速
	- 另外一类对象的生命周期却非常长，在某些极端的情况下还能够与JVM的生命周期保持一致

2、Java堆区进一步细分的话，可以划分为年轻代（YoungGen）和老年代（oldGen）

3、其中年轻代又可以划分为Eden空间、Survivor0空间和Survivor1空间（有时也叫做from区、to区）

<img src="https://npm.elemecdn.com/youthlql@1.0.8/JVM/chapter_005/0013.png">

<img src="https://npm.elemecdn.com/youthlql@1.0.8/JVM/chapter_005/0014.png">

- 配置新生代与老年代在堆结构的占比

  - 默认**-XX:NewRatio**=2，表示新生代占1，老年代占2，新生代占整个堆的1/3

  - 可以修改**-XX:NewRatio**=4，表示新生代占1，老年代占4，新生代占整个堆的1/5





1. 在HotSpot中，Eden空间和另外两个survivor空间缺省所占的比例是8 : 1 : 1，

2. 当然开发人员可以通过选项**-XX:SurvivorRatio**调整这个空间比例。比如-XX:SurvivorRatio=8

3. 几乎所有的Java对象都是在Eden区被new出来的。

4. 绝大部分的Java对象的销毁都在新生代进行了（有些大的对象在Eden区无法存储时候，将直接进入老年代），IBM公司的专门研究表明，新生代中80%的对象都是“朝生夕死”的。

5. 可以使用选项"-Xmn"设置新生代最大内存大小，但这个参数一般使用默认值就可以了。

   

<img src="https://npm.elemecdn.com/youthlql@1.0.8/JVM/chapter_005/0015.png">



```java
/**
 * -Xms600m -Xmx600m
 *
 * -XX:NewRatio ： 设置新生代与老年代的比例。默认值是2.
 * -XX:SurvivorRatio ：设置新生代中Eden区与Survivor区的比例。默认值是8
 * -XX:-UseAdaptiveSizePolicy ：关闭自适应的内存分配策略  （暂时用不到）
 * -Xmn:设置新生代的空间的大小。 （一般不设置）
 *
 * @author shkstart  shkstart@126.com
 * @create 2020  17:23
 */
public class EdenSurvivorTest {
    public static void main(String[] args) {
        System.out.println("我只是来打个酱油~");
        try {
            Thread.sleep(1000000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```



图解对象分配过程
----------



为新对象分配内存是一件非常严谨和复杂的任务，JVM的设计者们不仅需要考虑内存如何分配、在哪里分配等问题，并且由于内存分配算法与内存回收算法密切相关，所以还需要考虑GC执行完内存回收后是否会在内存空间中产生内存碎片。



**具体过程**

1.  new的对象先放伊甸园区。此区有大小限制。
2.  当伊甸园的空间填满时，程序又需要创建对象，JVM的垃圾回收器将对伊甸园区进行垃圾回收（MinorGC），将伊甸园区中的不再被其他对象所引用的对象进行销毁。再加载新的对象放到伊甸园区。
3.  然后将伊甸园中的剩余对象移动到幸存者0区。
4.  如果再次触发垃圾回收，此时上次幸存下来的放到幸存者0区的，如果没有回收，就会放到幸存者1区。
5.  如果再次经历垃圾回收，此时会重新放回幸存者0区，接着再去幸存者1区。
6.  啥时候能去养老区呢？可以设置次数。默认是15次。可以设置新生区进入养老区的年龄限制，设置 JVM 参数：**-XX:MaxTenuringThreshold**=N 进行设置
7.  在养老区，相对悠闲。当养老区内存不足时，再次触发GC：Major GC，进行养老区的内存清理
8.  若养老区执行了Major GC之后，发现依然无法进行对象的保存，就会产生OOM异常。



### 图解对象分配（一般情况）

1、我们创建的对象，一般都是存放在Eden区的，**当我们Eden区满了后，就会触发GC操作**，一般被称为 YGC / Minor GC操作

<img src="https://npm.elemecdn.com/youthlql@1.0.8/JVM/chapter_005/0016.png">

2、当我们进行一次垃圾收集后，红色的对象将会被回收，而绿色的独享还被占用着，存放在S0(Survivor From)区。同时我们给每个对象设置了一个年龄计数器，经过一次回收后还存在的对象，将其年龄加 1。

3、同时Eden区继续存放对象，当Eden区再次存满的时候，又会触发一个MinorGC操作，此时GC将会把 Eden和Survivor From中的对象进行一次垃圾收集，把存活的对象放到 Survivor To（S1）区，同时让存活的对象年龄 + 1

> 下一次再进行GC的时候，
>
> 1、这一次的s0区为空，所以成为下一次GC的S1区
>
> 2、这一次的s1区则成为下一次GC的S0区
>
> 3、也就是说s0区和s1区在互相转换。

<img src="https://npm.elemecdn.com/youthlql@1.0.8/JVM/chapter_005/0017.png">

4、我们继续不断的进行对象生成和垃圾回收，当Survivor中的对象的年龄达到15的时候，将会触发一次 Promotion 晋升的操作，也就是将年轻代中的对象晋升到老年代中

<img src="https://npm.elemecdn.com/youthlql@1.0.8/JVM/chapter_005/0018.png">

关于垃圾回收：频繁在新生区收集，很少在养老区收集，几乎不在永久区/元空间收集。

### 特殊情况说明



**对象分配的特殊情况**

1.  如果来了一个新对象，先看看 Eden 是否放的下？
    *   如果 Eden 放得下，则直接放到 Eden 区
    *   如果 Eden 放不下，则触发 YGC ，执行垃圾回收，看看还能不能放下？
2.  将对象放到老年区又有两种情况：
    *   如果 Eden 执行了 YGC 还是无法放不下该对象，那没得办法，只能说明是超大对象，只能直接放到老年代
    *   那万一老年代都放不下，则先触发FullGC ，再看看能不能放下，放得下最好，但如果还是放不下，那只能报 OOM 
3.  如果 Eden 区满了，将对象往幸存区拷贝时，发现幸存区放不下啦，那只能便宜了某些新对象，让他们直接晋升至老年区

<img src="https://npm.elemecdn.com/youthlql@1.0.8/JVM/chapter_005/0019.png">

### 常用调优工具



1.  JDK命令行
2.  Eclipse：Memory Analyzer Tool
3.  Jconsole
4.  Visual VM（实时监控，推荐）
5.  Jprofiler（IDEA插件）
6.  Java Flight Recorder（实时监控）
7.  GCViewer
8.  GCEasy



GC分类
----------



1.  我们都知道，JVM的调优的一个环节，也就是垃圾收集，我们需要尽量的避免垃圾回收，因为在垃圾回收的过程中，容易出现STW（Stop the World）的问题，**而 Major GC 和 Full GC出现STW的时间，是Minor GC的10倍以上**
  
2.  JVM在进行GC时，并非每次都对上面三个内存区域一起回收的，大部分时候回收的都是指新生代。针对Hotspot VM的实现，它里面的GC按照回收区域又分为两大种类型：一种是部分收集（Partial GC），一种是整堆收集（FullGC）
  



- 部分收集：不是完整收集整个Java堆的垃圾收集。其中又分为：
  - **新生代收集**（Minor GC/Young GC）：只是新生代（Eden，s0，s1）的垃圾收集
  - **老年代收集**（Major GC/Old GC）：只是老年代的圾收集。
  - 目前，只有CMS GC会有单独收集老年代的行为。
  - 注意，很多时候Major GC会和Full GC混淆使用，需要具体分辨是老年代回收还是整堆回收。
  - 混合收集（Mixed GC）：收集整个新生代以及部分老年代的垃圾收集。目前，只有G1 GC会有这种行为

- **整堆收集**（Full GC）：收集整个java堆和方法区的垃圾收集。

> 由于历史原因，外界各种解读，majorGC和Full GC有些混淆。



### Young GC

**年轻代 GC（Minor GC）触发机制**

1.  当年轻代空间不足时，就会触发Minor GC，这里的年轻代满指的是Eden代满。Survivor满不会主动引发GC，在Eden区满的时候，会顺带触发s0区的GC，也就是被动触发GC（每次Minor GC会清理年轻代的内存）
  
2.  因为Java对象大多都具备朝生夕灭的特性，所以Minor GC非常频繁，一般回收速度也比较快。这一定义既清晰又易于理解。
  
3.  Minor GC会引发STW（Stop The World），暂停其它用户的线程，等垃圾回收结束，用户线程才恢复运行
  

<img src="https://npm.elemecdn.com/youthlql@1.0.8/JVM/chapter_005/0020.png">



### Major/Full GC

> Full GC有争议，后续详解两者区别，暂时先看着

**老年代GC（MajorGC）触发机制**

1.  指发生在老年代的GC，对象从老年代消失时，我们说 “Major Gc” 或 “Full GC” 发生了
2.  出现了MajorGc，经常会伴随至少一次的Minor GC。（但非绝对的，在Parallel Scavenge收集器的收集策略里就有直接进行MajorGC的策略选择过程）

    *   也就是在老年代空间不足时，会先尝试触发Minor GC（哈？我有点迷？），如果之后空间还不足，则触发Major GC
3.  Major GC的速度一般会比Minor GC慢10倍以上，STW的时间更长。
4.  如果Major GC后，内存还不足，就报OOM了



**Full GC 触发机制（后面细讲）**

**触发Full GC执行的情况有如下五种：**

1.  调用System.gc()时，系统建议执行FullGC，但是不必然执行
2.  老年代空间不足
3.  方法区空间不足
4.  通过Minor GC后进入老年代的平均大小大于老年代的可用内存
5.  由Eden区、survivor space0（From Space）区向survivor space1（To Space）区复制时，对象大小大于To Space可用内存，则把该对象转存到老年代，且老年代的可用内存小于该对象大小

说明：Full GC 是开发或调优中尽量要避免的。这样STW时间会短一些



### GC日志分析

```java
/**
 * 测试MinorGC 、 MajorGC、FullGC
 * -Xms9m -Xmx9m -XX:+PrintGCDetails
 * @author shkstart  shkstart@126.com
 * @create 2020  14:19
 */
public class GCTest {
    public static void main(String[] args) {
        int i = 0;
        try {
            List<String> list = new ArrayList<>();
            String a = "atguigu.com";
            while (true) {
                list.add(a);
                a = a + a;
                i++;
            }

        } catch (Throwable t) {
            t.printStackTrace();
            System.out.println("遍历次数为：" + i);
        }
    }
}

```

输出：

```java
[GC (Allocation Failure) [PSYoungGen: 2037K->504K(2560K)] 2037K->728K(9728K), 0.0455865 secs] [Times: user=0.00 sys=0.00, real=0.06 secs] 
[GC (Allocation Failure) [PSYoungGen: 2246K->496K(2560K)] 2470K->1506K(9728K), 0.0009094 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
[GC (Allocation Failure) [PSYoungGen: 2294K->488K(2560K)] 3305K->2210K(9728K), 0.0009568 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
[GC (Allocation Failure) [PSYoungGen: 1231K->488K(2560K)] 7177K->6434K(9728K), 0.0005594 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
[GC (Allocation Failure) [PSYoungGen: 488K->472K(2560K)] 6434K->6418K(9728K), 0.0005890 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
[Full GC (Allocation Failure) [PSYoungGen: 472K->0K(2560K)] [ParOldGen: 5946K->4944K(7168K)] 6418K->4944K(9728K), [Metaspace: 3492K->3492K(1056768K)], 0.0045270 secs] [Times: user=0.00 sys=0.00, real=0.01 secs] 
[GC (Allocation Failure) [PSYoungGen: 0K->0K(1536K)] 4944K->4944K(8704K), 0.0004954 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
[Full GC (Allocation Failure) java.lang.OutOfMemoryError: Java heap space
	at java.util.Arrays.copyOf(Arrays.java:3332)
	at java.lang.AbstractStringBuilder.ensureCapacityInternal(AbstractStringBuilder.java:124)
	at java.lang.AbstractStringBuilder.append(AbstractStringBuilder.java:448)
	at java.lang.StringBuilder.append(StringBuilder.java:136)
	at com.atguigu.java1.GCTest.main(GCTest.java:20)
[PSYoungGen: 0K->0K(1536K)] [ParOldGen: 4944K->4877K(7168K)] 4944K->4877K(8704K), [Metaspace: 3492K->3492K(1056768K)], 0.0076061 secs] [Times: user=0.00 sys=0.02, real=0.01 secs] 
遍历次数为：16
Heap
 PSYoungGen      total 1536K, used 60K [0x00000000ffd00000, 0x0000000100000000, 0x0000000100000000)
  eden space 1024K, 5% used [0x00000000ffd00000,0x00000000ffd0f058,0x00000000ffe00000)
  from space 512K, 0% used [0x00000000fff80000,0x00000000fff80000,0x0000000100000000)
  to   space 1024K, 0% used [0x00000000ffe00000,0x00000000ffe00000,0x00000000fff00000)
 ParOldGen       total 7168K, used 4877K [0x00000000ff600000, 0x00000000ffd00000, 0x00000000ffd00000)
  object space 7168K, 68% used [0x00000000ff600000,0x00000000ffac3408,0x00000000ffd00000)
 Metaspace       used 3525K, capacity 4502K, committed 4864K, reserved 1056768K
  class space    used 391K, capacity 394K, committed 512K, reserved 1048576K
```



```java
[GC (Allocation Failure) [PSYoungGen: 2037K->504K(2560K)] 2037K->728K(9728K), 0.0455865 secs] [Times: user=0.00 sys=0.00, real=0.06 secs] 

```



* [PSYoungGen: 2037K->504K(2560K)]：年轻代总空间为 2560K ，当前占用 2037K ，经过垃圾回收后剩余504K

* 2037K->728K(9728K)：堆内存总空间为 9728K ，当前占用2037K ，经过垃圾回收后剩余728K

    

    

    


堆空间分代思想
---------



为什么要把Java堆分代？不分代就不能正常工作了吗？经研究，不同对象的生命周期不同。70%-99%的对象是临时对象。
*   新生代：有Eden、两块大小相同的survivor（又称为from/to或s0/s1）构成，to总为空。
*   老年代：存放新生代中经历多次GC仍然存活的对象。

<img src="https://npm.elemecdn.com/youthlql@1.0.8/JVM/chapter_005/0021.png">



其实不分代完全可以，分代的唯一理由就是优化GC性能。

- 如果没有分代，那所有的对象都在一块，就如同把一个学校的人都关在一个教室。GC的时候要找到哪些对象没用，这样就会对堆的所有区域进行扫描。（性能低）

*   而很多对象都是朝生夕死的，如果分代的话，把新创建的对象放到某一地方，当GC的时候先把这块存储“朝生夕死”对象的区域进行回收，这样就会腾出很大的空间出来。（多回收新生代，少回收老年代，性能会提高很多）



<img src="https://npm.elemecdn.com/youthlql@1.0.8/JVM/chapter_005/0022.png">



对象内存分配策略
--------



1.  如果对象在Eden出生并经过第一次Minor GC后仍然存活，并且能被Survivor容纳的话，将被移动到Survivor空间中，并将对象年龄设为1。
2.  对象在Survivor区中每熬过一次MinorGC，年龄就增加1岁，当它的年龄增加到一定程度（默认为15岁，其实每个JVM、每个GC都有所不同）时，就会被晋升到老年代
3.  对象晋升老年代的年龄阀值，可以通过选项**-XX:MaxTenuringThreshold**来设置



**针对不同年龄段的对象分配原则如下所示：**

1.  **优先分配到Eden**：开发中比较长的字符串或者数组，会直接存在老年代，但是因为新创建的对象都是朝生夕死的，所以这个大对象可能也很快被回收，但是因为老年代触发Major GC的次数比 Minor GC要更少，因此可能回收起来就会比较慢
2.  **大对象直接分配到老年代**：尽量避免程序中出现过多的大对象
3.  **长期存活的对象分配到老年代**
4.  **动态对象年龄判断**：如果Survivor区中相同年龄的所有对象大小的总和大于Survivor空间的一半，年龄大于或等于该年龄的对象可以直接进入老年代，无须等到MaxTenuringThreshold中要求的年龄。
5.  **空间分配担保**： -XX:HandlePromotionFailure 。



> 一些细节放在后面说



TLAB为对象分配内存（保证线程安全）
---------

### 为什么有 TLAB



2.  堆区是线程共享区域，任何线程都可以访问到堆区中的共享数据
3.  由于对象实例的创建在JVM中非常频繁，因此在并发环境下从堆区中划分内存空间是线程不安全的
4.  为避免多个线程操作同一地址，需要使用**加锁等机制**，进而影响分配速度。



### 什么是 TLAB

TLAB（Thread Local Allocation Buffer）

1.  从内存模型而不是垃圾收集的角度，对Eden区域继续进行划分，**JVM为每个线程分配了一个私有缓存区域，它包含在Eden空间内**。
2.  多线程同时分配内存时，使用TLAB可以避免一系列的非线程安全问题，同时还能够提升内存分配的吞吐量，因此我们可以将这种内存分配方式称之为**快速分配策略**。
3.  据我所知所有OpenJDK衍生出来的JVM都提供了TLAB的设计。

<img src="https://npm.elemecdn.com/youthlql@1.0.8/JVM/chapter_005/0023.png">

1、每个线程都有一个TLAB空间

2、当一个线程的TLAB存满时，可以使用公共区域（蓝色）的



### TLAB再说明



1. 尽管不是所有的对象实例都能够在TLAB中成功分配内存，但**JVM确实是将TLAB作为内存分配的首选**。

2. 在程序中，开发人员可以通过选项“**-XX:UseTLAB**”设置是否开启TLAB空间。

3. 默认情况下，TLAB空间的内存非常小，仅占有整个Eden空间的1%，当然我们可以通过选项“**-XX:TLABWasteTargetPercent**”设置TLAB空间所占用Eden空间的百分比大小。

4. 一旦对象在TLAB空间分配内存失败时，JVM就会尝试着通过**使用加锁机制确保数据操作的原子性**，从而直接在Eden空间中分配内存。

   

> 1、哪个线程要分配内存，就在哪个线程的本地缓冲区中分配，只有本地缓冲区用完 了，**分配新的缓存区时才需要同步锁定**                   ----这是《深入理解JVM》--第三版里说的
>
> 2、和这里讲的有点不同。我猜测说的意思是某一次分配，如果TLAB用完了，那么**这一次**先在Eden区直接分配。空闲下来后再加锁分配新的TLAB（TLAB内存较大，分配时间应该较长）



**TLAB 分配过程**

<img src="https://npm.elemecdn.com/youthlql@1.0.8/JVM/chapter_005/0024.png">



堆空间参数设置
---------

### 常用参数设置

> **官方文档**：https://docs.oracle.com/javase/8/docs/technotes/tools/unix/java.html
>
> 我们只说常用的



```java
/**
 * 测试堆空间常用的jvm参数：
 * -XX:+PrintFlagsInitial : 查看所有的参数的默认初始值
 * -XX:+PrintFlagsFinal  ：查看所有的参数的最终值（可能会存在修改，不再是初始值）
 *      具体查看某个参数的指令： jps：查看当前运行中的进程
 *                             jinfo -flag SurvivorRatio 进程id
 *
 * -Xms：初始堆空间内存 （默认为物理内存的1/64）
 * -Xmx：最大堆空间内存（默认为物理内存的1/4）
 * -Xmn：设置新生代的大小。(初始值及最大值)
 * -XX:NewRatio：配置新生代与老年代在堆结构的占比
 * -XX:SurvivorRatio：设置新生代中Eden和S0/S1空间的比例
 * -XX:MaxTenuringThreshold：设置新生代垃圾的最大年龄
 * -XX:+PrintGCDetails：输出详细的GC处理日志
 * 打印gc简要信息：① -XX:+PrintGC   ② -verbose:gc
 * -XX:HandlePromotionFailure：是否设置空间分配担保
 */
```



### 空间分配担保



1、在发生Minor GC之前，虚拟机会检查老年代最大可用的连续空间是否大于新生代所有对象的总空间。

*   如果大于，则此次Minor GC是安全的
*   如果小于，则虚拟机会查看**-XX:HandlePromotionFailure**设置值是否允担保失败。
    *   如果HandlePromotionFailure=true，那么会继续检查**老年代最大可用连续空间是否大于历次晋升到老年代的对象的平均大小**。
        *   如果大于，则尝试进行一次Minor GC，但这次Minor GC依然是有风险的；
        *   如果小于，则进行一次Full GC。
    *   如果HandlePromotionFailure=false，则进行一次Full GC。

    
    

**历史版本**

1.  在JDK6 Update 24之后，HandlePromotionFailure参数不会再影响到虚拟机的空间分配担保策略，观察openJDK中的源码变化，虽然源码中还定义了HandlePromotionFailure参数，但是在代码中已经不会再使用它。
2.  JDK6 Update 24之后的规则变为**只要老年代的连续空间大于新生代对象总大小或者历次晋升的平均大小就会进行Minor GC**，否则将进行Full GC。即 HandlePromotionFailure=true





堆是分配对象的唯一选择么？
--------



**在《深入理解Java虚拟机》中关于Java堆内存有这样一段描述：**

1.  随着JIT编译期的发展与**逃逸分析技术**逐渐成熟，**栈上分配、标量替换**优化技术将会导致一些微妙的变化，所有的对象都分配到堆上也渐渐变得不那么“绝对”了。
  
2.  在Java虚拟机中，对象是在Java堆中分配内存的，这是一个普遍的常识。但是，有一种特殊情况，那就是**如果经过逃逸分析（Escape Analysis）后发现，一个对象并没有逃逸出方法的话，那么就可能被优化成栈上分配**。这样就无需在堆上分配内存，也无须进行垃圾回收了。这也是最常见的堆外存储技术。
  
3.  此外，前面提到的基于OpenJDK深度定制的TaoBao VM，其中创新的GCIH（GC invisible heap）技术实现off-heap，将生命周期较长的Java对象从heap中移至heap外，并且GC不能管理GCIH内部的Java对象，以此达到降低GC的回收频率和提升GC的回收效率的目的。
  



### 逃逸分析



1.  如何将堆上的对象分配到栈，需要使用逃逸分析手段。
2.  这是一种可以有效减少Java程序中同步负载和内存堆分配压力的跨函数全局数据流分析算法。
3.  通过逃逸分析，Java Hotspot编译器能够分析出一个新的对象的引用的使用范围从而决定是否要将这个对象分配到堆上。
4.  逃逸分析的基本行为就是分析对象动态作用域：
    *   当一个对象在方法中被定义后，对象只在方法内部使用，则认为没有发生逃逸。
    *   当一个对象在方法中被定义后，它被外部方法所引用，则认为发生逃逸。例如作为调用参数传递到其他地方中。



**逃逸分析举例**

1、没有发生逃逸的对象，则可以分配到栈（无线程安全问题）上，随着方法执行的结束，栈空间就被移除（也就无需GC）

```java
public void my_method() {
    V v = new V();
    // use v
    // ....
    v = null;
}
```



2、下面代码中的 StringBuffer sb 发生了逃逸，不能在栈上分配

````java
public static StringBuffer createStringBuffer(String s1, String s2) {
    StringBuffer sb = new StringBuffer();
    sb.append(s1);
    sb.append(s2);
    return sb;
}
````



3、如果想要StringBuffer sb不发生逃逸，可以这样写

```java
public static String createStringBuffer(String s1, String s2) {
    StringBuffer sb = new StringBuffer();
    sb.append(s1);
    sb.append(s2);
    return sb.toString();
}
```



```java

/**
 * 逃逸分析
 *
 *  如何快速的判断是否发生了逃逸分析，大家就看new的对象实体是否有可能在方法外被调用。
 */
public class EscapeAnalysis {

    public EscapeAnalysis obj;

    /*
    方法返回EscapeAnalysis对象，发生逃逸
     */
    public EscapeAnalysis getInstance(){
        return obj == null? new EscapeAnalysis() : obj;
    }
    /*
    为成员属性赋值，发生逃逸
     */
    public void setObj(){
        this.obj = new EscapeAnalysis();
    }
    //思考：如果当前的obj引用声明为static的？仍然会发生逃逸。

    /*
    对象的作用域仅在当前方法中有效，没有发生逃逸
     */
    public void useEscapeAnalysis(){
        EscapeAnalysis e = new EscapeAnalysis();
    }
    /*
    引用成员变量的值，发生逃逸
     */
    public void useEscapeAnalysis1(){
        EscapeAnalysis e = getInstance();
        //getInstance().xxx()同样会发生逃逸
    }
}

```



**逃逸分析参数设置**

1.  在JDK 1.7 版本之后，HotSpot中默认就已经开启了逃逸分析
  
2.  如果使用的是较早的版本，开发人员则可以通过：
  
    *   选项“-XX:+DoEscapeAnalysis"显式开启逃逸分析
    *   通过选项“-XX:+PrintEscapeAnalysis"查看逃逸分析的筛选结果



**总结**

开发中能使用局部变量的，就不要使用在方法外定义。



### 代码优化

使用逃逸分析，编译器可以对代码做如下优化：

1.  **栈上分配**：将堆分配转化为栈分配。如果一个对象在子程序中被分配，要使指向该对象的指针永远不会发生逃逸，对象可能是栈上分配的候选，而不是堆上分配
2.  **同步省略**：如果一个对象被发现只有一个线程被访问到，那么对于这个对象的操作可以不考虑同步。
3.  **分离对象或标量替换**：有的对象可能不需要作为一个连续的内存结构存在也可以被访问到，那么对象的部分（或全部）可以不存储在内存，而是存储在CPU寄存器中。



### 栈上分配



1.  JIT编译器在编译期间根据逃逸分析的结果，发现如果一个对象并没有逃逸出方法的话，就可能被优化成栈上分配。分配完成后，继续在调用栈内执行，最后线程结束，栈空间被回收，局部变量对象也被回收。这样就无须进行垃圾回收了。
3.  常见的栈上分配的场景：在逃逸分析中，已经说明了，分别是给成员变量赋值、方法返回值、实例引用传递。



**栈上分配举例**

```java
/**
 * 栈上分配测试
 * -Xmx128m -Xms128m -XX:-DoEscapeAnalysis -XX:+PrintGCDetails
 */
public class StackAllocation {
    public static void main(String[] args) {
        long start = System.currentTimeMillis();

        for (int i = 0; i < 10000000; i++) {
            alloc();
        }
        // 查看执行时间
        long end = System.currentTimeMillis();
        System.out.println("花费的时间为： " + (end - start) + " ms");
        // 为了方便查看堆内存中对象个数，线程sleep
        try {
            Thread.sleep(1000000);
        } catch (InterruptedException e1) {
            e1.printStackTrace();
        }
    }

    private static void alloc() {
        User user = new User();//未发生逃逸
    }

    static class User {

    }
}
```

输出结果：

```
[GC (Allocation Failure) [PSYoungGen: 33280K->808K(38400K)] 33280K->816K(125952K), 0.0483350 secs] [Times: user=0.00 sys=0.00, real=0.06 secs] 
[GC (Allocation Failure) [PSYoungGen: 34088K->808K(38400K)] 34096K->816K(125952K), 0.0008411 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
[GC (Allocation Failure) [PSYoungGen: 34088K->792K(38400K)] 34096K->800K(125952K), 0.0008427 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
[GC (Allocation Failure) [PSYoungGen: 34072K->808K(38400K)] 34080K->816K(125952K), 0.0012223 secs] [Times: user=0.08 sys=0.00, real=0.00 secs] 
花费的时间为： 114 ms
```





1、JVM 参数设置

-Xmx128m -Xms128m -XX:-DoEscapeAnalysis -XX:+PrintGCDetails

2、日志打印：发生了 GC ，耗时 114ms





**开启逃逸分析的情况**

输出结果：

```
花费的时间为： 5 ms
```

1、参数设置

-Xmx128m -Xms128m -XX:+DoEscapeAnalysis -XX:+PrintGCDetails

2、日志打印：并没有发生 GC ，耗时5ms 。



### 同步省略（同步消除）



1.  线程同步的代价是相当高的，同步的后果是降低并发性和性能。
  
2.  在动态编译同步块的时候，JIT编译器可以借助逃逸分析来**判断同步块所使用的锁对象是否只能够被一个线程访问而没有被发布到其他线程**。
  
3.  如果没有，那么JIT编译器在编译这个同步块的时候就会取消对这部分代码的同步。这样就能大大提高并发性和性能。这个**取消同步的过程就叫同步省略，也叫锁消除**。
  



例如下面的代码

```java
public void f() {
    Object hollis = new Object();
    synchronized(hollis) {
        System.out.println(hollis);
    }
}
```



代码中对hollis这个对象加锁，但是hollis对象的生命周期只在f()方法中，并不会被其他线程所访问到，所以在JIT编译阶段就会被优化掉，优化成：

```java
public void f() {
    Object hellis = new Object();
	System.out.println(hellis);
}
```





**字节码分析**

```
public class SynchronizedTest {
    public void f() {
        Object hollis = new Object();
        synchronized(hollis) {
            System.out.println(hollis);
        }
    }
}
```



```java
 0 new #2 <java/lang/Object>
 3 dup
 4 invokespecial #1 <java/lang/Object.<init>>
 7 astore_1
 8 aload_1
 9 dup
10 astore_2
11 monitorenter
12 getstatic #3 <java/lang/System.out>
15 aload_1
16 invokevirtual #4 <java/io/PrintStream.println>
19 aload_2
20 monitorexit
21 goto 29 (+8)
24 astore_3
25 aload_2
26 monitorexit
27 aload_3
28 athrow
29 return
```

注意：字节码文件中并没有进行优化，可以看到加锁和释放锁的操作依然存在，**同步省略操作是在解释运行时发生的**



### 标量替换

**分离对象或标量替换**

1.  标量（scalar）是指一个无法再分解成更小的数据的数据。Java中的原始数据类型就是标量。
  
2.  相对的，那些还可以分解的数据叫做聚合量（Aggregate），Java中的对象就是聚合量，因为他可以分解成其他聚合量和标量。
  
3.  在JIT阶段，如果经过逃逸分析，发现一个对象不会被外界访问的话，那么经过JIT优化，就会把这个对象拆解成若干个其中包含的若干个成员变量来代替。这个过程就是标量替换。
  



**标量替换举例**

代码

```java
public static void main(String args[]) {
    alloc();
}
private static void alloc() {
    Point point = new Point(1,2);
    System.out.println("point.x" + point.x + ";point.y" + point.y);
}
class Point {
    private int x;
    private int y;
}
```



以上代码，经过标量替换后，就会变成

```java
private static void alloc() {
    int x = 1;
    int y = 2;
    System.out.println("point.x = " + x + "; point.y=" + y);
}
```



1.  可以看到，Point这个聚合量经过逃逸分析后，发现他并没有逃逸，就被替换成两个聚合量了。
2.  那么标量替换有什么好处呢？就是可以大大减少堆内存的占用。因为一旦不需要创建对象了，那么就不再需要分配堆内存了。
3.  标量替换为栈上分配提供了很好的基础。



**标量替换参数设置**

参数 -XX:+ElimilnateAllocations：开启了标量替换（默认打开），允许将对象打散分配在栈上。



**代码示例**

```java
/**
 * 标量替换测试
 *  -Xmx100m -Xms100m -XX:+DoEscapeAnalysis -XX:+PrintGC -XX:-EliminateAllocations
 * @author shkstart  shkstart@126.com
 * @create 2020  12:01
 */
public class ScalarReplace {
    public static class User {
        public int id;
        public String name;
    }

    public static void alloc() {
        User u = new User();//未发生逃逸
        u.id = 5;
        u.name = "www.atguigu.com";
    }

    public static void main(String[] args) {
        long start = System.currentTimeMillis();
        for (int i = 0; i < 10000000; i++) {
            alloc();
        }
        long end = System.currentTimeMillis();
        System.out.println("花费的时间为： " + (end - start) + " ms");
    }
}
```



**未开启标量替换**

1、JVM 参数

-Xmx100m -Xms100m -XX:+DoEscapeAnalysis -XX:+PrintGC -XX:-EliminateAllocations

2、日志

```java
[GC (Allocation Failure)  25600K->880K(98304K), 0.0012658 secs]
[GC (Allocation Failure)  26480K->832K(98304K), 0.0012124 secs]
[GC (Allocation Failure)  26432K->784K(98304K), 0.0009719 secs]
[GC (Allocation Failure)  26384K->832K(98304K), 0.0009071 secs]
[GC (Allocation Failure)  26432K->768K(98304K), 0.0010643 secs]
[GC (Allocation Failure)  26368K->824K(101376K), 0.0012354 secs]
[GC (Allocation Failure)  32568K->712K(100864K), 0.0011291 secs]
[GC (Allocation Failure)  32456K->712K(100864K), 0.0006368 secs]
花费的时间为： 99 ms
```



**开启标量替换**

1、JVM 参数

-Xmx100m -Xms100m -XX:+DoEscapeAnalysis -XX:+PrintGC -XX:+EliminateAllocations

2、日志：时间减少很多，且无GC

```
花费的时间为： 6 ms
```





上述代码在主函数中调用了1亿次alloc()方法，进行对象创建由于User对象实例需要占据约16字节的空间，因此累计分配空间达到将近1.5GB。如果堆空间小于这个值，就必然会发生GC。使用如下参数运行上述代码：

`-server -Xmx100m -Xms100m -XX:+DoEscapeAnalysis -XX:+PrintGC -XX:+EliminateAllocations`

这里设置参数如下：

1.  参数 -server：启动Server模式，因为在server模式下，才可以启用逃逸分析。
2.  参数 -XX:+DoEscapeAnalysis：启用逃逸分析
3.  参数 -Xmx10m：指定了堆空间最大为10MB
4.  参数 -XX:+PrintGC：将打印GC日志。
5.  参数 -XX:+EliminateAllocations：开启了标量替换（默认打开），允许将对象打散分配在栈上，比如对象拥有id和name两个字段，那么这两个字段将会被视为两个独立的局部变量进行分配

### 逃逸分析的不足

1.  关于逃逸分析的论文在1999年就已经发表了，但直到JDK1.6才有实现，而且这项技术到如今也并不是十分成熟的。
2.  其根本原因就是无法保证逃逸分析的性能消耗一定能高于他的消耗。虽然经过逃逸分析可以做标量替换、栈上分配、和锁消除。但是逃逸分析自身也是需要进行一系列复杂的分析的，这其实也是一个相对耗时的过程。
3.  一个极端的例子，就是经过逃逸分析之后，发现没有一个对象是不逃逸的。那这个逃逸分析的过程就白白浪费掉了。
4.  虽然这项技术并不十分成熟，但是它也是即时编译器优化技术中一个十分重要的手段。
5.  注意到有一些观点，认为通过逃逸分析，JVM会在栈上分配那些不会逃逸的对象，这在理论上是可行的，但是取决于JVM设计者的选择。据我所知，**Oracle Hotspot JVM中并未这么做**（刚刚演示的效果，是因为HotSpot实现了标量替换），这一点在逃逸分析相关的文档里已经说明，**所以可以明确在HotSpot虚拟机上，所有的对象实例都是创建在堆上**。
6.  目前很多书籍还是基于JDK7以前的版本，JDK已经发生了很大变化，intern字符串的缓存和静态变量曾经都被分配在永久代上，而永久代已经被元数据区取代。但是**intern字符串缓存和静态变量并不是被转移到元数据区，而是直接在堆上分配**，**所以这一点同样符合前面一点的结论：对象实例都是分配在堆上**。




> **堆是分配对象的唯一选择么？**

综上：**对象实例都是分配在堆上**。What the fuck？



小结
------



1.  年轻代是对象的诞生、成长、消亡的区域，一个对象在这里产生、应用，最后被垃圾回收器收集、结束生命。
  
2.  老年代放置长生命周期的对象，通常都是从Survivor区域筛选拷贝过来的Java对象。
  
3.  当然，也有特殊情况，我们知道普通的对象可能会被分配在TLAB上；
  
4.  如果对象较大，无法分配在 TLAB 上，则JVM会试图直接分配在Eden其他位置上；
  
5.  如果对象太大，完全无法在新生代找到足够长的连续空闲空间，JVM就会直接分配到老年代。
  
6.  当GC只发生在年轻代中，回收年轻代对象的行为被称为Minor GC。
  
7.  当GC发生在老年代时则被称为Major GC或者Full GC。
  
8.  一般的，Minor GC的发生频率要比Major GC高很多，即老年代中垃圾回收发生的频率将大大低于年轻代。