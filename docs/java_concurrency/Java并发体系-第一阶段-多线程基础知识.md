---
title: Java并发体系-第一阶段-多线程基础知识
tags:
  - Java并发
  - 原理
  - 源码
categories:
  - Java并发
  - 原理
keywords: Java并发，原理，源码
description: 万字系列长文讲解Java并发-第一阶段-多线程基础知识。
cover: 'https://npm.elemecdn.com/lql_static@latest/logo/Java_concurrency.png'
abbrlink: efc79183
date: 2020-10-05 22:40:58
---





# 程序、进程、线程的理解

1、程序(programm)
概念：是为完成特定任务、用某种语言编写的一组指令的集合。即指一段静态的代码。

2、进程(process)
概念：程序的一次执行过程，或是正在运行的一个程序。
说明：进程作为资源分配的单位，系统在运行时会为每个进程分配不同的内存区域

3、线程(thread)
概念：进程可进一步细化为线程，是一个程序内部的一条执行路径。
说明：线程作为CPU调度和执行的单位，每个线程拥独立的运行栈和程序计数器(pc)，线程切换的开销小。

<img src="https://npm.elemecdn.com/youthlql@1.0.8/Java_concurrency/Source_code/First_stage/0001.png">



补充：

<img src="https://npm.elemecdn.com/youthlql@1.0.8/Java_concurrency/Source_code/First_stage/0002.png">

进程可以细化为多个线程。
每个线程，拥有自己独立的：栈、程序计数器
多个线程，共享同一个进程中的结构：方法区、堆。



# 并行与并发

## 单核CPU与多核CPU的理解
- 单核CPU，其实是一种假的多线程，因为在一个时间单元内，也只能执行一个线程的任务。例如：虽然有多车道，但是收费站只有一个工作人员在收费，只有收了费才能通过，那么CPU就好比收费人员。如果某个人不想交钱，那么收费人员可以把他“挂起”（晾着他，等他想通了，准备好了钱，再去收费。）但是因为CPU时间单元特别短，因此感觉不出来。
- 如果是多核的话，才能更好的发挥多线程的效率。（现在的服务器都是多核的）
- 一个Java应用程序java.exe，其实至少三个线程：main()主线程，gc()垃圾回收线程，异常处理线程。当然如果发生异常，会影响主线程。

## 并行与并发的理解
并行：多个CPU同时执行多个任务。比如：多个人同时做不同的事。
并发：一个CPU(采用时间片)同时执行多个任务。比如：秒杀、多个人做同一件事

# 创建线程的几种方法



## 继承Thread类创建线程

多线程的创建，方式一：继承于Thread类
1. 创建一个继承于Thread类的子类
2. 重写Thread类的run() --> 将此线程执行的操作声明在run()中
3. 创建Thread类的子类的对象
4. 通过此对象调用start()

* 例子：遍历100以内的所有的偶数

```Java

//1. 创建一个继承于Thread类的子类
class MyThread extends Thread {
    //2. 重写Thread类的run()
    @Override
    public void run() {
        for (int i = 0; i < 100; i++) {
            if(i % 2 == 0){
                System.out.println(Thread.currentThread().getName() + ":" + i);
            }
        }
    }
}


public class ThreadTest {
    public static void main(String[] args) {
        //3. 创建Thread类的子类的对象
        MyThread t1 = new MyThread();

        //4.通过此对象调用start():①启动当前线程 ② 调用当前线程的run()
        t1.start();
        //问题一：我们不能通过直接调用run()的方式启动线程。
//        t1.run();

        /*
        问题二：再启动一个线程，遍历100以内的偶数。不可以还让已经start()的线程去执行。
        会报IllegalThreadStateException
        */    
//        t1.start();
        //我们需要重新创建一个线程的对象
        MyThread t2 = new MyThread();
        t2.start();


        //如下操作仍然是在main线程中执行的。
        for (int i = 0; i < 100; i++) {
            if(i % 2 == 0){
                System.out.println(Thread.currentThread().getName() + ":" + i + "***********main()************");
            }
        }
    }

}

```



## 实现Runnable接口创建线程

1、创建多线程的方式二：实现Runnable接口

1. 创建一个实现了Runnable接口的类
2. 实现类去实现Runnable中的抽象方法：run()
3. 创建实现类的对象
4. 将此对象作为参数传递到Thread类的构造器中，创建Thread类的对象
5. 通过Thread类的对象调用start()

2、 比较创建线程的两种方式。
 开发中：优先选择：实现Runnable接口的方式
 原因：实现的方式没有类的单继承性的局限性，实现的方式更适合来处理多个线程有共享数据的情况。 



```java

//1. 创建一个实现了Runnable接口的类
class MThread implements Runnable{

    //2. 实现类去实现Runnable中的抽象方法：run()
    @Override
    public void run() {
        for (int i = 0; i < 100; i++) {
            if(i % 2 == 0){
                System.out.println(Thread.currentThread().getName() + ":" + i);
            }

        }
    }
}


public class ThreadTest1 {
    public static void main(String[] args) {
        //3. 创建实现类的对象
        MThread mThread = new MThread();
        //4. 将此对象作为参数传递到Thread类的构造器中，创建Thread类的对象
        Thread t1 = new Thread(mThread);
        t1.setName("线程1");

        /*
        5. 通过Thread类的对象调用start():① 启动线程 ②调用当前线程的run()-->
        调用了Runnable类型的target的run()
        */
        t1.start();

        //再启动一个线程，遍历100以内的偶数
        Thread t2 = new Thread(mThread);
        t2.setName("线程2");
        t2.start();
    }

}
```



## Thread和Runnable的关系

 联系：public class Thread implements Runnable
 相同点：两种方式都需要重写run(),将线程要执行的逻辑声明在run()中。



### Runnable接口构造线程源码

