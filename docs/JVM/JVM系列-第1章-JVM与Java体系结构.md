---
title: JVM系列-第1章-JVM与Java体系结构
tags:
  - JVM
  - 虚拟机
categories:
  - JVM
  - 1.内存与垃圾回收篇
keywords: JVM，虚拟机。
description: JVM系列-第1章-JVM与Java体系结构。
cover: 'https://npm.elemecdn.com/lql_static@latest/logo/jvm.png'
abbrlink: 8c954c6
date: 2020-11-02 11:51:56
---





> 1、本系列博客，主要是面向Java8的虚拟机。如有特殊说明，会进行标注。
>
> 2、本系列博客主要参考**尚硅谷的JVM视频教程**，整理不易，所以图片打上了一些水印，还请读者见谅。后续可能会加上一些补充的东西。
>
> 3、尚硅谷的有些视频还不错（PS：不是广告，毕竟看了人家比较好的教程，得给人家打个call）
>
> 4、转载请注明出处，多谢~，希望大家一起能维护一个良好的开源环境。

第1章-JVM和Java体系架构
=====================

前言
--------

你是否也遇到过这些问题？

1.  运行着的线上系统突然卡死，系统无法访问，甚至直接OOM！
2.  想解决线上JVM GC问题，但却无从下手。
3.  新项目上线，对各种JVM参数设置一脸茫然，直接默认吧然后就JJ了。
4.  每次面试之前都要重新背一遍JVM的一些原理概念性的东西，然而面试官却经常问你在实际项目中如何调优VM参数，如何解决GC、OOM等问题，一脸懵逼。

<img src="https://npm.elemecdn.com/youthlql@1.0.8/JVM/chapter_001/0001.png">

大部分Java开发人员，除了会在项目中使用到与Java平台相关的各种高精尖技术，对于Java技术的核心Java虚拟机了解甚少。

开发人员如何看待上层框架
---------



1.  一些有一定工作经验的开发人员，打心眼儿里觉得SSM、微服务等上层技术才是重点，基础技术并不重要，这其实是一种本末倒置的“病态”。
2.  如果我们把核心类库的API比做数学公式的话，那么Java虚拟机的知识就好比公式的推导过程。

<img src="https://npm.elemecdn.com/youthlql@1.0.8/JVM/chapter_001/0002.png">

- 计算机系统体系对我们来说越来越远，在不了解底层实现方式的前提下，通过高级语言很容易编写程序代码。但事实上计算机并不认识高级语言。



架构师每天都在思考什么？
---------

1.  应该如何让我的系统更快？
2.  如何避免系统出现瓶颈？

**知乎上有条帖子：应该如何看招聘信息，直通年薪50万+？**

1.  参与现有系统的性能优化，重构，保证平台性能和稳定性
2.  根据业务场景和需求，决定技术方向，做技术选型
3.  能够独立架构和设计海量数据下高并发分布式解决方案，满足功能和非功能需求
4.  解决各类潜在系统风险，核心功能的架构与代码编写
5.  分析系统瓶颈，解决各种疑难杂症，性能调优等

我们为什么要学习JVM
-----------

1.  面试的需要（BATJ、TMD，PKQ等面试都爱问）
2.  中高级程序员必备技能
  - 项目管理、调优的需要
3.  追求极客的精神，
    - 比如：垃圾回收算法、JIT、底层原理



Java VS C++
-------------

1.  垃圾收集机制为我们打理了很多繁琐的工作，大大提高了开发的效率，但是，垃圾收集也不是万能的，懂得JVM内部的内存结构、工作机制，是设计高扩展性应用和诊断运行时问题的基础，也是Java工程师进阶的必备能力。
2.  C++语言需要程序员自己来分配内存和回收内存，对于高手来说可能更加舒服，但是对于普通开发者，如果技术实力不够，很容易造成内存泄漏。而Java全部交给JVM进行内存分配和回收，这也是一种趋势，减少程序员的工作量。

<img src="https://npm.elemecdn.com/youthlql@1.0.8/JVM/chapter_001/0003.png">

## 什么人需要学JVM？

