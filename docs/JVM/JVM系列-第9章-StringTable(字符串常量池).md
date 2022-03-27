---
title: JVM系列-第9章-StringTable(字符串常量池)
tags:
  - JVM
  - 虚拟机
categories:
  - JVM
  - 1.内存与垃圾回收篇
keywords: JVM，虚拟机。
description: JVM系列-第9章-StringTable(字符串常量池)。
cover: 'https://npm.elemecdn.com/lql_static@latest/logo/jvm.png'
abbrlink: ee2ba71e
date: 2020-11-16 12:38:02
---



StringTable（字符串常量池）
==================

String的基本特性
--------------



1.  String：字符串，使用一对 “” 引起来表示
  
  ```java
    String s1 = "atguigu" ;   			// 字面量的定义方式
    String s2 =  new String("hello");     // new 对象的方式
  ```
  
2.  String被声明为final的，不可被继承
  
3.  String实现了Serializable接口：表示字符串是支持序列化的。实现了Comparable接口：表示String可以比较大小
  
4.  String在jdk8及以前内部定义了`final char value[]`用于存储字符串数据。JDK9时改为`byte[]`
  

## 为什么 JDK9 改变了 String 的结构

>  **官方文档**：http://openjdk.java.net/jeps/254



**为什么改为 byte\[\] 存储？**

1.  String类的当前实现将字符存储在char数组中，每个字符使用两个字节(16位)。
2.  从许多不同的应用程序收集的数据表明，字符串是堆使用的主要组成部分，而且大多数字符串对象只包含拉丁字符（Latin-1）。这些字符只需要一个字节的存储空间，因此这些字符串对象的内部char数组中有一半的空间将不会使用，产生了大量浪费。
3.  之前 String 类使用 UTF-16 的 char\[\] 数组存储，现在改为 byte\[\] 数组 外加一个编码标识存储。该编码表示如果你的字符是ISO-8859-1或者Latin-1，那么只需要一个字节存。如果你是其它字符集，比如UTF-8，你仍然用两个字节存
4.  结论：String再也不用char\[\] 来存储了，改成了byte \[\] 加上编码标记，节约了一些空间
5.  同时基于String的数据结构，例如StringBuffer和StringBuilder也同样做了修改

```java
// 之前
private final char value[];
// 之后
private final byte[] value
```



### String 的基本特性



- String：代表不可变的字符序列。简称：不可变性。

1.  当对字符串重新赋值时，需要重写指定内存区域赋值，不能使用原有的value进行赋值。
2.  当对现有的字符串进行连接操作时，也需要重新指定内存区域赋值，不能使用原有的value进行赋值。
3.  当调用String的replace()方法修改指定字符或字符串时，也需要重新指定内存区域赋值，不能使用原有的value进行赋值。

- 通过字面量的方式（区别于new）给一个字符串赋值，此时的字符串值声明在字符串常量池中。



**当对字符串重新赋值时，需要重写指定内存区域赋值，不能使用原有的value进行赋值**

代码

```java
	@Test
    public void test1() {
        String s1 = "abc";//字面量定义的方式，"abc"存储在字符串常量池中
        String s2 = "abc";
        s1 = "hello";

        System.out.println(s1 == s2);//判断地址：true  --> false

        System.out.println(s1);//
        System.out.println(s2);//abc

    }
```

字节码指令
*   取字符串 “abc” 时，使用的是同一个符号引用：#2
*   取字符串 “hello” 时，使用的是另一个符号引用：#3



**当对现有的字符串进行连接操作时，也需要重新指定内存区域赋值，不能使用原有的value进行赋值**

```java
	@Test
    public void test2() {
        String s1 = "abc";
        String s2 = "abc";
        s2 += "def";
        System.out.println(s2);//abcdef
        System.out.println(s1);//abc
    }
```



**当调用string的replace()方法修改指定字符或字符串时，也需要重新指定内存区域赋值，不能使用原有的value进行赋值**

```java
@Test
public void test3() {
    String s1 = "abc";
    String s2 = s1.replace('a', 'm');
    System.out.println(s1);//abc
    System.out.println(s2);//mbc
}
```



**一道笔试题**

```java
public class StringExer {
    String str = new String("good");
    char[] ch = {'t', 'e', 's', 't'};

    public void change(String str, char ch[]) {
        str = "test ok";
        ch[0] = 'b';
    }

    public static void main(String[] args) {
        StringExer ex = new StringExer();
        ex.change(ex.str, ex.ch);
        System.out.println(ex.str);//good
        System.out.println(ex.ch);//best
    }

}
```

str 的内容并没有变：“test ok” 位于字符串常量池中的另一个区域（地址），进行赋值操作并没有修改原来 str 指向的引用的内容



### String 的底层结构



**字符串常量池是不会存储相同内容的字符串的**