```java
/*下面是Thread类的部分源码*/

//1.用Runnable接口创建线程时会进入这个方法
public Thread(Runnable target) {
        init(null, target, "Thread-" + nextThreadNum(), 0);
    }

//2.接着调用这个方法
private void init(ThreadGroup g, Runnable target, String name,
                      long stackSize) {
        init(g, target, name, stackSize, null, true);
    }

//3.再调用这个方法
private void init(ThreadGroup g, Runnable target, String name,
                      long stackSize, AccessControlContext acc,
                      boolean inheritThreadLocals) {
        if (name == null) {
            throw new NullPointerException("name cannot be null");
        }

        this.name = name;

        Thread parent = currentThread();
        SecurityManager security = System.getSecurityManager();
        if (g == null) {
            /* Determine if it's an applet or not */

            /* If there is a security manager, ask the security manager
               what to do. */
            if (security != null) {
                g = security.getThreadGroup();
            }

            /* If the security doesn't have a strong opinion of the matter
               use the parent thread group. */
            if (g == null) {
                g = parent.getThreadGroup();
            }
        }

        /* checkAccess regardless of whether or not threadgroup is
           explicitly passed in. */
        g.checkAccess();

        /*
         * Do we have the required permissions?
         */
        if (security != null) {
            if (isCCLOverridden(getClass())) {
                security.checkPermission(SUBCLASS_IMPLEMENTATION_PERMISSION);
            }
        }

        g.addUnstarted();

        this.group = g;
        this.daemon = parent.isDaemon();
        this.priority = parent.getPriority();
        if (security == null || isCCLOverridden(parent.getClass()))
            this.contextClassLoader = parent.getContextClassLoader();
        else
            this.contextClassLoader = parent.contextClassLoader;
        this.inheritedAccessControlContext =
                acc != null ? acc : AccessController.getContext();
        //4.最后在这里将Runnable接口(target)赋值给Thread自己的target成员属性     
        this.target = target;
        setPriority(priority);
        if (inheritThreadLocals && parent.inheritableThreadLocals != null)
            this.inheritableThreadLocals =
                ThreadLocal.createInheritedMap(parent.inheritableThreadLocals);
        /* Stash the specified stack size in case the VM cares */
        this.stackSize = stackSize;

        /* Set thread ID */
        tid = nextThreadID();
    }

/*如果你是实现了runnable接口，那么在上面的代码中target便不会为null，那么最终就会通过重写的
规则去调用真正实现了Runnable接口(你之前传进来的那个Runnable接口实现类)的类里的run方法*/
@Override
    public void run() {
    
        if (target != null) {
            target.run();
        }
    }
```

1、多线程的设计之中，使用了代理设计模式的结构，用户自定义的线程主体只是负责项目核心功能的实现，而所有的辅助实现全部交由Thread类来处理。
2、在进行Thread启动多线程的时候调用的是start()方法，而后找到的是run()方法，但通过Thread类的构造方法传递了一个Runnable接口对象的时候，那么该接口对象将被Thread类中的target属性所保存，在start()方法执行的时候会调用Thread类中的run()方法。而这个run()方法去调用实现了Runnable接口的那个类所重写过run()方法，进而执行相应的逻辑。多线程开发的本质实质上是在于多个线程可以进行同一资源的抢占，那么Thread主要描述的是线程，而资源的描述是通过Runnable完成的。如下图所示：

<img src="https://npm.elemecdn.com/youthlql@1.0.8/Java_concurrency/Source_code/First_stage/0003.png">





### Thread类构造线程源码

```java
MyThread t2 = new MyThread(); //这个构造函数会默认调用Super();也就是Thread类的无参构造
```



```java
//代码从上往下顺序执行
public Thread() {
        init(null, null, "Thread-" + nextThreadNum(), 0);
    }
    
private void init(ThreadGroup g, Runnable target, String name,
                      long stackSize) {
        init(g, target, name, stackSize, null, true);
    }
    
private void init(ThreadGroup g, Runnable target, String name,
                      long stackSize, AccessControlContext acc,
                      boolean inheritThreadLocals) {
        if (name == null) {
            throw new NullPointerException("name cannot be null");
        }

        this.name = name;

        Thread parent = currentThread();
        SecurityManager security = System.getSecurityManager();
        if (g == null) {
            /* Determine if it's an applet or not */

            /* If there is a security manager, ask the security manager
               what to do. */
            if (security != null) {
                g = security.getThreadGroup();
            }

            /* If the security doesn't have a strong opinion of the matter
               use the parent thread group. */
            if (g == null) {
                g = parent.getThreadGroup();
            }
        }

        /* checkAccess regardless of whether or not threadgroup is
           explicitly passed in. */
        g.checkAccess();

        /*
         * Do we have the required permissions?
         */
        if (security != null) {
            if (isCCLOverridden(getClass())) {
                security.checkPermission(SUBCLASS_IMPLEMENTATION_PERMISSION);
            }
        }

        g.addUnstarted();

        this.group = g;
        this.daemon = parent.isDaemon();
        this.priority = parent.getPriority();
        if (security == null || isCCLOverridden(parent.getClass()))
            this.contextClassLoader = parent.getContextClassLoader();
        else
            this.contextClassLoader = parent.contextClassLoader;
        this.inheritedAccessControlContext =
                acc != null ? acc : AccessController.getContext();
        this.target = target;
        setPriority(priority);
        if (inheritThreadLocals && parent.inheritableThreadLocals != null)
            this.inheritableThreadLocals =
                ThreadLocal.createInheritedMap(parent.inheritableThreadLocals);
        /* Stash the specified stack size in case the VM cares */
        this.stackSize = stackSize;

        /* Set thread ID */
        tid = nextThreadID();
    }

/*由于这里是通过继承Thread类来实现的线程，那么target这个东西就是Null。但是因为你继承
了Runnable接口并且重写了run()，所以最终还是调用子类的run()*/
 @Override
    public void run() {
        if (target != null) {
            target.run();
        }
    }
```



### 最直观的代码描述

```java
class Window extends Thread{


    private  int ticket = 100;
    @Override
    public void run() {

        while(true){

            if(ticket > 0){
                System.out.println(getName() + "：卖票，票号为：" + ticket);
                ticket--;
            }else{
                break;
            }

        }

    }
}


public class WindowTest {
    public static void main(String[] args) {
        Window t1 = new Window();
        Window t2 = new Window();
        Window t3 = new Window();


        t1.setName("窗口1");
        t2.setName("窗口2");
        t3.setName("窗口3");

        t1.start();
        t2.start();
        t3.start();

    }
}
```



```java
class Window1 implements Runnable{

    private int ticket = 100;

    @Override
    public void run() {
        while(true){
            if(ticket > 0){
                System.out.println(Thread.currentThread().getName() + ":卖票，票号为：" + ticket);
                ticket--;
            }else{
                break;
            }
        }
    }
}


public class WindowTest1 {
    public static void main(String[] args) {
        Window1 w = new Window1();

        Thread t1 = new Thread(w);
        Thread t2 = new Thread(w);
        Thread t3 = new Thread(w);

        t1.setName("窗口1");
        t2.setName("窗口2");
        t3.setName("窗口3");

        t1.start();
        t2.start();
        t3.start();
    }

}
```

1、继承Thread类的方式，new了三个Thread，实际上是有300张票。

2、实现Runnable接口的方式，new了三个Thread，实际上是有100张票。

3、也就是说实现Runnable接口的线程中，成员属性是所有线程共有的。但是继承Thread类的线程中，成员属性是各个线程独有的，其它线程看不到，除非采用static的方式才能使各个线程都能看到。

4、就像上面说的Runnable相当于资源，Thread才是线程。用Runnable创建线程时，new了多个Thread，但是传进去的参数都是同一个Runnable（资源）。用Thread创建线程时，就直接new了多个线程，每个线程都有自己的Runnable（资源）。在Thread源码中就是用target变量（这是一个Runnable类型的变量）来表示这个资源。

