---
title: HashMap-JDK8源码讲解及常见面试题
tags:
  - Java集合
  - HashMap
categories:
  - Java集合
  - HashMap
keywords: Java集合，HashMap。
description: HashMap-JDK8源码讲解及常见面试题。
cover: 'https://npm.elemecdn.com/lql_static@latest/logo/java.png'
abbrlink: cbc5672a
date: 2020-11-01 10:22:05
---







> JDK7说过的东西，本篇文章不再讲解

# 数据结构

## 红黑树

在JDK8中，优化了HashMap的数据结构，引入了红黑树。即HashMap的数据结构：数组+链表+红黑树。HashMap变成了这样。

<img src="https://npm.elemecdn.com/youthlql@1.0.8/Java_collection/HashMap/JDK8/0001.png">

### 为什么要引入红黑树

1、主要是为了提高HashMap的性能，即解决发生hash冲突后，因为链表过长而导致索引效率慢的问题

2、链表的索引速度是O(n)，而利用了红黑树快速增删改查的特点，时间复杂度就是O(logn)。



## Node类

`HashMap`中的数组元素，链表节点均采用`Node`类 实现，与 `JDK 1.7` 的对比（`Entry`类），仅仅只是换了名字。

就是一些常规的方法

```java
/** 
  * Node  = HashMap的内部类，实现了Map.Entry接口，本质是 = 一个映射(键值对)
  * 实现了getKey()、getValue()、equals(Object o)和hashCode()等方法
  **/  
  static class Node<K,V> implements Map.Entry<K,V> {

        final int hash; 
        final K key; 
        V value; 
        Node<K,V> next;

        // 构造方法
        Node(int hash, K key, V value, Node<K,V> next) {
            this.hash = hash;
            this.key = key;
            this.value = value;
            this.next = next;
        }
        
        public final K getKey()        { return key; }  
        public final V getValue()      { return value; } 
        public final String toString() { return key + "=" + value; }

        public final V setValue(V newValue) {
            V oldValue = value;
            value = newValue;
            return oldValue;
        }

        public final int hashCode() {
            return Objects.hashCode(key) ^ Objects.hashCode(value);
        }

        public final boolean equals(Object o) {
            if (o == this)
                return true;
            if (o instanceof Map.Entry) {
                Map.Entry<?,?> e = (Map.Entry<?,?>)o;
                if (Objects.equals(key, e.getKey()) &&
                    Objects.equals(value, e.getValue()))
                    return true;
            }
            return false;
        }
    }

```



## TreeNode类

```java

  static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V> {  

  	// 属性 = 父节点、左子树、右子树、删除辅助节点 + 颜色
    TreeNode<K,V> parent;  
    TreeNode<K,V> left;   
    TreeNode<K,V> right;
    TreeNode<K,V> prev;   
    boolean red;   

    // 构造函数
    TreeNode(int hash, K key, V val, Node<K,V> next) {  
        super(hash, key, val, next);  
    }  
  
    // 返回当前节点的根节点  
    final TreeNode<K,V> root() {  
        for (TreeNode<K,V> r = this, p;;) {  
            if ((p = r.parent) == null)  
                return r;  
            r = p;  
        }  
    } 

```



## 重要参数

> JDK7里讲过的就不再讲了

```java
   static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; 
  
   static final int MAXIMUM_CAPACITY = 1 << 30; 

   final float loadFactor; 
  
   static final float DEFAULT_LOAD_FACTOR = 0.75f; 

   int threshold;

  // 存储数据的Node类型 数组，长度 = 2的幂；
   transient Node<K,V>[] table;  
   transient int size;
   
   //与红黑树相关的参数
   
   /*
   1、单链表(桶)的树化阈值：即 链表转成红黑树的阈值，在存储数据时，当链表长度 > 该值时，
   则将链表转换成红黑树
   */
   static final int TREEIFY_THRESHOLD = 8; 

	/*
   1、桶的链表还原阈值：即 红黑树转为链表的阈值，当在扩容（resize（））时（此时HashMap的数据
   存储位置会重新计算），在重新计算存储位置后，当原有的红黑树内节点数量 < 6时，则将 红黑树转换
   成链表
	*/
   static final int UNTREEIFY_THRESHOLD = 6;

   /*
   1、最小树形化容量阈值：即 当哈希表中的容量 > 该值时，才允许树形化链表 （即 将链表 转换成红黑树）。
   否则，若 （单链表）桶内元素太多时，则直接扩容，而不是树形化。
   2、为了避免进行扩容、树形化选择的冲突，这个值不能小于 4 * TREEIFY_THRESHOLD
   */
   static final int MIN_TREEIFY_CAPACITY = 64;
  

```