1.  String的String Pool（字符串常量池）是一个固定大小的Hashtable，默认值大小长度是1009。如果放进String Pool的String非常多，就会造成Hash冲突严重，从而导致链表会很长，而链表长了后直接会造成的影响就是当调用String.intern()方法时性能会大幅下降。
2.  使用-XX:StringTablesize可设置StringTable的长度
3.  在JDK6中StringTable是固定的，就是1009的长度，所以如果常量池中的字符串过多就会导致效率下降很快，StringTablesize设置没有要求
4.  在JDK7中，StringTable的长度默认值是60013，StringTablesize设置没有要求
5.  在JDK8中，StringTable的长度默认值是60013，StringTable可以设置的最小值为1009

<img src="https://npm.elemecdn.com/youthlql@1.0.8/JVM/chapter_009/0001.png">



<img src="https://npm.elemecdn.com/youthlql@1.0.8/JVM/chapter_009/0002.png">



**测试不同 StringTable 长度下，程序的性能**

代码

```java
/**
 * 产生10万个长度不超过10的字符串，包含a-z,A-Z
 */
public class GenerateString {
    public static void main(String[] args) throws IOException {
        FileWriter fw =  new FileWriter("words.txt");

        for (int i = 0; i < 100000; i++) {
            //1 - 10
           int length = (int)(Math.random() * (10 - 1 + 1) + 1);
            fw.write(getString(length) + "\n");
        }

        fw.close();
    }

    public static String getString(int length){
        String str = "";
        for (int i = 0; i < length; i++) {
            //65 - 90, 97-122
            int num = (int)(Math.random() * (90 - 65 + 1) + 65) + (int)(Math.random() * 2) * 32;
            str += (char)num;
        }
        return str;
    }
}

```



```java
public class StringTest2 {
    public static void main(String[] args) {

        BufferedReader br = null;
        try {
            br = new BufferedReader(new FileReader("words.txt"));
            long start = System.currentTimeMillis();
            String data;
            while((data = br.readLine()) != null){
                data.intern(); //如果字符串常量池中没有对应data的字符串的话，则在常量池中生成
            }

            long end = System.currentTimeMillis();

            System.out.println("花费的时间为：" + (end - start));//1009:143ms  100009:47ms
        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            if(br != null){
                try {
                    br.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }

            }
        }
    }
}

```



*   -XX:StringTableSize=1009 ：程序耗时 143ms
  
*   -XX:StringTableSize=100009 ：程序耗时 47ms
  

String 的内存分配
--------------





1.  在Java语言中有8种基本数据类型和一种比较特殊的类型String。这些类型为了使它们在运行过程中速度更快、更节省内存，都提供了一种常量池的概念。
  
2.  常量池就类似一个Java系统级别提供的缓存。8种基本数据类型的常量池都是系统协调的，String类型的常量池比较特殊。它的主要使用方法有两种。
  
    *   直接使用双引号声明出来的String对象会直接存储在常量池中。比如：`String info="atguigu.com";`
      
    *   如果不是用双引号声明的String对象，可以使用String提供的intern()方法。这个后面重点谈
      



1.  Java 6及以前，字符串常量池存放在永久代
2.  Java 7中 Oracle的工程师对字符串池的逻辑做了很大的改变，即将字符串常量池的位置调整到Java堆内
    *   所有的字符串都保存在堆（Heap）中，和其他普通对象一样，这样可以让你在进行调优应用时仅需要调整堆大小就可以了。
    *   字符串常量池概念原本使用得比较多，但是这个改动使得我们有足够的理由让我们重新考虑在Java 7中使用String.intern()。
3.  Java8元空间，字符串常量在堆



<img src="https://npm.elemecdn.com/youthlql@1.0.8/JVM/chapter_009/0003.png">



<img src="https://npm.elemecdn.com/youthlql@1.0.8/JVM/chapter_009/0004.png">



### StringTable 为什么要调整？

> **官方文档**:https://www.oracle.com/java/technologies/javase/jdk7-relnotes.html#jdk7changes



1.  为什么要调整位置？
    *   永久代的默认空间大小比较小
    *   永久代垃圾回收频率低，大量的字符串无法及时回收，容易进行Full GC产生STW或者容易产生OOM：PermGen Space
    *   堆中空间足够大，字符串可被及时回收
2.  在JDK 7中，interned字符串不再在Java堆的永久代中分配，而是在Java堆的主要部分（称为年轻代和年老代）中分配，与应用程序创建的其他对象一起分配。
3.  此更改将导致驻留在主Java堆中的数据更多，驻留在永久生成中的数据更少，因此可能需要调整堆大小。

**代码示例**