5、同时因为这两个的区别，在并发编程中，继承了Thread的子类在进行线程同步时不能将成员变量当做锁，因为多个线程拿到的不是同一把锁，不过用static变量可以解决这个问题。而实现了Runnable接口的类在进行线程同步时没有这个问题。





## 实现Callable接口创建线程

```Java
//Callable实现多线程
class MyThread implements Callable<String> {//线程的主体类

    @Override
    public String call() throws Exception {
        for (int x = 0; x < 10; x++) {
            System.out.println("*******线程执行，x=" + x + "********");
        }
        return "线程执行完毕";
    }
}

public class Demo1 {
    public static void main(String[] args) throws Exception {
        FutureTask<String> task = new FutureTask<>(new MyThread());
        new Thread(task).start();
        System.out.println("线程返回数据" + task.get());

    }
}
```

Callable最主要的就是提供带有返回值的call方法来创建线程。不过Callable要和Future实现类连着用，关于Future的一系列知识会在后面几个系列讲到。



# 策略模式在Thread和Runnable中的应用

Runnable接口最重要的方法-----run方法，使用了**策略者模式**将执行的逻辑(run方法)和程序的执行单元(start0方法)分离出来，使用户可以定义自己的程序处理逻辑，更符合面向对象的思想。







# Thread的构造方法

- 创建线程对象Thread，`默认有一个线程名，以Thread-开头，从0开始计数`，如“Thread-0、Thread-1、Thread-2 …”

- 如果没有传递Runnable或者没有覆写Thread的run方法，`该Thread不会调用任何方法`

- 如果传递Runnable接口的实例或者覆写run方法，则`会执行该方法的逻辑单元`（逻辑代码）

- 如果构造线程对象时，未传入ThreadGroup，`Thread会默认获取父线程的ThreadGroup作为该线程的ThreadGroup`，此时子线程和父线程会在同一个ThreadGroup中

- stackSize可以`提高线程栈的深度`，放更多栈帧，但是会`减少能创建的线程数目`

- stackSize默认是0，`如果是0，代表着被忽略，该参数会被JNI函数调用`，但是注意某些平台可能会失效，`可以通过“-Xss10m”设置`

具体的介绍可以看Java的API文档

```Java
/*下面是Thread 的部分源码*/

public Thread(Runnable target) {
    init(null, target, "Thread-" + nextThreadNum(), 0);
}

public Thread(String name) {
    init(null, null, name, 0);
}
		↓ ↓	↓	
         ↓ ↓	
          ↓	
private void init(ThreadGroup g, Runnable target, String name,
                      long stackSize) {
    init(g, target, name, stackSize, null, true);
}
		↓ ↓	↓	
         ↓ ↓	
          ↓	
private void init(ThreadGroup g, Runnable target, String name,
                      long stackSize, AccessControlContext acc,
                      boolean inheritThreadLocals) {
    //中间源码省略
    this.target = target;//①
}

/* What will be run. */
private Runnable target; //Thread类中的target属性

@Override
public void run() {
    if (target != null) { //②
        target.run();
    }
}

```

> 源码标记解读：
>
> 1、如果Thread类的构造方法传递了一个Runnable接口对象
>
> ①那么该接口对象将被Thread类中的target属性所保存。
>
> ②在start()方法执行的时候会调用Thread类中的run()方法。因为target不为null， target.run()就去调用实现Runnable接口的子类重写的run()。
>
> 2、如果Thread类的构造方传没传Runnable接口对象
>
> ①Thread类中的target属性保存的就是null。
>
> ②在start()方法执行的时候会调用Thread类中的run()方法。因为target为null，只能去调用继承Thread的子类所重写的run()。



JVM一旦启动，虚拟机栈的大小已经确定了。但是如果你创建Thread的时候传了stackSize（该线程占用的stack大小），该参数会被JNI函数去使用。如果没传这个参数，就默认为0，表示忽略这个参数。注：stackSize在有一些平台上是无效的。



# start()源码

```Java
public synchronized void start() {
   
    if (threadStatus != 0)
        throw new IllegalThreadStateException();//①

   
    group.add(this);

    boolean started = false;
    try {
        start0();
        started = true;
    } finally {
        try {
            if (!started) {
                group.threadStartFailed(this);
            }
        } catch (Throwable ignore) {
            /* do nothing. If start0 threw a Throwable then
              it will be passed up the call stack */
        }
    }
}

private native void start0();


@Override
public void run() {
    if (target != null) {
        target.run();
    }
}
```

> 源码标记解读：
>
> ①当多次调用start()，会抛出throw new IllegalThreadStateException()异常。也就是每一个线程类的对象只允许启动一次，如果重复启动则就抛出此异常。





## 为什么线程的启动不直接使用run()而必须使用start()呢?

1、如果直接调用run()方法，相当于就是简单的调用一个普通方法。

<img src="https://npm.elemecdn.com/youthlql@1.0.8/Java_concurrency/Source_code/First_stage/0004.png">

2、run()的调用是在start0()这个Native C++方法里调用的



# 线程生命周期

Java 线程在运行的生命周期中的指定时刻只可能处于下面 6 种不同状态的其中一个状态，这几个状态在Java源码中用枚举来表示。

<img src="https://npm.elemecdn.com/youthlql@1.0.8/Java_concurrency/Source_code/First_stage/0005.png">

线程在生命周期中并不是固定处于某一个状态而是随着代码的执行在不同状态之间切换。Java 线程状态变迁如下图所示。

<img src="https://npm.elemecdn.com/youthlql@1.0.8/Java_concurrency/Source_code/First_stage/0006.png">

>  图中 wait到 runnable状态的转换中，`join`实际上是`Thread`类的方法，但这里写成了`Object`。

1、由上图可以看出：线程创建之后它将处于 **NEW（新建）** 状态，调用 `start()` 方法后开始运行，线程这时候处于 **READY（可运行）** 状态。可运行状态的线程获得了 CPU 时间片（timeslice）后就处于 **RUNNING（运行）** 状态。

2、操作系统隐藏 Java 虚拟机（JVM）中的 READY 和 RUNNING 状态，它只能看到 RUNNABLE 状态，所以 Java 系统一般将这两个状态统称为 **RUNNABLE（运行中）** 状态 。

3、调用sleep()方法，会进入Blocked状态。sleep()结束之后，Blocked状态首先回到的是Runnable状态中的Ready（也就是可运行状态，但并未运行）。只有拿到了cpu的时间片才会进入Runnable中的Running状态。



# Thread常用API

* 获取当前存活的线程数：`public int activeCount()`
* 获取当前线程组的线程的集合：`public int enumerate(Thread[] list)`



# 一个Java程序有哪些线程？

1、当你调用一个线程start()方法的时候，此时至少有两个线程，一个是调用你的线程，还有一个是被你创建出来的线程。

例子：

```java
public static void main(String[] args) {
    Thread t1 = new Thread() {
        @Override
        public void run() {
            System.out.println("==========");
        }
    };
    t1.start();
}
```