# 构造函数源码

```java

public class HashMap<K,V>
    extends AbstractMap<K,V>
    implements Map<K,V>, Cloneable, Serializable{


    public HashMap() {
        this.loadFactor = DEFAULT_LOAD_FACTOR;
    }

    public HashMap(int initialCapacity) {
        this(initialCapacity, DEFAULT_LOAD_FACTOR);
        
    }

    /**
     * 构造函数3：指定"容量大小"和"加载因子"的构造函数
     * 加载因子和容量由自己指定
     */
    public HashMap(int initialCapacity, float loadFactor) {

    	// 指定初始容量必须非负，否则报错  
   		 if (initialCapacity < 0)  
           throw new IllegalArgumentException("Illegal initial capacity: " +  
                                           initialCapacity); 

        // HashMap的最大容量只能是MAXIMUM_CAPACITY，哪怕传入的 > 最大容量
        if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;

        // 填充比必须为正  
    	if (loadFactor <= 0 || Float.isNaN(loadFactor))  
        	throw new IllegalArgumentException("Illegal load factor: " +  
                                           loadFactor);  
        // 设置加载因子
        this.loadFactor = loadFactor;

    
        /*
        1、设置扩容阈值
        2、此处不是真正的阈值，仅仅只是将传入的容量大小转化为：>传入容量大小的最小的2的幂，
        该阈值后面会重新计算
        */
        this.threshold = tableSizeFor(initialCapacity); 

    }

    public HashMap(Map<? extends K, ? extends V> m) {
        this.loadFactor = DEFAULT_LOAD_FACTOR; 

        // 将传入的子Map中的全部元素逐个添加到HashMap中
        putMapEntries(m, false); 
    }
}

```



## tableSizeFor()

```java
  /**
     * 作用：将传入的容量大小转化为：>传入容量大小的最小的2的幂
     * 与JDK 1.7对比：类似于JDK 1.7 中 inflateTable()里的 roundUpToPowerOf2(toSize)
     */
    static final int tableSizeFor(int cap) {
     int n = cap - 1;
     n |= n >>> 1;
     n |= n >>> 2;
     n |= n >>> 4;
     n |= n >>> 8;
     n |= n >>> 16;
     return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}
```



```java

public class test {

    public static void main(String[] args) {
        int n = 65538;  //这个数字是2^16 + 2
        System.out.println("开始:" + Integer.toBinaryString(n));
        int res = tableSizeFor(n);
        System.out.println("最终结果:" + res);
    }


    static final int MAXIMUM_CAPACITY = 1 << 30;

    static final int tableSizeFor(int cap) {
        int n = cap - 1;
        n |= n >>> 1;
        System.out.println(Integer.toBinaryString(n));
        n |= n >>> 2;
        System.out.println(Integer.toBinaryString(n));
        n |= n >>> 4;
        System.out.println(Integer.toBinaryString(n));
        n |= n >>> 8;
        System.out.println(Integer.toBinaryString(n));
        n |= n >>> 16;
        System.out.println(Integer.toBinaryString(n));
        return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
    }
}

```

输出结果：

```
开始:10000000000000010
11000000000000001
11110000000000001
11111111000000001
11111111111111111
11111111111111111
最终结果:131072

Process finished with exit code 0
```

**第一次运行：**
    10000000000000010        n >>> 1;
    01000000000000000        进行|运算
    11000000000000001
分析：
    把最大位的1，通过位移后移一位，并且通过|运算，组合起来