```java
/**
 * jdk6中：
 * -XX:PermSize=6m -XX:MaxPermSize=6m -Xms6m -Xmx6m
 *
 * jdk8中：
 * -XX:MetaspaceSize=6m -XX:MaxMetaspaceSize=6m -Xms6m -Xmx6m
 */
public class StringTest3 {
    public static void main(String[] args) {
        //使用Set保持着常量池引用，避免full gc回收常量池行为
        Set<String> set = new HashSet<String>();
        //在short可以取值的范围内足以让6MB的PermSize或heap产生OOM了。
        short i = 0;
        while(true){
            set.add(String.valueOf(i++).intern());
        }
    }
}
```



输出结果：我真没骗你，字符串真的在堆中（JDK8）


```java
Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
	at java.util.HashMap.resize(HashMap.java:703)
	at java.util.HashMap.putVal(HashMap.java:662)
	at java.util.HashMap.put(HashMap.java:611)
	at java.util.HashSet.add(HashSet.java:219)
	at com.atguigu.java.StringTest3.main(StringTest3.java:22)

Process finished with exit code 1
```



String 的基本操作
--------------



Java语言规范里要求完全相同的字符串字面量，应该包含同样的Unicode字符序列（包含同一份码点序列的常量），并且必须是指向同一个String类实例。

### 举例1

```java
public class StringTest4 {
    public static void main(String[] args) {
        System.out.println();//2293
        System.out.println("1");//2294
        System.out.println("2");
        System.out.println("3");
        System.out.println("4");
        System.out.println("5");
        System.out.println("6");
        System.out.println("7");
        System.out.println("8");
        System.out.println("9");
        System.out.println("10");//2303
        //如下的字符串"1" 到 "10"不会再次加载
        System.out.println("1");//2304
        System.out.println("2");//2304
        System.out.println("3");
        System.out.println("4");
        System.out.println("5");
        System.out.println("6");
        System.out.println("7");
        System.out.println("8");
        System.out.println("9");
        System.out.println("10");//2304
    }
}
```



分析字符串常量池的变化

1、程序启动时已经加载了 2293 个字符串常量

<img src="https://npm.elemecdn.com/youthlql@1.0.8/JVM/chapter_009/0005.png">

2、加载了一个换行符（println），所以多了一个

<img src="https://npm.elemecdn.com/youthlql@1.0.8/JVM/chapter_009/0006.jpg">

3、加载了字符串常量 “1”~“9”

<img src="https://npm.elemecdn.com/youthlql@1.0.8/JVM/chapter_009/0007.jpg">

4、加载字符串常量 “10”

<img src="https://npm.elemecdn.com/youthlql@1.0.8/JVM/chapter_009/0008.jpg">

5、之后的字符串"1" 到 "10"不会再次加载

<img src="https://npm.elemecdn.com/youthlql@1.0.8/JVM/chapter_009/0009.png">



### 举例2

```java
//官方示例代码
class Memory {
    public static void main(String[] args) {//line 1
        int i = 1;//line 2
        Object obj = new Object();//line 3
        Memory mem = new Memory();//line 4
        mem.foo(obj);//line 5
    }//line 9

    private void foo(Object param) {//line 6
        String str = param.toString();//line 7
        System.out.println(str);
    }//line 8
}
```



分析运行时内存（foo() 方法是实例方法，其实图中少了一个 this 局部变量）

<img src="https://npm.elemecdn.com/youthlql@1.0.8/JVM/chapter_009/0010.png">



字符串拼接操作
---------

### 先说结论



1.  常量与常量的拼接结果在常量池，原理是编译期优化
2.  常量池中不会存在相同内容的变量
3.  拼接前后，只要其中有一个是变量，结果就在堆中。变量拼接的原理是StringBuilder
4.  如果拼接的结果调用intern()方法，根据该字符串是否在常量池中存在，分为：
    *   如果存在，则返回字符串在常量池中的地址
    *   如果字符串常量池中不存在该字符串，则在常量池中创建一份，并返回此对象的地址



**1、常量与常量的拼接结果在常量池，原理是编译期优化**

代码

```java
@Test
    public void test1(){
        String s1 = "a" + "b" + "c";//编译期优化：等同于"abc"
        String s2 = "abc"; //"abc"一定是放在字符串常量池中，将此地址赋给s2
        /*
         * 最终.java编译成.class,再执行.class
         * String s1 = "abc";
         * String s2 = "abc"
         */
        System.out.println(s1 == s2); //true
        System.out.println(s1.equals(s2)); //true
    }
```



从字节码指令看出：编译器做了优化，将 “a” + “b” + “c” 优化成了 “abc”

```java
0 ldc #2 <abc>
2 astore_1
3 ldc #2 <abc>
5 astore_2
6 getstatic #3 <java/lang/System.out>
9 aload_1
10 aload_2
11 if_acmpne 18 (+7)
14 iconst_1
15 goto 19 (+4)
18 iconst_0
19 invokevirtual #4 <java/io/PrintStream.println>
22 getstatic #3 <java/lang/System.out>
25 aload_1
26 aload_2
27 invokevirtual #5 <java/lang/String.equals>
30 invokevirtual #4 <java/io/PrintStream.println>
33 return
```