这里面就是一个调用你的线程（main线程），一个被你创建出来的线程（t1，名字可能是Thread-0）

2、当JVM启动后，实际有多个线程，但是至少有一个非守护线程（比如main线程）。

<img src="https://npm.elemecdn.com/youthlql@1.0.8/Java_concurrency/Source_code/First_stage/0007.png">

- Finalizer：GC守护线程

- RMI：Java自带的远程方法调用（秋招面试，有个面试官问过）

- Monitor ：是一个守护线程，负责监听一些操作，也在main线程组中

- 其它：我用的是IDEA，其它的应该是IDEA的线程，比如鼠标监听啥的。







# 守护线程



```Java
public static void main(String[] args) throws InterruptedException {

    Thread t = new Thread() {

        @Override
        public void run() {
            try {
                System.out.println(Thread.currentThread().getName() + " running");
                Thread.sleep(100000);//①
                System.out.println(Thread.currentThread().getName() + " done.");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }; //new
    
    t.setDaemon(true);//②
	t.start();
    Thread.sleep(5_000);   //JDK1.7
    System.out.println(Thread.currentThread().getName());
}
```

> 源码标记解读：
>
> ①变量名为t的线程Thread-0，睡眠100秒。
>
> ②但是在主函数里Thread-0设置成了main线程的守护线程。所以5秒之后main线程结束了，即使在①这里守护线程还是处于睡眠100秒的状态，但由于他是守护线程，非守护线程main结束了，守护线程也必须结束。
>
> 1、但是如果Thread-0线程不是守护线程，即使main线程结束了，Thread-0线程仍然会睡眠100秒再结束。
>
> * 当主线程死亡后，守护线程会跟着死亡
> * 可以帮助做一些辅助性的东西，如“心跳检测”
> * 设置守护线程：`public final void setDaemon(boolean on)`

## 用处

A和B之间有一条网络连接，可以用守护线程来进行发送心跳，一旦A和B连接断开，非守护线程就结束了，守护线程（也就是心跳没有必要再发送了）也刚好断开。

```Java
public static void main(String[] args) {

    Thread t = new Thread(() -> {
        Thread innerThread = new Thread(() -> {
            try {
                while (true) {
                    System.out.println("Do some thing for health check.");
                    Thread.sleep(1_000);
                }
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });

      //  innerThread.setDaemon(true);
        innerThread.start();

        try {
            Thread.sleep(1_000);
            System.out.println("T thread finish done.");
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    });
    //t.setDaemon(true);
    t.start();
}

/*
设置该线程为守护线程必须在启动它之前。如果t.start()之后，再t.setDaemon(true);
会抛出IllegalThreadStateException
*/
```

> 输出结果：
>
> Do some thing for health check.
> Do some thing for health check.
> T thread finish done.  		//此时main线程已经结束，但是由于innerThread还在发送心跳，应用不会关闭
> Do some thing for health check.
> Do some thing for health check.
> Do some thing for health check.
> Do some thing for health check.



> 守护线程还有其它很多用处，在后面的文章里还会有出现。



# join方法



**例子1**

```java
public static void main(String[] args) throws InterruptedException {
    Thread t1 = new Thread(() -> {
        IntStream.range(1, 1000)
                .forEach(i -> System.out.println(Thread.currentThread().getName() + "->" + i));
    });
    Thread t2 = new Thread(() -> {
        IntStream.range(1, 1000)
                .forEach(i -> System.out.println(Thread.currentThread().getName() + "->" + i));
    });

    t1.start();
    t2.start();
    t1.join();
    t2.join();

    Optional.of("All of tasks finish done.").ifPresent(System.out::println);
    IntStream.range(1, 1000)
            .forEach(i -> System.out.println(Thread.currentThread().getName() + "->" + i));
}
```

* 默认传入的数字为0，这里是在main线程里调用了两个线程的join()，所以main线程会等到Thread-0和Thread-1线程执行完再执行它自己。
* join必须在start方法之后，并且join()是对wait()的封装。（源码中可以清楚的看到）

- 也就是说，t.join()方法阻塞调用此方法的线程(calling thread)进入 TIMED_WAITING或WAITING 状态。直到线程t完成，此线程再继续。
- join也有人理解成插队，比如在main线程中调用t.join()，就是t线程要插main线程的队，main线程要去等待。



**例子2**

```java
public static void main(String[] args) throws InterruptedException {
        Thread t1 = new Thread(() -> {
            IntStream.range(1, 1000)
                    .forEach(i -> System.out.println(Thread.currentThread().getName() + "->" + i));
        });
        Thread t2 = new Thread(() -> {
            try {
                t1.join();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            IntStream.range(1, 1000)
                    .forEach(i -> System.out.println(Thread.currentThread().getName() + "->" + i));
        });

        t1.start();
        t2.start();
//        t1.join();
        t2.join();

        Optional.of("All of tasks finish done.").ifPresent(System.out::println);
        IntStream.range(1, 1000)
                .forEach(i -> System.out.println(Thread.currentThread().getName() + "->" + i));
    }
```

- 这里是在t2（<span style="color:red;font-weight:bold">我们以后就都用变量名来称呼线程了</span>）线程了。t1.join()了。所以t2线程会等待t1线程打印完，t2自己才会打印。然后t2.join()，main线程也要等待t2线程。总体执行顺序就是t1-->t2-->main
- 通过上方例子可以用join实现类似于CompletableFuture的异步任务编排。（后面会讲）



# 中断

1、Java 中的中断和操作系统的中断还不一样，这里就按照**状态**来理解吧，不要和操作系统的中断联系在一起

2、记住中断只是一个状态，Java的方法可以选择对这个中断进行响应，也可以选择不响应。响应的意思就是写相对应的代码执行相对应的操作，不响应的意思就是什么代码都不写。

## 几个方法

```java
// Thread 类中的实例方法，持有线程实例引用即可检测线程中断状态
public boolean isInterrupted() {}

/*
1、Thread 中的静态方法，检测调用这个方法的线程是否已经中断
2、注意：这个方法返回中断状态的同时，会将此线程的中断状态重置为 false
如果我们连续调用两次这个方法的话，第二次的返回值肯定就是 false 了
*/
public static boolean interrupted() {}

// Thread 类中的实例方法，用于设置一个线程的中断状态为 true
public void interrupt() {}
```





## 小tip



```java
public static boolean interrupted()

public boolean isInterrupted()//这个会清除中断状态

```

为什么要这么设置呢？原因在于：

* interrupted()是一个静态方法，可以在Runnable接口实例中使用
* isInterrupted()是一个Thread的实例方法，在重写Thread的run方法时使用



```java
public class ThreadInterrupt {
    public static void main(String[] args) throws InterruptedException {
        Thread t1 = new Thread(() -> {
            System.out.println(Thread.interrupted());
        });  //这个new Thread用的是runnable接口那个构造函数

        Thread t2 = new Thread(){
            @Override
            public void run() {
                System.out.println(isInterrupted());
            }
        };//这个new Thread用的就是Thread的空参构造

    }
}

```