**第二次运行：**
    11000000000000001        n >>> 2;
    00110000000000000        进行|运算
    11110000000000001
分析：
    把最大的两位，已经变成1的，往后移动两位，并且通过|运算，组合起来

**第三次运行：**
    11110000000000001        n >>> 4;
    00001111000000000        进行|运算
    11111111000000001
分析：
    把最大4位，已经变成1的，往后移动4位，并且通过|运算，组合起来

**第四次运行：**
    11111111000000001        n >>> 8;
    00000000111111110        进行|运算
    11111111111111111
分析：
    把最大的8位，已经变成1的，往后移动8位，并且通过|运算，组合起来

**第五次运算：**
    同上。因为我的数据，最大只到17位，所有第五次没有效果。可以用32位来进行运算，第五次是通过前16位已经变成1的数据，往后移动16位，然后通过或运算，最后的结果是32位都变成1。

> 原理就是，保证造成一个所有位都为1的数据。并且通过最后的+1。变成2^N次方的数据。



# put源码

```java
	public V put(K key, V value) {
        //在第一个参数里就直接计算出了hash值
        return putVal(hash(key), key, value, false, true);
    }
```



```java
	final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        
        
        /*
        1、若哈希表的数组tab为空，则通过resize()进行初始化，所以，初始化哈希表的时机就是第1次
        调用put函数时，即调用resize() 初始化创建。
        */
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
        
        /* if分支
        1、根据键值key计算的hash值，计算插入存储的数组索引i
    	2、插入时，需判断是否存在Hash冲突：
    	  2-1、若不存在（即当前table[i] == null），则直接在该数组位置新建节点，插入完毕。
    	  2-2、否则代表发生hash冲突，进入else分支
        */
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
        
        else {
            Node<K,V> e; K k;
           //判断 table[i]的元素的key是否与需插入的key一样，若相同则直接用新value覆盖旧value
            //【即更新操作】
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
            
            //继续判断：需插入的数据结构是否为红黑树or链表。若是红黑树，则直接在树中插入or更新键值对     
            else if (p instanceof TreeNode)
                /*
                1、putTreeVal作用：向红黑树插入 or 更新数据（键值对）
      			2、过程：遍历红黑树判断该节点的key是否与需插入的key是否相同：
           		   2-1、若相同，则新value覆盖旧value
           		   2-2、若不相同，则插入
                */
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            
            //进入到这个分支说明是链表节点
            else {
                /*
                过程：
                1、遍历table[i]，判断Key是否已存在：采用equals()对比当前遍历节点的key 与
                需插入数据的key：若已存在，则直接用新value覆盖旧value
       		   2、遍历完毕后仍无发现上述情况，则直接在链表尾部插入数据(尾插法)
       		   3、新增节点后，需判断链表长度是否>8（8 = 桶的树化阈值）：若是，则把链表转换为红黑树
                */
                for (int binCount = 0; ; ++binCount) {
                    //对于2情况的操作  尾插法插入尾部
                    if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);
                        //对于3情况的操作
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);
                        break;
                    }
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }
            // 对1情况的后续操作：发现key已存在，直接用新value 覆盖 旧value，返回旧value
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
        }
        ++modCount;
        // 插入成功后，判断实际存在的键值对数量size > threshold
        if (++size > threshold)
            resize();
        afterNodeInsertion(evict);
        return null;
    }
```



## hash()

```java

      //JDK7实现:使用hashCode() + 4次位运算 + 5次异或运算（9次扰动）
     static final int hash(int h) {
        h ^= k.hashCode(); 
        h ^= (h >>> 20) ^ (h >>> 12);
        return h ^ (h >>> 7) ^ (h >>> 4);
     }

      //JDK8实现: 使用hashCode() + 1次位运算 + 1次异或运算（2次扰动） 
      static final int hash(Object key) {
           int h;
          /*
          1、当key = null时，hash值 = 0，所以HashMap的key可为null      
	      2、当key ≠ null时，则通过先计算出 key的 hashCode()（记为h），然后对哈希码进行扰动处理。
	      高位参与低位的运算：h ^ (h >>> 16) 
          */
          return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
            
     }

```