IDEA 反编译 class 文件后，来看这个问题

<img src="https://npm.elemecdn.com/youthlql@1.0.8/JVM/chapter_009/0011.png">



**2、拼接前后，只要其中有一个是变量，结果就在堆中**

**调用 intern() 方法，则主动将字符串对象存入字符串常量池中，并将其地址返回**

```java
@Test
    public void test2(){
        String s1 = "javaEE";
        String s2 = "hadoop";

        String s3 = "javaEEhadoop";
        String s4 = "javaEE" + "hadoop";//编译期优化
        //如果拼接符号的前后出现了变量，则相当于在堆空间中new String()，具体的内容为拼接的结果：javaEEhadoop
        String s5 = s1 + "hadoop";
        String s6 = "javaEE" + s2;
        String s7 = s1 + s2;

        System.out.println(s3 == s4);//true
        System.out.println(s3 == s5);//false
        System.out.println(s3 == s6);//false
        System.out.println(s3 == s7);//false
        System.out.println(s5 == s6);//false
        System.out.println(s5 == s7);//false
        System.out.println(s6 == s7);//false
        //intern():判断字符串常量池中是否存在javaEEhadoop值，如果存在，则返回常量池中javaEEhadoop的地址；
        //如果字符串常量池中不存在javaEEhadoop，则在常量池中加载一份javaEEhadoop，并返回次对象的地址。
        String s8 = s6.intern();
        System.out.println(s3 == s8);//true
    }

```



从字节码角度来看：拼接前后有变量，都会使用到 StringBuilder 类

```java
0 ldc #6 <javaEE>
2 astore_1
3 ldc #7 <hadoop>
5 astore_2
6 ldc #8 <javaEEhadoop>
8 astore_3
9 ldc #8 <javaEEhadoop>
11 astore 4
13 new #9 <java/lang/StringBuilder>
16 dup
17 invokespecial #10 <java/lang/StringBuilder.<init>>
20 aload_1
21 invokevirtual #11 <java/lang/StringBuilder.append>
24 ldc #7 <hadoop>
26 invokevirtual #11 <java/lang/StringBuilder.append>
29 invokevirtual #12 <java/lang/StringBuilder.toString>
32 astore 5
34 new #9 <java/lang/StringBuilder>
37 dup
38 invokespecial #10 <java/lang/StringBuilder.<init>>
41 ldc #6 <javaEE>
43 invokevirtual #11 <java/lang/StringBuilder.append>
46 aload_2
47 invokevirtual #11 <java/lang/StringBuilder.append>
50 invokevirtual #12 <java/lang/StringBuilder.toString>
53 astore 6
55 new #9 <java/lang/StringBuilder>
58 dup
59 invokespecial #10 <java/lang/StringBuilder.<init>>
62 aload_1
63 invokevirtual #11 <java/lang/StringBuilder.append>
66 aload_2
67 invokevirtual #11 <java/lang/StringBuilder.append>
70 invokevirtual #12 <java/lang/StringBuilder.toString>
73 astore 7
75 getstatic #3 <java/lang/System.out>
78 aload_3
79 aload 4
81 if_acmpne 88 (+7)
84 iconst_1
85 goto 89 (+4)
88 iconst_0
89 invokevirtual #4 <java/io/PrintStream.println>
92 getstatic #3 <java/lang/System.out>
95 aload_3
96 aload 5
98 if_acmpne 105 (+7)
101 iconst_1
102 goto 106 (+4)
105 iconst_0
106 invokevirtual #4 <java/io/PrintStream.println>
109 getstatic #3 <java/lang/System.out>
112 aload_3
113 aload 6
115 if_acmpne 122 (+7)
118 iconst_1
119 goto 123 (+4)
122 iconst_0
123 invokevirtual #4 <java/io/PrintStream.println>
126 getstatic #3 <java/lang/System.out>
129 aload_3
130 aload 7
132 if_acmpne 139 (+7)
135 iconst_1
136 goto 140 (+4)
139 iconst_0
140 invokevirtual #4 <java/io/PrintStream.println>
143 getstatic #3 <java/lang/System.out>
146 aload 5
148 aload 6
150 if_acmpne 157 (+7)
153 iconst_1
154 goto 158 (+4)
157 iconst_0
158 invokevirtual #4 <java/io/PrintStream.println>
161 getstatic #3 <java/lang/System.out>
164 aload 5
166 aload 7
168 if_acmpne 175 (+7)
171 iconst_1
172 goto 176 (+4)
175 iconst_0
176 invokevirtual #4 <java/io/PrintStream.println>
179 getstatic #3 <java/lang/System.out>
182 aload 6
184 aload 7
186 if_acmpne 193 (+7)
189 iconst_1
190 goto 194 (+4)
193 iconst_0
194 invokevirtual #4 <java/io/PrintStream.println>
197 aload 6
199 invokevirtual #13 <java/lang/String.intern>
202 astore 8
204 getstatic #3 <java/lang/System.out>
207 aload_3
208 aload 8
210 if_acmpne 217 (+7)
213 iconst_1
214 goto 218 (+4)
217 iconst_0
218 invokevirtual #4 <java/io/PrintStream.println>
221 return
```