也就是说接口中不能调用Thread的实例方法，只能通过静态方法来判断是否发生中断



## 重难点

当然，中断除了是线程状态外，还有其他含义，否则也不需要专门搞一个这个概念出来了。

> 初学者肯定以为 thread.interrupt() 方法是用来暂停线程的，主要是和它对应中文翻译的“中断”有关。中断在并发中是常用的手段，请大家一定好好掌握。可以将中断理解为线程的状态，它的特殊之处在于设置了中断状态为 true 后，这几个方法会感知到：
>
> 1. wait(), wait(long), wait(long, int), join(), join(long), join(long, int), sleep(long), sleep(long, int)
>
>    这些方法都有一个共同之处，方法签名上都有`throws InterruptedException`，这个就是用来响应中断状态修改的。
>
> 2. 如果线程阻塞在 InterruptibleChannel 类的 IO 操作中，那么这个 channel 会被关闭。
>
> 3. 如果线程阻塞在一个 Selector 中，那么 select 方法会立即返回。
>
> 对于以上 3 种情况是最特殊的，因为他们能自动感知到中断（这里说自动，当然也是基于底层实现），**并且在做出相应的操作后都会重置中断状态为 false**。然后执行相应的操作（通常就是跳到 catch 异常处）。
>
> 如果不是以上3种情况，那么，线程的 interrupt() 方法被调用，会将线程的中断状态设置为 true。
>
> 那是不是只有以上 3 种方法能自动感知到中断呢？不是的，如果线程阻塞在 LockSupport.park(Object obj) 方法，也叫挂起，这个时候的中断也会导致线程唤醒，但是唤醒后不会重置中断状态，所以唤醒后去检测中断状态将是 true。