1. 拥有一定开发经验的Java开发人员，希望升职加薪
2. 软件设计师，架构师
3. 系统调优人员
4. 虚拟机爱好者，JVM实践者

推荐及参考书籍
------

**官方文档**

**英文文档规范**：https://docs.oracle.com/javase/specs/index.html

<img src="https://npm.elemecdn.com/youthlql@1.0.8/JVM/chapter_001/0004.png">

**中文书籍：**

<img src="https://npm.elemecdn.com/youthlql@1.0.8/JVM/chapter_001/0005.png">

> 周志明老师的这本书**非常推荐看**，不过只推荐看第三版，第三版较第二版更新了很多，个人觉得没必要再看第二版。

<img src="https://npm.elemecdn.com/youthlql@1.0.8/JVM/chapter_001/0006.png">



<img src="https://npm.elemecdn.com/youthlql@1.0.8/JVM/chapter_001/0007.png">

TIOBE排行榜
-----------

**TIOBE 排行榜**：https://www.tiobe.com/tiobe-index/

<img src="https://npm.elemecdn.com/youthlql@1.0.8/JVM/chapter_001/0008.png">

- 世界上没有最好的编程语言，只有最适用于具体应用场景的编程语言。
- 目前网上一直流传Java被python，go撼动Java第一的地位。学习者不需要太担心，Java强大的生态圈，也不是说是朝夕之间可以被撼动的。



Java生态圈
----------

Java是目前应用最为广泛的软件开发平台之一。随着Java以及Java社区的不断壮大Java 也早已不再是简简单单的一门计算机语言了，它更是一个平台、一种文化、一个社区。

1.  作为一个平台，Java虚拟机扮演着举足轻重的作用
    *   Groovy、Scala、JRuby、Kotlin等都是Java平台的一部分
2.  作为一种文化，Java几乎成为了“开源”的代名词。
    *   第三方开源软件和框架。如Tomcat、Struts，MyBatis，Spring等。
    *   就连JDK和JVM自身也有不少开源的实现，如openJDK、Harmony。
3.  作为一个社区，Java拥有全世界最多的技术拥护者和开源社区支持，有数不清的论坛和资料。从桌面应用软件、嵌入式开发到企业级应用、后台服务器、中间件，都可以看到Java的身影。其应用形式之复杂、参与人数之众多也令人咋舌。



Java-跨平台的语言
------------



<img src="https://npm.elemecdn.com/youthlql@1.0.8/JVM/chapter_001/0009.png">

JVM-跨语言的平台
------



<img src="https://npm.elemecdn.com/youthlql@1.0.8/JVM/chapter_001/0010.png">



1.  随着Java7的正式发布，Java虚拟机的设计者们通过JSR-292规范基本实现在Java虚拟机平台上运行非Java语言编写的程序。
2.  Java虚拟机根本不关心运行在其内部的程序到底是使用何种编程语言编写的，它只关心“字节码”文件。也就是说Java虚拟机拥有语言无关性，并不会单纯地与Java语言“终身绑定”，只要其他编程语言的编译结果满足并包含Java虚拟机的内部指令集、符号表以及其他的辅助信息，它就是一个有效的字节码文件，就能够被虚拟机所识别并装载运行。

- Java不是最强大的语言，但是JVM是最强大的虚拟机

1.  我们平时说的java字节码，指的是用java语言编译成的字节码。准确的说任何能在jvm平台上执行的字节码格式都是一样的。所以应该统称为：jvm字节码。

2.  不同的编译器，可以编译出相同的字节码文件，字节码文件也可以在不同的JVM上运行。

3.  Java虚拟机与Java语言并没有必然的联系，它只与特定的二进制文件格式——Class文件格式所关联，Class文件中包含了Java虚拟机指令集（或者称为字节码、Bytecodes）和符号表，还有一些其他辅助信息。

多语言混合编程
----------

1.  Java平台上的多语言混合编程正成为主流，通过特定领域的语言去解决特定领域的问题是当前软件开发应对日趋复杂的项目需求的一个方向。
  