### 字符串拼接的底层细节

**举例1**

```java
    @Test
    public void test3(){
        String s1 = "a";
        String s2 = "b";
        String s3 = "ab";
        /*
        如下的s1 + s2 的执行细节：(变量s是我临时定义的）
        ① StringBuilder s = new StringBuilder();
        ② s.append("a")
        ③ s.append("b")
        ④ s.toString()  --> 约等于 new String("ab")，但不等价

        补充：在jdk5.0之后使用的是StringBuilder,在jdk5.0之前使用的是StringBuffer
         */
        String s4 = s1 + s2;//
        System.out.println(s3 == s4);//false
    }
```



字节码指令

```java
0 ldc #14 <a>
2 astore_1
3 ldc #15 <b>
5 astore_2
6 ldc #16 <ab>
8 astore_3
9 new #9 <java/lang/StringBuilder>
12 dup
13 invokespecial #10 <java/lang/StringBuilder.<init>>
16 aload_1
17 invokevirtual #11 <java/lang/StringBuilder.append>
20 aload_2
21 invokevirtual #11 <java/lang/StringBuilder.append>
24 invokevirtual #12 <java/lang/StringBuilder.toString>
27 astore 4
29 getstatic #3 <java/lang/System.out>
32 aload_3
33 aload 4
35 if_acmpne 42 (+7)
38 iconst_1
39 goto 43 (+4)
42 iconst_0
43 invokevirtual #4 <java/io/PrintStream.println>
46 return
```





**举例2**

```java
/*
    1. 字符串拼接操作不一定使用的是StringBuilder!
       如果拼接符号左右两边都是字符串常量或常量引用，则仍然使用编译期优化，即非StringBuilder的方式。
    2. 针对于final修饰类、方法、基本数据类型、引用数据类型的量的结构时，能使用上final的时候建议使用上。
     */
    @Test
    public void test4(){
        final String s1 = "a";
        final String s2 = "b";
        String s3 = "ab";
        String s4 = s1 + s2;
        System.out.println(s3 == s4);//true
    }
```



从字节码角度来看：为变量 s4 赋值时，直接使用 #16 符号引用，即字符串常量 “ab”

```java
0 ldc #14 <a>
2 astore_1
3 ldc #15 <b>
5 astore_2
6 ldc #16 <ab>
8 astore_3
9 ldc #16 <ab>
11 astore 4
13 getstatic #3 <java/lang/System.out>
16 aload_3
17 aload 4
19 if_acmpne 26 (+7)
22 iconst_1
23 goto 27 (+4)
26 iconst_0
27 invokevirtual #4 <java/io/PrintStream.println>
30 return
```





**拼接操作与 append 操作的效率对比**

```java

    @Test
    public void test6(){

        long start = System.currentTimeMillis();

//        method1(100000);//4014
        method2(100000);//7

        long end = System.currentTimeMillis();

        System.out.println("花费的时间为：" + (end - start));
    }

    public void method1(int highLevel){
        String src = "";
        for(int i = 0;i < highLevel;i++){
            src = src + "a";//每次循环都会创建一个StringBuilder、String
        }
//        System.out.println(src);

    }

    public void method2(int highLevel){
        //只需要创建一个StringBuilder
        StringBuilder src = new StringBuilder();
        for (int i = 0; i < highLevel; i++) {
            src.append("a");
        }
//        System.out.println(src);
    }
```



1.  体会执行效率：通过StringBuilder的append()的方式添加字符串的效率要远高于使用String的字符串拼接方式！
  
2.  原因：
  
    1.  StringBuilder的append()的方式：
        *   自始至终中只创建过一个StringBuilder的对象
    2.  使用String的字符串拼接方式：
        *   创建过多个StringBuilder和String（调的toString方法）的对象，内存占用更大；
        *   如果进行GC，需要花费额外的时间（在拼接的过程中产生的一些中间字符串可能永远也用不到，会产生大量垃圾字符串）。
3.  改进的空间：
  
    *   在实际开发中，如果基本确定要前前后后添加的字符串长度不高于某个限定值highLevel的情况下，建议使用构造器实例化：
    *   `StringBuilder s = new StringBuilder(highLevel); //new char[highLevel]`
    *   这样可以避免频繁扩容



intern() 的使用
--------------

### intern() 方法的说明



```java
public native String intern();
```



1.  intern是一个native方法，调用的是底层C的方法
  
2.  字符串常量池池最初是空的，由String类私有地维护。在调用intern方法时，如果池中已经包含了由equals(object)方法确定的与该字符串内容相等的字符串，则返回池中的字符串地址。否则，该字符串对象将被添加到池中，并返回对该字符串对象的地址。（这是源码里的大概翻译）
  