> 资料:  [Oracle官方文档](https://docs.oracle.com/javase/specs/index.html)    --->   [ The Java® Language Specification Java SE 8 Edition](https://docs.oracle.com/javase/specs/jls/se8/html/index.html)   --->   第17章 Threads and Locks



## InterruptedException

它是一个特殊的异常，不是说 JVM 对其有特殊的处理，而是它的使用场景比较特殊。通常，我们可以看到，像 Object 中的 wait() 方法，ReentrantLock 中的 lockInterruptibly() 方法，Thread 中的 sleep() 方法等等，这些方法都带有 `throws InterruptedException`，我们通常称这些方法为阻塞方法（blocking method）。

阻塞方法一个很明显的特征是，它们需要花费比较长的时间（不是绝对的，只是说明时间不可控），还有它们的方法结束返回往往依赖于外部条件，如 wait 方法依赖于其他线程的 notify，lock 方法依赖于其他线程的 unlock等等。

当我们看到方法上带有 `throws InterruptedException` 时，我们就要知道，这个方法应该是阻塞方法，我们如果希望它能早点返回的话，我们往往可以通过中断来实现。 

除了几个特殊类（如 Object，Thread等）外，感知中断并提前返回是通过轮询中断状态来实现的。我们自己需要写可中断的方法的时候，就是通过在合适的时机（通常在循环的开始处）去判断线程的中断状态，然后做相应的操作（通常是方法直接返回或者抛出异常）。当然，我们也要看到，如果我们一次循环花的时间比较长的话，那么就需要比较长的时间才能**感知**到线程中断了。




## wait()中断测试

```Java
public static void main(String[] args) {

    Thread t = new Thread() {
        @Override
        public void run() {
            while (true) {
                synchronized (MONITOR) {
                    try {
                        MONITOR.wait(10);
                    } catch (InterruptedException e) {
                        System.out.println("wait响应中断");//pos_1
                        e.printStackTrace();//pos_2
                        System.out.println(isInterrupted());//pos_3
                    }
                }
            }
        }
    };

    t.start();
    try {
        Thread.sleep(100);
    } catch (InterruptedException e) {
        System.out.println("sleep响应中断");
        e.printStackTrace();
    }
    System.out.println(t.isInterrupted());//pos_4
    t.interrupt();
    System.out.println(t.isInterrupted());//pos_5
}
```

> 注释掉e.printStackTrace();的输出
>
> false						//pos_4
> true						 //pos_5	   
> wait响应中断			//pos_1
> false						 //pos_3		



## join中断测试

```java
Thread main = Thread.currentThread();
Thread t2 = new Thread() {
    @Override
    public void run() {
        try {
            Thread.sleep(100);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        main.interrupt();  //pos_1
        System.out.println("interrupt");
    }
};

t2.start();
try {
    t.join();  //pos_2
} catch (InterruptedException e) {
    e.printStackTrace();
}
```

1、pos_2这里join的是main线程，所以pos_1这里需要中断main线程，才能收到中断信息。



# 关闭线程

## 优雅的关闭(通过一个Boolean)

```Java
private static class Worker extends Thread {
    private volatile boolean start = true;

    @Override
    public void run() {
        while (start) {
           //执行相应的工作
        }
    }

    public void shutdown() {
        this.start = false;
    }
}

public static void main(String[] args) {
    Worker worker = new Worker();
    worker.start();

    try {
        Thread.sleep(10000);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }

    worker.shutdown();
}
```





## 通过判断中断状态

```java
private static class Worker extends Thread {

    @Override
    public void run() {
        while (true) {
            if (Thread.interrupted()){
                break;
            }
            //pos_1    
        }
    
    }
}

public static void main(String[] args) {
    Worker worker = new Worker();
    worker.start();

    try {
        Thread.sleep(3000);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }

    worker.interrupt();
}
```

1、但是如果pos_1位置有一个很费时的IO操作，就没有机会执行到if判断那里，也就不能关闭线程。所以就需要下面的暴力方法



## 暴力关闭（守护线程）

```java
public class ThreadService {

    //执行线程
    private Thread executeThread;

    private boolean finished = false;

    public void execute(Runnable task) {
        executeThread = new Thread() {
            @Override
            public void run() {
                Thread runner = new Thread(task);
                runner.setDaemon(true);//创建一个守护线程，让守护线程来执行工作

                runner.start();
                try {
/**
 * 1、要让executeThread等守护线程执行完，才能执行executeThread自己的逻辑。不然守护线程
 * 可能就来不及执行真正的工作就死了。所以这里要join
 * 2、runner.join()，所以实际上等待的是executeThread       //pos_1
 */
                    runner.join();
                    finished = true;
                } catch (InterruptedException e) {
                    //e.printStackTrace();
                }
            }
        };

        executeThread.start();
    }

    public void shutdown(long mills) {
        long currentTime = System.currentTimeMillis();
        while (!finished) {
            if ((System.currentTimeMillis() - currentTime) >= mills) {
                System.out.println("任务超时，需要结束他!");
                /*
                 * pos_1那里，由于实际等待的是executeThread，所以这里中断executeThread。
                 * pos_1就可以捕获到中断，执行线程(executeThread)就结束了，进而真正执行任务的
                 * 守护线程runner也结束了
                 */
                executeThread.interrupt();
                break;
            }

            try {
                executeThread.sleep(1);
            } catch (InterruptedException e) {
                System.out.println("执行线程被打断!");
                break;
            }
        }

        finished = false;
    }
}


public class ThreadCloseForce {


    public static void main(String[] args) {

        ThreadService service = new ThreadService();
        long start = System.currentTimeMillis();
        service.execute(() -> {
            //load a very heavy resource. 模拟任务超时
            /*while (true) {
			
            }*/
            try {
                Thread.sleep(5000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });
        service.shutdown(10000);
        long end = System.currentTimeMillis();
        System.out.println(end - start);
    }
}
```

**使用场景**：分布式文件拷贝，如果拷贝的时间过长，则关闭该程序，防止程序一直阻塞。或者其他执行耗时很长的任务

守护线程的应用场景有很多



# 并发编程中的三个问题

## 可见性



### 可见性概念

可见性（Visibility）：是指一个线程对共享变量进行修改，另一个先立即得到修改后的新值。

### 可见性演示

```java
/* 笔记
 * 1.当没有加Volatile的时候,while循环会一直在里面循环转圈
 * 2.当加了之后Volatile,由于可见性,一旦num改了之后,就会通知其他线程
 * 3.还有注意的时候不能用if,if不会重新拉回来再判断一次。(也叫做虚假唤醒)
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





# volatile

> 1、关于可见性，重排序等等的硬件原理，MESI缓存一致性，内存屏障，JMM等等这些，请看我的后面文章。第一阶段只是介绍下用法，不涉及原理。
>
> 2、如果你在第一篇文章没有找到你想要的内容，请看我后面的内容。并发的体系，我自认为讲的还是比较全面的。

## volatile保证可见性代码

> 读者可以把两个代码运行一下，就能明显看到不加volatile的死循环（就是程序一直显示没结束）

```java
/* 笔记
 * 1.当没有加Volatile的时候,while循环会一直在里面转圈
 * 2.当加了之后Volatile,由于可见性,一旦num改了之后,就会通知其他线程
 * 3.还有注意的时候不能用if,if不会重新拉回来再判断一次
 * */
public class Video04_02 {

    public static void main(String[] args) {
        MyData2 myData = new MyData2();

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

class MyData2{

    volatile int num = 0;

    public void addTo60(){
        this.num = 60;
    }
}
```

## volatile保证有序性代码

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

    volatile int num = 0;
    volatile boolean ready = false;
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

读者可以将运行结果对比着来看，就能发现区别。

volatile只能保证可见性和有序性（禁止指令重排），无法保证原子性。



# CAS

volatile自己虽然不能保证原子性，但是和CAS结合起来就可以保证原子性了。CAS+volatile一起用就可以同时解决**并发编程中的三个问题**了，保证并发安全。

## CAS 是什么？

- CAS：比较并交换compareAndSet,它是一条 CPU 并发原语，它的功能是判断内存某个位置的值是否为预期值，如果是则更改为新的值，这个过程是原子性的。
  
- 例: AtomicInteger 的 compareAndSet('期望值','设置值') 方法，期望值与目标值一致时，修改目标变量为设置值，期望值与目标值不一致时，返回 false 和最新主存的变量值
  
- CAS 的底层原理
  	例: AtomicInteger.getAndIncrement()
    	调用 Unsafe 类中的 CAS 方法，JVM 会帮我们实现出 CAS 汇编指令
    		这是一种完全依赖于硬件的功能，通过它实现原子操作。
    		原语的执行必须是连续的，在执行过程中不允许被中断，CAS 是 CPU 的一条原子指令。

- CAS的思想就是乐观锁的思想

## AtomicInteger

在JUC并发包中，CAS和AtomicInteger（原子类的value值都被volatile修饰了）一起保证了并发安全。下面我们以AtomicInteger.getAndIncrement() 方法讲一下。



<img src="https://npm.elemecdn.com/youthlql@1.0.8/Java_concurrency/Source_code/First_stage/0009.png">



```java
/**
 * unsafe: rt.jar/sun/misc/Unsafe.class
 *   Unsafe 是 CAS 的核心类，由于 Java 无法直接访问底层系统，需要通过本地<native>方法来访问
 *	 Unsafe 相当于一个后门，基于该类可以直接操作特定内存的数据
 *	 Unsafe 其内部方法都是 native 修饰的，可以像 C 的指针一样直接操作内存
 *	 Java 中的 CAS 操作执行依赖于 Unsafe 的方法，直接调用操作系统底层资源执行程序
 *
 * this: 当前对象
 *	 变量 value 由 volatile 修饰，保证了多线程之间的内存可见性、禁止重排序
 *
 * valueOffset: 内存地址
 *	 表示该变量值在内存中的偏移地址，因为 Unsafe 就是根据内存偏移地址获取数据
 *
 * 1: 固定写死，原值加1
 */
public final int getAndIncrement(){
    return unsafe.getAndAddInt(this,valueOffset,1);
}

/**
 * Unsafe.getAndAddInt()
 * getIntVolatile: 通过内存地址去主存中取对应数据
 * 
 * while(!this.compareAndSwapInt(var1,var2,var5,var5 + var4)):
 * 	 将本地 value 与主存中取出的数据对比，如果相同，对其作运算，
 * 		此时返回 true，取反后 while 结束，返回最终值。
 * 	 如果不相同，此时返回 false，取反后 while 循环继续运行，此时为自旋锁<重复尝试>
 *		由于 value 是被 volatile 修饰的，所以拿到主存中最新值，再循环直至成功。
 */
public final int getAndAddInt(Object var1,long var2,int var4){
    int var5;
    do{
        var5 = this.getIntVolatile(var1,var2); // 从主存中拷贝变量到本地内存
    } while(!this.compareAndSwapInt(var1,var2,var5,var5 + var4));
    return var5;
}
```



## CAS 代码演示

```Java
public class CASDemo {

    public static void main(String[] args) {
        AtomicInteger num = new AtomicInteger(5);
        // TODO
        System.out.println(num.compareAndSet(5, 1024) + "\t current num" + num.get());
        System.out.println(num.compareAndSet(5, 2019) + "\t current num" + num.get());
    }
```



## CAS三大问题

- 如果 CAS 长时间一直不成功，会给 CPU 带来很大的开销，在Java的实现中是一直通过while循环自旋CAS获取锁。

- 只能保证一个共享变量的原子操作

- 引出了 ABA 问题



## ABA问题



### 什么是ABA问题？

```java
/**
 * @Author: 吕
 * @Date: 2019/9/24 16:43
 * <p>
 * 功能描述: CAS引发的ABA问题
 */
public class Video19_01 {
    static AtomicReference<Integer> num = new AtomicReference<>(100);

    public static void main(String[] args) {
	
        new Thread(() ->{
            num.compareAndSet(100, 101);
            num.compareAndSet(101,100);
        },"线程A").start();

        new Thread(() ->{
            //保证A线程已经修改完
            try {
                TimeUnit.SECONDS.sleep(1);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            boolean b = num.compareAndSet(100, 2019);
            System.out.println(b + "\t 当前最新值" + num.get().toString());
        },"线程B").start();
    }
}
```

**CAS 会导致 ABA 问题：**

例: A、B线程从主存取出变量 value

-> A 在 N次计算中改变 value 的值
-> A 最终计算结果还原 value 最初的值
-> B 计算后，比较主存值与自身 value 值一致，修改成功

尽管各个线程的 CAS 都操作成功，但是并不代表这个过程就是没有问题的。



### ABA问题的解决

```java
/**
 * @Author: 吕
 * @Date: 2019/9/24 16:49
 * <p>
 * 功能描述: ABA问题的解决
 */
public class Video19_02 {
    static AtomicStampedReference<Integer> num = new AtomicStampedReference<>(100,1);

    public static void main(String[] args) {
        int stamp = num.getStamp();//初始版本号

        new Thread(() ->{
            num.compareAndSet(100,101,num.getStamp(),num.getStamp() + 1);
            System.out.println(Thread.currentThread().getName() + "\t 版本号" + num.getStamp());
            num.compareAndSet(101,100,num.getStamp(),num.getStamp() + 1);
            System.out.println(Thread.currentThread().getName() + "\t 版本号" + num.getStamp());
        },"线程A").start();


        new Thread(() ->{
            try {
                TimeUnit.SECONDS.sleep(3);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            boolean b = num.compareAndSet(100, 209, stamp, num.getStamp() + 1);
            System.out.println(b + "\t 当前版本号: \t" + num.getStamp());
            System.out.println("当前最新值 \t" + num.getReference().toString());
        },"线程B").start();
    }
}
```

思想很简单，可以很明显的看出来用版本号的方式解决了ABA的问题。

* 除了对象值，AtomicStampedReference内部还维护了一个“状态戳”。
* 状态戳可类比为时间戳，是一个整数值，每一次修改对象值的同时，也要修改状态戳，从而区分相同对象值的不同状态。
* 当AtomicStampedReference设置对象值时，对象值以及状态戳都必须满足期望值，写入才会成功。

## 只能保证一个共享变量的原子操作

- 当对一个共享变量执行操作时，我们可以使用循环CAS的方式来保证原子操作，但是多个共享变量操作时，循环CAS就无法保证操作的原子性，这个时候就可以用锁。还有一个方法，就是把多个共享变量合并成一个共享变量来操作。比如，有两个共享变量i=2,j=a合并一下ij=2a，然后用CAS来操作ij。从java1.5开始，JDK提供了AtomicReference类来保证引用对象之间的原子性，就可以把多个变量放在一个对象里来进行CAS操作。
- 所以一般来说为了同时解决ABA问题和只能保证一个共享变量，原子类使用时大部分使用的是`AtomicStampedReference`



# UnSafe

Unsafe类是在sun.misc包下，不属于Java标准。但是很多Java的基础类库，包括一些被广泛使用的高性能开发库都是基于Unsafe类开发的，比如Netty、Cassandra、Hadoop、Kafka等。Unsafe类在提升Java运行效率，增强Java语言底层操作能力方面起了很大的作用。

Java和C++语言的一个重要区别就是Java中我们无法直接操作一块内存区域，不能像C++中那样可以自己申请内存和释放内存。 **Java中的Unsafe类为我们提供了类似C++手动管理内存的能力，同时也有了指针的问题。**

首先，Unsafe类是"final"的，不允许继承。且构造函数是private的:

```java
public final class Unsafe {
    private static final Unsafe theUnsafe;
    public static final int INVALID_FIELD_OFFSET = -1;

    private static native void registerNatives();

    private Unsafe() {
    }
    ...

}
```

**因此我们无法在外部对Unsafe进行实例化。**

## 获取Unsafe

Unsafe无法实例化，那么怎么获取Unsafe呢？答案就是通过反射来获取Unsafe：

```java
public Unsafe getUnsafe() throws IllegalAccessException {
    Field unsafeField = Unsafe.class.getDeclaredFields()[0];
    unsafeField.setAccessible(true);
    Unsafe unsafe = (Unsafe) unsafeField.get(null);
    return unsafe;
}
```

Unsafe的功能如下图：

<img src="https://npm.elemecdn.com/youthlql@1.0.8/Java_concurrency/Source_code/First_stage/0008.png">

## CAS相关

JUC中大量运用了CAS操作，可以说CAS操作是JUC的基础，因此CAS操作是非常重要的。Unsafe中提供了int,long和Object的CAS操作：

```java
public final native boolean compareAndSwapObject(Object var1, long var2, Object var4, Object var5);

public final native boolean compareAndSwapInt(Object var1, long var2, int var4, int var5);

public final native boolean compareAndSwapLong(Object var1, long var2, long var4, long var6);
```

## 偏移量相关

```java
public native long staticFieldOffset(Field var1);

public native long objectFieldOffset(Field var1);

```

* staticFieldOffset方法用于获取静态属性Field在对象中的偏移量，读写静态属性时必须获取其偏移量。
* objectFieldOffset方法用于获取非静态属性Field在对象实例中的偏移量，读写对象的非静态属性时会用到这个偏移量

## 类加载

```java
public native Class<?> defineClass(String var1, byte[] var2, int var3, int var4, ClassLoader var5, ProtectionDomain var6);

public native Class<?> defineAnonymousClass(Class<?> var1, byte[] var2, Object[] var3);

public native Object allocateInstance(Class<?> var1) throws InstantiationException;

public native boolean shouldBeInitialized(Class<?> var1);

public native void ensureClassInitialized(Class<?> var1);
```

* defineClass方法定义一个类，用于动态地创建类。
* defineAnonymousClass用于动态的创建一个匿名内部类。
* `allocateInstance`方法用于创建一个类的实例，但是不会调用这个实例的构造方法，如果这个类还未被初始化，则初始化这个类。
* shouldBeInitialized方法用于判断是否需要初始化一个类。
* ensureClassInitialized方法用于保证已经初始化过一个类。

**举例**

```java
public class UnsafeFooTest {
    private static Unsafe geUnsafe() {

        try {
            Field f = Unsafe.class.getDeclaredField("theUnsafe");
            f.setAccessible(true);
            return (Unsafe) f.get(null);
        } catch (NoSuchFieldException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        }
        return null;

    }

    static class Simple {
        private long l = 0;

        public Simple() {
            this.l = 1;
            System.out.println("我被初始化了");
        }

        public long getL() {
            return l;
        }
    }

    public static void main(String[] args) throws Exception {

        Unsafe unsafe = geUnsafe();

        Simple s = (Simple) unsafe.allocateInstance(Simple.class);
        System.out.println(s.getL());
    }
}
```

**结果：**

> 0

* 可以发现，利用Unsafe获取实例，不会调用构造方法

## 普通读写

通过Unsafe可以读写一个类的属性，即使这个属性是私有的，也可以对这个属性进行读写。

读写一个Object属性的相关方法

```java
public native int getInt(Object var1, long var2);

public native void putInt(Object var1, long var2, int var4);
```

* getInt用于从对象的指定偏移地址处读取一个int。
* putInt用于在对象指定偏移地址处写入一个int。其他的primitive type也有对应的方法。

**举例**

```java
public class UnsafeFooTest {
    private static Unsafe geUnsafe() {

        try {
            Field f = Unsafe.class.getDeclaredField("theUnsafe");
            f.setAccessible(true);
            return (Unsafe) f.get(null);
        } catch (NoSuchFieldException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        }
        return null;

    }

    static class Guard{
        private int ACCESS_ALLOWED = 1;
        private boolean allow(){
            return 50 == ACCESS_ALLOWED;
        }

        public void work(){
            if (allow()){
                System.out.println("我被允许工作....");
            }
        }
    }

    public static void main(String[] args) throws Exception {
        Unsafe unsafe = geUnsafe();
        Guard guard = new Guard();

        Field f = guard.getClass().getDeclaredField("ACCESS_ALLOWED");

        unsafe.putInt(guard,unsafe.objectFieldOffset(f),50);
        System.out.println("强行赋值...");
        guard.work();

    }
}

```

**结果**

> 强行赋值...

我被允许工作...

## 类加载

```java
public native Class<?> defineClass(String var1, byte[] var2, int var3, int var4, ClassLoader var5, ProtectionDomain var6);

public native Class<?> defineAnonymousClass(Class<?> var1, byte[] var2, Object[] var3);

public native Object allocateInstance(Class<?> var1) throws InstantiationException;

public native boolean shouldBeInitialized(Class<?> var1);

public native void ensureClassInitialized(Class<?> var1);
```

* defineClass方法定义一个类，用于动态地创建类。
* defineAnonymousClass用于动态的创建一个匿名内部类。
* allocateInstance方法用于创建一个类的实例，但是不会调用这个实例的构造方法，如果这个类还未被初始化，则初始化这个类。
* shouldBeInitialized方法用于判断是否需要初始化一个类。
* ensureClassInitialized方法用于保证已经初始化过一个类。

## 内存屏障

```java
public native void loadFence();

public native void storeFence();

public native void fullFence();
```

* loadFence：保证在这个屏障之前的所有读操作都已经完成。
* storeFence：保证在这个屏障之前的所有写操作都已经完成。
* fullFence：保证在这个屏障之前的所有读写操作都已经完成。

## 线程调度

```java
public native void unpark(Object var1);

public native void park(boolean var1, long var2);

public native void monitorEnter(Object var1);

public native void monitorExit(Object var1);

public native boolean tryMonitorEnter(Object var1);
```

* park方法和unpark方法相信看过LockSupport类的都不会陌生，这两个方法主要用来挂起和唤醒线程。
* LockSupport中的park和unpark方法正是通过Unsafe来实现的：

```java
public static void park(Object blocker) {
    Thread t = Thread.currentThread();
    setBlocker(t, blocker);
    UNSAFE.park(false, 0L);
    setBlocker(t, null);
}

public static void unpark(Thread thread) {
    if (thread != null)
        UNSAFE.unpark(thread);
}
```

**monitorEnter方法和monitorExit方法用于加锁，Java中的synchronized锁就是通过这两个指令来实现的。**

# synchronized优化

> synchronized可以同时保证可见性，有序性，原子性。这个东西就不讲了

从JDk 1.6开始，JVM就对synchronized锁进行了很多的优化。synchronized说是锁，但是他的底层加锁的方式可能不同，偏向锁的方式来加锁，自旋锁的方式来加锁，轻量级锁的方式来加锁



## 锁消除

锁消除是JIT编译器对synchronized锁做的优化，在编译的时候，JIT会通过逃逸分析技术，来分析synchronized锁对象，是不是只可能被一个线程来加锁，没有其他的线程来竞争加锁，这个时候编译就不用加入monitorenter和monitorexit的指令。这就是，仅仅一个线程争用锁的时候，就可以消除这个锁了，提升这段代码的执行的效率，因为可能就只有一个线程会来加锁，不涉及到多个线程竞争锁

 

## 锁粗化 



```java
synchronized(this) {

}

 

synchronized(this) { 

}

 

synchronized(this) {

}

```

这个意思就是，JIT编译器如果发现有代码里连续多次加锁释放锁的代码，会给合并为一个锁，就是锁粗化，把一个锁给搞粗了，避免频繁多次加锁释放锁

## 偏向锁

这个意思就是说，monitorenter和monitorexit是要使用CAS操作加锁和释放锁的，开销较大，因此如果发现大概率只有一个线程会主要竞争一个锁，那么会给这个锁维护一个偏好（Bias），后面他加锁和释放锁，基于Bias来执行，不需要通过CAS，性能会提升很多。但是如果有偏好之外的线程来竞争锁，就要收回之前分配的偏好。可能只有一个线程会来竞争一个锁，但是也有可能会有其他的线程来竞争这个锁，但是其他线程唉竞争锁的概率很小。如果有其他的线程来竞争这个锁，此时就会收回之前那个线程分配的那个Bias偏好

 

## 轻量级锁

如果偏向锁没能成功实现，就是因为不同线程竞争锁太频繁了，此时就会尝试采用轻量级锁的方式来加锁，就是将对象头的Mark Word里有一个轻量级锁指针，尝试指向持有锁的线程，然后判断一下是不是自己加的锁，如果是自己加的锁，那就执行代码就好了。如果不是自己加的锁，那就是加锁失败，说明有其他人加了锁，这个时候就是升级为重量级锁

 

## 适应性锁

这是JIT编译器对锁做的另外一个优化，如果各个线程持有锁的时间很短，那么一个线程竞争锁不到，就会暂停，发生上下文切换，让其他线程来执行。但是其他线程很快释放锁了，然后暂停的线程再次被唤醒。也就是说在这种情况下，线程会频繁的上下文切换，导致开销过大。所以对这种线程持有锁时间很短的情况，是可以采取忙等策略的，也就是一个线程没竞争到锁，进入一个while循环不停等待，不会暂停不会发生线程上下文切换，等到机会获取锁就继续执行好了

 

 






# 参考：

- https://blog.csdn.net/qq_43040688/article/details/103979628
- 《Java 并发编程艺术》
- B站汪文君系列并发



# 强调

1、第一阶段只是简单的讲一下，在后面的系列里，会从硬件，C++源码层面讲解volatile，synchronized，内存屏障，MESI-缓存一致性等等进行讲解。

2、还有一些问题，在基础阶段可能不太好讲。比如中断这个东西，可能理解的云里雾里的，后面的系列讲到AQS的时候，结合Java源码再讲的话，你会非常好理解。