2.  试想一下，在一个项目之中，并行处理用Clojure语言编写，展示层使用JRuby/Rails，中间层则是Java，每个应用层都将使用不同的编程语言来完成，而且，接口对每一层的开发者都是透明的，各种语言之间的交互不存在任何困难，就像使用自己语言的原生API一样方便，因为它们最终都运行在一个虚拟机之上。
  
3.  对这些运行于Java虚拟机之上、Java之外的语言，来自系统级的、底层的支持正在迅速增强，以JSR-292为核心的一系列项目和功能改进（如DaVinci Machine项目、Nashorn引擎、InvokeDynamic指令、java.lang.invoke包等），推动Java虚拟机从“Java语言的虚拟机”向 “多语言虚拟机”的方向发展。
  

如何真正搞懂JVM？
-----------

1.  Java虚拟机非常复杂，要想真正理解它的工作原理，最好的方式就是自己动手编写一个！
2.  自己动手写一个Java虚拟机，难吗？
3.  天下事有难易乎？为之，则难者亦易矣；不为，则易者亦难矣

<img src="https://npm.elemecdn.com/youthlql@1.0.8/JVM/chapter_001/0011.png">

Java发展重大事件
------------

*   1990年，在Sun计算机公司中，由Patrick Naughton、MikeSheridan及James Gosling领导的小组Green Team，开发出的新的程序语言，命名为Oak，后期命名为Java
*   1995年，Sun正式发布Java和HotJava产品，Java首次公开亮相。
*   1996年1月23日Sun Microsystems发布了JDK 1.0。
*   1998年，JDK1.2版本发布。同时，Sun发布了JSP/Servlet、EJB规范，以及将Java分成了J2EE、J2SE和J2ME。这表明了Java开始向企业、桌面应用和移动设备应用3大领域挺进。
*   2000年，JDK1.3发布，Java HotSpot Virtual Machine正式发布，成为Java的默认虚拟机。
*   2002年，JDK1.4发布，古老的Classic虚拟机退出历史舞台。
*   2003年年底，Java平台的scala正式发布，同年Groovy也加入了Java阵营。
*   2004年，JDK1.5发布。同时JDK1.5改名为JavaSE5.0。
*   2006年，JDK6发布。同年，Java开源并建立了OpenJDK。顺理成章，Hotspot虚拟机也成为了OpenJDK中的默认虚拟机。
*   2007年，Java平台迎来了新伙伴Clojure。
*   2008年，oracle收购了BEA，得到了JRockit虚拟机。
*   2009年，Twitter宣布把后台大部分程序从Ruby迁移到Scala，这是Java平台的又一次大规模应用。
*   2010年，Oracle收购了Sun，获得Java商标和最真价值的HotSpot虚拟机。此时，Oracle拥有市场占用率最高的两款虚拟机HotSpot和JRockit，并计划在未来对它们进行整合：HotRockit。JCP组织管理Java语言
*   2011年，JDK7发布。在JDK1.7u4中，正式启用了新的垃圾回收器G1。
*   **2017年，JDK9发布。将G1设置为默认GC，替代CMS**
*   同年，IBM的J9开源，形成了现在的Open J9社区
*   2018年，Android的Java侵权案判决，Google赔偿Oracle计88亿美元
*   同年，Oracle宣告JavagE成为历史名词JDBC、JMS、Servlet赠予Eclipse基金会
*   **同年，JDK11发布，LTS版本的JDK，发布革命性的ZGC，调整JDK授权许可**
*   2019年，JDK12发布，加入RedHat领导开发的Shenandoah GC

## Open JDK和Oracle JDK

<img src="https://npm.elemecdn.com/youthlql@1.0.8/JVM/chapter_001/0012.png">

- 在JDK11之前，Oracle JDK中还会存在一些Open JDK中没有的，闭源的功能。但在JDK11中，我们可以认为Open JDK和Oracle JDK代码实质上已经达到完全一致的程度了。
- 主要的区别就是两者更新周期不一样

虚拟机
--------

### 虚拟机概念