3.  如果不是用双引号声明的String对象，可以使用String提供的intern方法：intern方法会从字符串常量池中查询当前字符串是否存在，若不存在就会将当前字符串放入常量池中。比如：
  
        String myInfo = new string("I love atguigu").intern();
  
4.  也就是说，如果在任意字符串上调用String.intern方法，那么其返回结果所指向的那个类实例，必须和直接以常量形式出现的字符串实例完全相同。因此，下列表达式的值必定是true
  
        ("a"+"b"+"c").intern()=="abc"
  
5.  通俗点讲，Interned String就是确保字符串在内存里只有一份拷贝，这样可以节约内存空间，加快字符串操作任务的执行速度。注意，这个值会被存放在字符串内部池（String Intern Pool）
  



### new String() 的说明

#### new String(“ab”)会创建几个对象？

```java
/**
 * 题目：
 * new String("ab")会创建几个对象？看字节码，就知道是两个。
 *     一个对象是：new关键字在堆空间创建的
 *     另一个对象是：字符串常量池中的对象"ab"。 字节码指令：ldc
 *
 */
public class StringNewTest {
    public static void main(String[] args) {
        String str = new String("ab");
    }
}
```



字节码指令

```java
0 new #2 <java/lang/String>
3 dup
4 ldc #3 <ab>
6 invokespecial #4 <java/lang/String.<init>>
9 astore_1
10 return
```



`0 new #2 <java/lang/String>`：在堆中创建了一个 String 对象

`4 ldc #3 <ab>` ：在字符串常量池中放入 “ab”（如果之前字符串常量池中没有 “ab” 的话）



#### new String(“a”) + new String(“b”) 会创建几个对象？

代码

```java
/**
 * 思考：
 * new String("a") + new String("b")呢？
 *  对象1：new StringBuilder()
 *  对象2： new String("a")
 *  对象3： 常量池中的"a"
 *  对象4： new String("b")
 *  对象5： 常量池中的"b"
 *
 *  深入剖析： StringBuilder的toString():
 *      对象6 ：new String("ab")
 *       强调一下，toString()的调用，在字符串常量池中，没有生成"ab"
 *
 */
public class StringNewTest {
    public static void main(String[] args) {

        String str = new String("a") + new String("b");
    }
}
```

字节码指令

```java
0 new #2 <java/lang/StringBuilder>
3 dup
4 invokespecial #3 <java/lang/StringBuilder.<init>>
7 new #4 <java/lang/String>
10 dup
11 ldc #5 <a>
13 invokespecial #6 <java/lang/String.<init>>
16 invokevirtual #7 <java/lang/StringBuilder.append>
19 new #4 <java/lang/String>
22 dup
23 ldc #8 <b>
25 invokespecial #6 <java/lang/String.<init>>
28 invokevirtual #7 <java/lang/StringBuilder.append>
31 invokevirtual #9 <java/lang/StringBuilder.toString>
34 astore_1
35 return
```

**答案是4个或5个或6个**

字节码指令分析：
1.  `0 new #2 <java/lang/StringBuilder>` ：拼接字符串会创建一个 StringBuilder 对象
2.  `7 new #4 <java/lang/String>` ：创建 String 对象，对应于 new String(“a”)
3.  `11 ldc #5 <a>` ：在字符串常量池中放入 “a”（如果之前字符串常量池中没有 “a” 的话）
4.  `19 new #4 <java/lang/String>` ：创建 String 对象，对应于 new String(“b”)
5.  `23 ldc #8 <b>` ：在字符串常量池中放入 “b”（如果之前字符串常量池中没有 “b” 的话）
6.  `31 invokevirtual #9 <java/lang/StringBuilder.toString>` ：调用 StringBuilder 的 toString() 方法，会生成一个 String 对象

<img src="https://npm.elemecdn.com/youthlql@1.0.8/JVM/chapter_009/0012.png">





### 有点难的面试题

> **有点难的面试题**