JDK8 hash的运算原理：高位参与低位运算，使得hash更加均匀。

<img src="https://npm.elemecdn.com/youthlql@1.0.8/Java_collection/HashMap/JDK8/0002.png">





## resize()

这个方法改动比较大

```java
   
   //该函数有2种使用情况：1、初始化哈希表 2、当前数组容量过小，需扩容   
   final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table; // 扩容前的数组（当前数组）
    int oldCap = (oldTab == null) ? 0 : oldTab.length; // 扩容前的数组的容量
    int oldThr = threshold;// 扩容前的数组的阈值
    int newCap, newThr = 0;

    // 针对情况2：若扩容前的数组容量超过最大值，则不再扩充
    if (oldCap > 0) {
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }

        // 针对情况2：若无超过最大值，就扩充为原来的2倍
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                 oldCap >= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr << 1; // 通过右移扩充2倍
    }

    // 针对情况1：初始化哈希表（采用指定值或者默认值）
    else if (oldThr > 0) 
        newCap = oldThr;

    else {  
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }

    // 计算新的扩容阈值
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                  (int)ft : Integer.MAX_VALUE);
    }

    threshold = newThr;
    @SuppressWarnings({"rawtypes","unchecked"})
        Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
	
    //旧数组数据移动到新数组里，整体过程也是遍历旧数组每个数据
    if (oldTab != null) {
        // 把每个bucket都移动到新的buckets中
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null;

                if (e.next == null)
                    newTab[e.hash & (newCap - 1)] = e;
                else if (e instanceof TreeNode)
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);

                else { // 链表优化重hash的代码块
                    Node<K,V> loHead = null, loTail = null;
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    //这个待会细讲
                    do {
                        next = e.next;
                        //原索引
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        // 原索引 + oldCap
                        else {
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    // 原索引放到bucket里
                    if (loTail != null) {
                        loTail.next = null;
                        newTab[j] = loHead;
                    }
                    // 原索引+oldCap放到bucket里
                    if (hiTail != null) {
                        hiTail.next = null;
                        newTab[j + oldCap] = hiHead;
                    }
                }
            }
        }
    }
    return newTab;
}

```



JDK8扩容时，数据在数组下标的计算方式

<img src="https://npm.elemecdn.com/youthlql@1.0.8/Java_collection/HashMap/JDK8/0003.png">

* `JDK8`根据此结论作出的新元素存储位置计算规则非常简单，提高了扩容效率。

- 这与 `JDK7`在计算新元素的存储位置有很大区别：`JDK7`在扩容后，都需按照原来方法进行rehash，效率不高。



# get源码

```java

   public V get(Object key) {
    Node<K,V> e;
    // 计算需获取数据的hash值,通过getNode（）获取所查询的数据,获取后，判断数据是否为空
    return (e = getNode(hash(key), key)) == null ? null : e.value;
	}


final Node<K,V> getNode(int hash, Object key) {
    Node<K,V>[] tab; Node<K,V> first, e; int n; K k;

    //计算存放在数组table中的位置
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (first = tab[(n - 1) & hash]) != null) {

        // 先在数组中找，若存在，则直接返回
        if (first.hash == hash && // always check first node
            ((k = first.key) == key || (key != null && key.equals(k))))
            return first;

        //若数组中没有，则到红黑树中寻找
        if ((e = first.next) != null) {
            // 在树中get
            if (first instanceof TreeNode)
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);

            //若红黑树中也没有，则通过遍历，到链表中寻找
            do {
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    return e;
            } while ((e = e.next) != null);
        }
    }
    return null;
}
```





# ---下面是常见面试题---

# HashMap在JDK7和8中区别？

1、hash冲突时：JDK7用的是头插法，而JDK1.8及之后使用的都是尾插法。JDK7是用单链表进行的纵向延伸，当采用头插法时会容易出现逆序且环形链表死循环问题。但是在JDK8之后是使用尾插法，能够避免出现逆序且链表死循环的问题。

2、扩容时：JDK7需要重新进行rehash。JDK8则直接时判断hash值新参与的位是0还是1，0就是原位置，1就是原位置+就容量