- 所谓虚拟机（Virtual Machine），就是一台虚拟的计算机。它是一款软件，用来执行一系列虚拟计算机指令。大体上，虚拟机可以分为系统虚拟机和程序虚拟机。

  - 大名鼎鼎的Virtual Box，VMware就属于系统虚拟机，它们完全是对物理计算机硬件的仿真(模拟)，提供了一个可运行完整操作系统的软件平台。

  + 程序虚拟机的典型代表就是Java虚拟机，它专门为执行单个计算机程序而设计，在Java虚拟机中执行的指令我们称为Java字节码指令。

- 无论是系统虚拟机还是程序虚拟机，在上面运行的软件都被限制于虚拟机提供的资源中。

### Java虚拟机

1.  Java虚拟机是一台执行Java字节码的虚拟计算机，它拥有独立的运行机制，其运行的Java字节码也未必由Java语言编译而成。
2.  JVM平台的各种语言可以共享Java虚拟机带来的跨平台性、优秀的垃圾回器，以及可靠的即时编译器。
3.  **Java技术的核心就是Java虚拟机**（JVM，Java Virtual Machine），因为所有的Java程序都运行在Java虚拟机内部。



**作用：**

Java虚拟机就是二进制字节码的运行环境，负责装载字节码到其内部，解释/编译为对应平台上的机器指令执行。每一条Java指令，Java虚拟机规范中都有详细定义，如怎么取操作数，怎么处理操作数，处理结果放在哪里。

**特点：**

1.  一次编译，到处运行
2.  自动内存管理
3.  自动垃圾回收功能



JVM的位置
----------

JVM是运行在操作系统之上的，它与硬件没有直接的交互

<img src="https://npm.elemecdn.com/youthlql@1.0.8/JVM/chapter_001/0013.png">



<img src="https://npm.elemecdn.com/youthlql@1.0.8/JVM/chapter_001/0014.png">



JVM的整体结构
-------------

1.  HotSpot VM是目前市面上高性能虚拟机的代表作之一。
2.  它采用解释器与即时编译器并存的架构。
3. 在今天，Java程序的运行性能早已脱胎换骨，已经达到了可以和C/C++程序一较高下的地步。

   

<img src="https://npm.elemecdn.com/youthlql@1.0.8/JVM/chapter_001/0015.png">



Java代码执行流程
--------------

凡是能生成被Java虚拟机所能解释、运行的字节码文件，那么理论上我们就可以自己设计一套语言了

<img src="https://npm.elemecdn.com/youthlql@1.0.8/JVM/chapter_001/0016.png">



JVM的架构模型
-----------

Java编译器输入的指令流基本上是一种**基于栈的指令集架构**，另外一种指令集架构则是**基于寄存器的指令集架构**。具体来说：这两种架构之间的区别：

### 基于栈的指令集架构

基于栈式架构的特点：

1.  设计和实现更简单，适用于资源受限的系统；
2.  避开了寄存器的分配难题：使用零地址指令方式分配
3.  指令流中的指令大部分是零地址指令，其执行过程依赖于操作栈。指令集更小，编译器容易实现
4.  不需要硬件支持，可移植性更好，更好实现跨平台

### 基于寄存器的指令级架构

基于寄存器架构的特点：

1.  典型的应用是x86的二进制指令集：比如传统的PC以及Android的Davlik虚拟机。
2.  指令集架构则完全依赖硬件，与硬件的耦合度高，可移植性差
3.  性能优秀和执行更高效
4.  花费更少的指令去完成一项操作
5.  在大部分情况下，基于寄存器架构的指令集往往都以一地址指令、二地址指令和三地址指令为主，而基于栈式架构的指令集却是以零地址指令为主

### 两种架构的举例

同样执行2+3这种逻辑操作，其指令分别如下：

*   **基于栈的计算流程（以Java虚拟机为例）：**

    ```java
    iconst_2 //常量2入栈
    istore_1
    iconst_3 // 常量3入栈
    istore_2
    iload_1
    iload_2
    iadd //常量2/3出栈，执行相加
    istore_0 // 结果5入栈
    ```
    
    8个指令
    
    
    
*   **而基于寄存器的计算流程**

    ```java
    mov eax,2 //将eax寄存器的值设为1
    add eax,3 //使eax寄存器的值加3
    ```
    
    2个指令

> 具体后面会讲



### JVM架构总结