```java
**
 * 如何保证变量s指向的是字符串常量池中的数据呢？
 * 有两种方式：
 * 方式一： String s = "shkstart";//字面量定义的方式
 * 方式二： 调用intern()
 *         String s = new String("shkstart").intern();
 *         String s = new StringBuilder("shkstart").toString().intern();
 *
 */
public class 	StringIntern {
    public static void main(String[] args) {

        String s = new String("1");
        s.intern();//调用此方法之前，字符串常量池中已经存在了"1"
        String s2 = "1";
        System.out.println(s == s2);//jdk6：false   jdk7/8：false
        
        /*
         1、s3变量记录的地址为：new String("11")
         2、经过上面的分析，我们已经知道执行完pos_1的代码，在堆中有了一个new String("11")
         这样的String对象。但是在字符串常量池中没有"11"
         3、接着执行s3.intern()，在字符串常量池中生成"11"
           3-1、在JDK6的版本中，字符串常量池还在永久代，所以直接在永久代生成"11",也就有了新的地址
           3-2、而在JDK7的后续版本中，字符串常量池被移动到了堆中，此时堆里已经有new String（"11"）了
           出于节省空间的目的，直接将堆中的那个字符串的引用地址储存在字符串常量池中。没错，字符串常量池
           中存的是new String（"11"）在堆中的地址
         4、所以在JDK7后续版本中，s3和s4指向的完全是同一个地址。
         */
        String s3 = new String("1") + new String("1");//pos_1
	    s3.intern();
        
        String s4 = "11";//s4变量记录的地址：使用的是上一行代码代码执行时，在常量池中生成的"11"的地址
        System.out.println(s3 == s4);//jdk6：false  jdk7/8：true
    }


}

```

解释的已经比较清楚了，下面看一下内存图

**内存分析**

JDK6 ：正常眼光判断即可
*   new String() 即在堆中
*   str.intern() 则把字符串放入常量池中

<img src="https://npm.elemecdn.com/youthlql@1.0.8/JVM/chapter_009/0013.png">



JDK7及后续版本，**注意大坑**

<img src="https://npm.elemecdn.com/youthlql@1.0.8/JVM/chapter_009/0014.png">



#### 面试题的拓展

```java
/**
 * StringIntern.java中练习的拓展：
 *
 */
public class StringIntern1 {
    public static void main(String[] args) {
        //执行完下一行代码以后，字符串常量池中，是否存在"11"呢？答案：不存在！！
        String s3 = new String("1") + new String("1");//new String("11")
        //在字符串常量池中生成对象"11"，代码顺序换一下，实打实的在字符串常量池里有一个"11"对象
        String s4 = "11";  
        String s5 = s3.intern();

        // s3 是堆中的 "ab" ，s4 是字符串常量池中的 "ab"
        System.out.println(s3 == s4);//false

        // s5 是从字符串常量池中取回来的引用，当然和 s4 相等
        System.out.println(s5 == s4);//true
    }
}
```





### intern() 方法的练习



**练习 1**

```java
public class StringExer1 {
    public static void main(String[] args) {
        String x = "ab";
        String s = new String("a") + new String("b");//new String("ab")
        //在上一行代码执行完以后，字符串常量池中并没有"ab"
		/*
		1、jdk6中：在字符串常量池（此时在永久代）中创建一个字符串"ab"
        2、jdk8中：字符串常量池（此时在堆中）中没有创建字符串"ab",而是创建一个引用，指向new String("ab")，		  将此引用返回
        3、详解看上面
		*/
        String s2 = s.intern();

        System.out.println(s2 == "ab");//jdk6:true  jdk8:true
        System.out.println(s == "ab");//jdk6:false  jdk8:true
    }
}
```

**JDK6**