3、引入了红黑树（原因前面说过）

4、hash的计算：JDK7是9次扰动（4次位运算 + 5次异或运算），JDK8时是2次扰动（1次位运算 + 1次异或运算）。

5、JDK7是先扩容再插入k-v，JDK8时是插入后一起扩容。

# 为什么不直接用hash码作为数组table的下标？

1、哈希码一般是int型，范围是-(2^31) -- 2^31 - 1。容易出现哈希码与数组大小范围不匹配的情况，即计算出来的哈希码可能不在数组大小范围内，从而导致无法匹配存储位置。

2、常见解决办法就是hash值与数组长度取模。

# 为什么容量要求为2的幂？

一般来说散列表容量的常规设计思路是容量取素数，因为素数导致冲突的概率 < 合数。比如Hashtable初始化容量就是11（不过扩容后不能保证是素数）

**hashmap这样设计的原因是**

1、保证哈希码的均匀性。首先容量可为奇数，也可为偶数。假设数组长度为奇数，那么二进制最后一位是1。假设数组长度为偶数，那么二进制最后一位是0。如果是奇数  hash&(length - 1) 铁定是偶数，就会导致浪费了数组的一半位置（奇数索引无法被放数据，hash冲突概率高）。如果是2的幂这种偶数，length - 1就是奇数，那么最终的hash&(length-1)计算出来的索引位置取决于hash值，也就是说可以是偶数索引，也可以是奇数索引，均匀分布。

2、length是2的幂时 hash&(length - 1)等价于hash % length。但是&效率更高，而只有length是2的幂，这两个才等价。



# 二次扰动的好处

高位充分参与低位运算，加大哈希码低位的随机性，使得分布更均匀，从而提高对应数组存储下标位置的随机性 & 均匀性，最终减少Hash冲突



# 什么样类型的数据适合做hashmap的key?

像Integer这种，内部属性value被final修饰，保证了Hash值的不可更改性，有效的减少了hash冲突



# 为什么选择8作为树化阈值？

```java
	//Java8代码官方解释的原因
   * Because TreeNodes are about twice the size of regular nodes, we
     * use them only when bins contain enough nodes to warrant use
     * (see TREEIFY_THRESHOLD). And when they become too small (due to
     * removal or resizing) they are converted back to plain bins.  In
     * usages with well-distributed user hashCodes, tree bins are
     * rarely used.  Ideally, under random hashCodes, the frequency of
     * nodes in bins follows a Poisson distribution
     * (http://en.wikipedia.org/wiki/Poisson_distribution) with a
     * parameter of about 0.5 on average for the default resizing
     * threshold of 0.75, although with a large variance because of
     * resizing granularity. Ignoring variance, the expected
     * occurrences of list size k are (exp(-0.5) * pow(0.5, k) /
     * factorial(k)). The first values are:
     *
     * 0:    0.60653066
     * 1:    0.30326533
     * 2:    0.07581633
     * 3:    0.01263606
     * 4:    0.00157952
     * 5:    0.00015795
     * 6:    0.00001316
     * 7:    0.00000094
     * 8:    0.00000006
     * more: less than 1 in ten million

```

由于treenodes的大小大约是常规节点的两倍，因此我们仅在容器包含足够的节点以保证使用时才使用它们，当它们变得太小（由于移除或调整大小）时，它们会被转换回普通的node节点，容器中节点分布在hash桶中的频率遵循泊松分布，桶的长度超过8的概率非常非常小，作者是根据概率统计而选择了8作为阀值。



# 为什么选择6和8作为链表化和树化的阈值?

1、首先就是遵循泊松分布概率选了6和8

2、其次：如果选择6和8（如果链表小于等于6树还原转为链表，大于等于8转为树），中间有个差值7可以有效防止链表和树频繁转换。假设一下，如果设计成链表个数超过8则链表转换成树结构，链表个数小于8则树结构转换成链表，如果一个HashMap不停的插入、删除元素，链表个数在8左右徘徊，就会频繁的发生树转链表、链表转树，效率会很低。