1.  **由于跨平台性的设计，Java的指令都是根据栈来设计的**。不同平台CPU架构不同，所以不能设计为基于寄存器的。栈的优点：跨平台，指令集小，编译器容易实现，缺点是性能比寄存器差一些。
  
2.  时至今日，尽管嵌入式平台已经不是Java程序的主流运行平台了（准确来说应该是HotSpot VM的宿主环境已经不局限于嵌入式平台了），那么为什么不将架构更换为基于寄存器的架构呢？
  - 因为基于栈的架构跨平台性好、指令集小，虽然相对于基于寄存器的架构来说，基于栈的架构编译得到的指令更多，执行性能也不如基于寄存器的架构好，但考虑到其跨平台性与移植性，我们还是选用栈的架构
  



JVM的生命周期
-----------

### 虚拟机的启动

Java虚拟机的启动是通过引导类加载器（bootstrap class loader）创建一个初始类（initial class）来完成的，这个类是由虚拟机的具体实现指定的。

### 虚拟机的执行

1.  一个运行中的Java虚拟机有着一个清晰的任务：执行Java程序
2.  程序开始执行时他才运行，程序结束时他就停止
3.  **执行一个所谓的Java程序的时候，真真正正在执行的是一个叫做Java虚拟机的进程**

### 虚拟机的退出

**有如下的几种情况：**

1.  程序正常执行结束
  
2.  程序在执行过程中遇到了异常或错误而异常终止
  
3.  由于操作系统用现错误而导致Java虚拟机进程终止
  
4.  某线程调用Runtime类或System类的exit()方法，或Runtime类的halt()方法，并且Java安全管理器也允许这次exit()或halt()操作。
  
5.  除此之外，JNI（Java Native Interface）规范描述了用JNI Invocation API来加载或卸载 Java虚拟机时，Java虚拟机的退出情况。
  



JVM发展历程
-----------

### Sun Classic VM

1.  早在1996年Java1.0版本的时候，Sun公司发布了一款名为sun classic VM的Java虚拟机，它同时也是**世界上第一款商用Java虚拟机**，JDK1.4时完全被淘汰。
2.  这款虚拟机内部只提供解释器，没有即时编译器，因此效率比较低。【即时编译器会把热点代码的本地机器指令缓存起来，那么以后使用热点代码的时候，效率就比较高】
3.  如果使用JIT编译器，就需要进行外挂。但是一旦使用了JIT编译器，JIT就会接管虚拟机的执行系统。解释器就不再工作，解释器和编译器不能配合工作。
    - 我们将字节码指令翻译成机器指令也是需要花时间的，如果只使用JIT，就需要把所有字节码指令都翻译成机器指令，就会导致翻译时间过长，也就是说在程序刚启动的时候，等待时间会很长。
    - 而解释器就是走到哪，解释到哪。
4.  现在Hotspot内置了此虚拟机。

### Exact VM

1.  为了解决上一个虚拟机问题，jdk1.2时，Sun提供了此虚拟机。
  
2.  Exact Memory Management：准确式内存管理
  
    *   也可以叫Non-Conservative/Accurate Memory Management
      
    *   虚拟机可以知道内存中某个位置的数据具体是什么类型。
    
3.  具备现代高性能虚拟机的维形
  
    *   热点探测（寻找出热点代码进行缓存）
      
    *   编译器与解释器混合工作模式
    
4.  只在Solaris平台短暂使用，其他平台上还是classic vm，英雄气短，终被Hotspot虚拟机替换
  

### HotSpot VM（重点）

1.  HotSpot历史
  
    *   最初由一家名为“Longview Technologies”的小公司设计
      
    *   1997年，此公司被Sun收购；2009年，Sun公司被甲骨文收购。
      
    *   JDK1.3时，HotSpot VM成为默认虚拟机
    
2.  目前**Hotspot占有绝对的市场地位，称霸武林**。
  
    *   不管是现在仍在广泛使用的JDK6，还是使用比例较多的JDK8中，默认的虚拟机都是HotSpot
      
    *   Sun/oracle JDK和openJDK的默认虚拟机
      
    *   因此本课程中默认介绍的虚拟机都是HotSpot，相关机制也主要是指HotSpot的GC机制。（比如其他两个商用虚机都没有方法区的概念）
    