![image-20201116113423492](https://npm.elemecdn.com/youthlql@1.0.8/JVM/chapter_009/0015.png)

**JDK7/8**

<img src="https://npm.elemecdn.com/youthlql@1.0.8/JVM/chapter_009/0016.png">





**练习2**

```java
public class StringExer1 {
    public static void main(String[] args) {
        //加一行这个
        String x = "ab";
        String s = new String("a") + new String("b");//new String("ab")

        String s2 = s.intern();

        System.out.println(s2 == "ab");//jdk6:true  jdk8:true
        System.out.println(s == "ab");//jdk6:false  jdk8:true
    }
}
```

<img src="https://npm.elemecdn.com/youthlql@1.0.8/JVM/chapter_009/0017.png">

**练习3**

```java
public class StringExer2 {
    public static void main(String[] args) {
        String s1 = new String("ab");//执行完以后，会在字符串常量池中会生成"ab"

        s1.intern();
        String s2 = "ab";
        System.out.println(s1 == s2);//false
    }
}
```



**验证**

```java
public class StringExer2 {
    // 对象内存地址可以使用System.identityHashCode(object)方法获取
    public static void main(String[] args) {
        String s1 = new String("a") + new String("b");//执行完以后，不会在字符串常量池中会生成"ab"
        System.out.println(System.identityHashCode(s1));
        s1.intern();
        System.out.println(System.identityHashCode(s1));
        String s2 = "ab";
        System.out.println(System.identityHashCode(s2));
        System.out.println(s1 == s2); // true
    }
}
```

输出结果：

```java
1836019240
1836019240
1836019240
true
```



### intern() 的效率测试（空间角度）

```java
/**
 * 使用intern()测试执行效率：空间使用上
 *
 * 结论：对于程序中大量存在存在的字符串，尤其其中存在很多重复字符串时，使用intern()可以节省内存空间。
 *
 */
public class StringIntern2 {
    static final int MAX_COUNT = 1000 * 10000;
    static final String[] arr = new String[MAX_COUNT];

    public static void main(String[] args) {
        Integer[] data = new Integer[]{1,2,3,4,5,6,7,8,9,10};

        long start = System.currentTimeMillis();
        for (int i = 0; i < MAX_COUNT; i++) {
//            arr[i] = new String(String.valueOf(data[i % data.length]));
            arr[i] = new String(String.valueOf(data[i % data.length])).intern();

        }
        long end = System.currentTimeMillis();
        System.out.println("花费的时间为：" + (end - start));

        try {
            Thread.sleep(1000000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.gc();
    }
}
```



1、直接 new String ：由于每个 String 对象都是 new 出来的，所以程序需要维护大量存放在堆空间中的 String 实例，程序内存占用也会变高

```java
arr[i] = new String(String.valueOf(data[i % data.length]));
```

<img src="https://npm.elemecdn.com/youthlql@1.0.8/JVM/chapter_009/0018.png">



<img src="https://npm.elemecdn.com/youthlql@1.0.8/JVM/chapter_009/0019.png">

2、使用 intern() 方法：由于数组中字符串的引用都指向字符串常量池中的字符串，所以程序需要维护的 String 对象更少，内存占用也更低

```java
//调用了intern()方法使用了字符串常量池里的字符串，那么前面堆里的字符串便会被GC掉，这也是intern省内存的关键原因
arr[i] = new String(String.valueOf(data[i % data.length])).intern();
```

<img src="https://npm.elemecdn.com/youthlql@1.0.8/JVM/chapter_009/0020.png" >



<img src="https://npm.elemecdn.com/youthlql@1.0.8/JVM/chapter_009/0021.png">



**结论**：

1.  对于程序中大量使用存在的字符串时，尤其存在很多已经重复的字符串时，使用intern()方法能够节省很大的内存空间。
2.  大的网站平台，需要内存中存储大量的字符串。比如社交网站，很多人都存储：北京市、海淀区等信息。这时候如果字符串都调用intern() 方法，就会很明显降低内存的大小。

## StringTable 的垃圾回收

```java
/**
 * String的垃圾回收:
 * -Xms15m -Xmx15m -XX:+PrintStringTableStatistics -XX:+PrintGCDetails
 */
public class StringGCTest {
    public static void main(String[] args) {
        for (int j = 0; j < 100000; j++) {
            String.valueOf(j).intern();
        }
    }
}
```



输出结果：
*   在 PSYoungGen 区发生了垃圾回收
*   Number of entries 和 Number of literals 明显没有 100000
*   以上两点均说明 StringTable 区发生了垃圾回收

<img src="https://npm.elemecdn.com/youthlql@1.0.8/JVM/chapter_009/0022.jpg">



<img src="https://npm.elemecdn.com/youthlql@1.0.8/JVM/chapter_009/0023.jpg"/>



G1 中的 String 去重操作
-------------------

> **官方文档**：http://openjdk.java.net/jeps/192

暂时了解一下，后面会详解垃圾回收器

**String去重操作的背景**

> 注意不是字符串常量池的去重操作，字符串常量池本身就没有重复的

1.  背景：对许多Java应用（有大的也有小的）做的测试得出以下结果：
    *   堆存活数据集合里面String对象占了25%
    *   堆存活数据集合里面重复的String对象有13.5%
    *   String对象的平均长度是45
2.  许多大规模的Java应用的瓶颈在于内存，测试表明，在这些类型的应用里面，Java堆中存活的数据集合差不多25%是String对象。更进一步，这里面差不多一半String对象是重复的，重复的意思是说：`str1.equals(str2)= true`。堆上存在重复的String对象必然是一种内存的浪费。这个项目将在G1垃圾收集器中实现自动持续对重复的String对象进行去重，这样就能避免浪费内存。



**String 去重的的实现**

1.  当垃圾收集器工作的时候，会访问堆上存活的对象。对每一个访问的对象都会检查是否是候选的要去重的String对象。
2.  如果是，把这个对象的一个引用插入到队列中等待后续的处理。一个去重的线程在后台运行，处理这个队列。处理队列的一个元素意味着从队列删除这个元素，然后尝试去重它引用的String对象。
3.  使用一个Hashtable来记录所有的被String对象使用的不重复的char数组。当去重的时候，会查这个Hashtable，来看堆上是否已经存在一个一模一样的char数组。
4.  如果存在，String对象会被调整引用那个数组，释放对原来的数组的引用，最终会被垃圾收集器回收掉。
5.  如果查找失败，char数组会被插入到Hashtable，这样以后的时候就可以共享这个数组了。



**命令行选项**

1.  UseStringDeduplication(bool) ：开启String去重，默认是不开启的，需要手动开启。
2.  PrintStringDeduplicationStatistics(bool) ：打印详细的去重统计信息
3.  stringDeduplicationAgeThreshold(uintx) ：达到这个年龄的String对象被认为是去重的候选对象