3.  从服务器、桌面到移动端、嵌入式都有应用。
  
4.  名称中的HotSpot指的就是它的热点代码探测技术。
  
    *   通过计数器找到最具编译价值代码，触发即时编译或栈上替换
    
    *   通过编译器与解释器协同工作，在最优化的程序响应时间与最佳执行性能中取得平衡

### JRockit（商用三大虚拟机之一）

1.  专注于服务器端应用：它可以不太关注程序启动速度，因此JRockit内部不包含解析器实现，全部代码都靠即时编译器编译后执行。
  
2.  大量的行业基准测试显示，JRockit JVM是世界上最快的JVM：使用JRockit产品，客户已经体验到了显著的性能提高（一些超过了70%）和硬件成本的减少（达50%）。
  
3.  优势：全面的Java运行时解决方案组合
  
    *   JRockit面向延迟敏感型应用的解决方案JRockit Real Time提供以毫秒或微秒级的JVM响应时间，适合财务、军事指挥、电信网络的需要
      
    *   Mission Control服务套件，它是一组以极低的开销来监控、管理和分析生产环境中的应用程序的工具。
    
4.  2008年，JRockit被Oracle收购。
  
5.  Oracle表达了整合两大优秀虚拟机的工作，大致在JDK8中完成。整合的方式是在HotSpot的基础上，移植JRockit的优秀特性。
  
6.  高斯林：目前就职于谷歌，研究人工智能和水下机器人
  

### IBM的J9（商用三大虚拟机之一）

1.  全称：IBM Technology for Java Virtual Machine，简称IT4J，内部代号：J9
  
2.  市场定位与HotSpot接近，服务器端、桌面应用、嵌入式等多用途VM广泛用于IBM的各种Java产品。
  
3.  目前，有影响力的三大商用虚拟机之一，也号称是世界上最快的Java虚拟机。
  
4.  2017年左右，IBM发布了开源J9VM，命名为openJ9，交给Eclipse基金会管理，也称为Eclipse OpenJ9
  
5.  OpenJDK -> 是JDK开源了，包括了虚拟机
  

### KVM和CDC/CLDC Hotspot

1.  Oracle在Java ME产品线上的两款虚拟机为：CDC/CLDC HotSpot Implementation VM 
2.  KVM（Kilobyte）是CLDC-HI早期产品
3.  目前移动领域地位尴尬，智能机被Android和iOS二分天下。
4.  KVM简单、轻量、高度可移植，面向更低端的设备上还维持自己的一片市场

    *   智能控制器、传感器
      
    *   老人手机、经济欠发达地区的功能手机
5.  所有的虚拟机的原则：一次编译，到处运行。


### Azul VM

1.  前面三大“高性能Java虚拟机”使用在**通用硬件平台上**
2.  这里Azul VW和BEA Liquid VM是与**特定硬件平台绑定**、软硬件配合的专有虚拟机：高性能Java虚拟机中的战斗机。
3.  Azul VM是Azul Systems公司在HotSpot基础上进行大量改进，运行于Azul Systems公司的专有硬件Vega系统上的Java虚拟机。
4.  每个Azul VM实例都可以管理至少数十个CPU和数百GB内存的硬件资源，并提供在巨大内存范围内实现可控的GC时间的垃圾收集器、专有硬件优化的线程调度等优秀特性。
5.  2010年，Azul Systems公司开始从硬件转向软件，发布了自己的Zing JVM，可以在通用x86平台上提供接近于Vega系统的特性。


### Liquid VM

1.  高性能Java虚拟机中的战斗机。
2.  BEA公司开发的，直接运行在自家Hypervisor系统上
3.  Liquid VM即是现在的JRockit VE（Virtual Edition）。**Liquid VM不需要操作系统的支持，或者说它自己本身实现了一个专用操作系统的必要功能，如线程调度、文件系统、网络支持等**。
5.  随着JRockit虚拟机终止开发，Liquid vM项目也停止了。

### Apache Marmony

1.  Apache也曾经推出过与JDK1.5和JDK1.6兼容的Java运行平台Apache Harmony。
  
2.  它是IElf和Intel联合开发的开源JVM，受到同样开源的Open JDK的压制，Sun坚决不让Harmony获得JCP认证，最终于2011年退役，IBM转而参与OpenJDK
  
3.  虽然目前并没有Apache Harmony被大规模商用的案例，但是它的Java类库代码吸纳进了Android SDK。
  

### Micorsoft JVM

1.  微软为了在IE3浏览器中支持Java Applets，开发了Microsoft JVM。
  
2.  只能在window平台下运行。但确是当时Windows下性能最好的Java VM。
  
3.  1997年，Sun以侵犯商标、不正当竞争罪名指控微软成功，赔了Sun很多钱。微软WindowsXP SP3中抹掉了其VM。现在Windows上安装的jdk都是HotSpot。
  

### Taobao JVM

1.  由AliJVM团队发布。阿里，国内使用Java最强大的公司，覆盖云计算、金融、物流、电商等众多领域，需要解决高并发、高可用、分布式的复合问题。有大量的开源产品。
  
2.  **基于OpenJDK开发了自己的定制版本AlibabaJDK**，简称AJDK。是整个阿里Java体系的基石。
  
3.  基于OpenJDK Hotspot VM发布的国内第一个优化、深度定制且开源的高性能服务器版Java虚拟机。
  
    *   创新的GCIH（GCinvisible heap）技术实现了off-heap，**即将生命周期较长的Java对象从heap中移到heap之外，并且GC不能管理GCIH内部的Java对象，以此达到降低GC的回收频率和提升GC的回收效率的目的**。
    *   GCIH中的**对象还能够在多个Java虚拟机进程中实现共享**
    *   使用crc32指令实现JvM intrinsic降低JNI的调用开销
    *   PMU hardware的Java profiling tool和诊断协助功能
    *   针对大数据场景的ZenGC
4.  taobao vm应用在阿里产品上性能高，**硬件严重依赖inte1的cpu，损失了兼容性，但提高了性能**
  - 目前已经在淘宝、天猫上线，把Oracle官方JvM版本全部替换了。
  

### Dalvik VM

1.  谷歌开发的，应用于Android系统，并在Android2.2中提供了JIT，发展迅猛。
  
2.  **Dalvik VM只能称作虚拟机，而不能称作“Java虚拟机”**，它没有遵循 Java虚拟机规范
  
3.  不能直接执行Java的Class文件
  
4.  基于寄存器架构，不是jvm的栈架构。
  
5.  执行的是编译以后的dex（Dalvik Executable）文件。执行效率比较高。
  - 它执行的dex（Dalvik Executable）文件可以通过class文件转化而来，使用Java语法编写应用程序，可以直接使用大部分的Java API等。
  
7.  Android 5.0使用支持提前编译（Ahead of Time Compilation，AoT）的ART VM替换Dalvik VM。
  

### Graal VM（未来虚拟机）

1.  2018年4月，Oracle Labs公开了GraalvM，号称 “**Run Programs Faster Anywhere**”，勃勃野心。与1995年java的”write once，run anywhere"遥相呼应。
  
2.  GraalVM在HotSpot VM基础上增强而成的**跨语言全栈虚拟机，可以作为“任何语言”**的运行平台使用。语言包括：Java、Scala、Groovy、Kotlin；C、C++、Javascript、Ruby、Python、R等
  
3.  支持不同语言中混用对方的接口和对象，支持这些语言使用已经编写好的本地库文件
  
4.  工作原理是将这些语言的源代码或源代码编译后的中间格式，通过解释器转换为能被Graal VM接受的中间表示。Graal VM提供Truffle工具集快速构建面向一种新语言的解释器。在运行时还能进行即时编译优化，获得比原生编译器更优秀的执行效率。
  
5.  **如果说HotSpot有一天真的被取代，Graalvm希望最大**。但是Java的软件生态没有丝毫变化。



### 总结

具体JVM的内存结构，其实取决于其实现，不同厂商的JVM，或者同一厂商发布的不同版本，都有可能存在一定差异。主要以Oracle HotSpot VM为默认虚拟机。