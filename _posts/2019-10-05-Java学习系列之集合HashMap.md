---
layout: post
title:  "Java学习系列之集合HashMap"
categories: Java
tags:  Java 集合
---

* content
{:toc}

本文系记录对Java中集合类HashMap的学习资料，如有异议，欢迎联系我讨论修改。PS:图侵删！如果看不清图请访问https://github.com/zx950519/zx950519.github.io/blob/master/_posts/2019-10-05-Java%E5%AD%A6%E4%B9%A0%E7%B3%BB%E5%88%97%E4%B9%8B%E9%9B%86%E5%90%88.md






- [1 哈希](#1-哈希)
  - [1.1 简述对HashMap的理解](#11-简述对hashmap的理解)
  - [1.2 HashMap的源码设计特性](#12-hashmap的源码设计特性)
  - [1.3 HashMap中常见方法的源码分析](#13-hashmap中常见方法的源码分析)
  - [1.4 HashMap中resize在多线程环境下的潜在危害](#14-hashmap中resize在多线程环境下的潜在危害)版本区别
  - [1.5 HashMap在不同JDK版本下的区别](#15-hashmap在不同JDK版本下的区别)
  - [1.5 HashMap存在的问题](#15-hashmap存在的问题)
  - [1.6 和其他集合类的比较](#16-和其他集合类的比较)





# 1 哈希

## 1.1 简述对HashMap的理解
HashMap 主要用来存放键值对，它基于哈希表的Map接口实现，是常用的Java集合之一。JDK1.8 之前 HashMap 由数组+链表组成的，数组是HashMap的主体，链表则是主要为了解决哈希冲突而存在的（“拉链法”解决冲突）。JDK1.8 以后在解决哈希冲突时有了较大的变化，当链表长度大于阈值（默认为 8）时，将链表转化为红黑树，以减少搜索时间。

## 1.2 HashMap的源码设计特性

## HashMap的几个构造函数
HashMap默认不对Table[]进行初始化，而是采用延迟的思想，当真正put时才会涉及扩容操作。  
- 不传任何值：仅把负载因子设定为默认负载因子
- 传一个Map类的参数：把负载因子设定为默认负载因子并调用putMapEntries(Map<? extends K, ? extends V> m, boolean evict)执行复制
- 传一个capacity值：调用下面的构造函数
- 传一个capacity值与loadFactor：对阈值与负载因子进行初始化

## HashMap的长度为什么是2的幂次方
为了能让 HashMap 存取高效，尽量较少碰撞，也就是要尽量把数据分配均匀。我们上面也讲到了过了，Hash 值的范围值-2147483648到2147483647，前后加起来大概40亿的映射空间，只要哈希函数映射得比较均匀松散，一般应用是很难出现碰撞的。但问题是一个40亿长度的数组，内存是放不下的。所以这个散列值是不能直接拿来用的。用之前还要先做对数组的长度取模运算，得到的余数才能用来要存放的位置也就是对应的数组下标。这个数组下标的计算方法是“ (n - 1) & hash”。（n代表数组长度）。取余(%)操作中如果除数是2的幂次则等价于与其除数减一的与(&)操作（也就是说 hash%length==hash&(length-1)的前提是 length 是2的 n 次方；）。并且 采用二进制位操作 &，相对于%能够提高运算效率，这就解释了 HashMap 的长度为什么是2的幂次方。

## HashMap为何链表长度到8才扩展为红黑树
当HashCode的离散性强时，使用树形bin的概率极低，官方在源码给出如下数据：  
```java
0:    0.60653066
1:    0.30326533
2:    0.07581633
3:    0.01263606
4:    0.00157952
5:    0.00015795
6:    0.00001316
7:    0.00000094
8:    0.00000006
```
可以看到，一个bin中链表长度达到8个元素的概率为0.00000006，几乎是不可能事件。之所以选择8是根据概率统计决定。  

##  HashMap如何保证其容量为2的n次方
先考虑如何求一个数的掩码，对于 10010000，它的掩码为 11111111，可以使用以下方法得到：  
```xml
mask |= mask >> 1    11011000
mask |= mask >> 2    11111110
mask |= mask >> 4    11111111
```
mask+1 是大于原始数字的最小的 2 的 n 次方。  
```xml
num     10010000
mask+1 100000000
```
```java
static final int tableSizeFor(int cap) {
    int n = cap - 1;    // 减一是因为该数本身就是结果
    n |= n >>> 1;
    n |= n >>> 2;
    n |= n >>> 4;
    n |= n >>> 8;
    n |= n >>> 16;
    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}
```
## 为什么HashMap中String、Integer这样的包装类适合作为key键
![](https://ws1.sinaimg.cn/large/005L0VzSgy1g4j93h03b8j31e80f8wh2.jpg)

## HashMap中的key若Object类型<则需实现哪些方法？
![](https://ws1.sinaimg.cn/large/005L0VzSgy1g4j94446j0j31ii0k00v5.jpg)


## 1.3 HashMap中常见方法的源码分析

## 简述HashMap的Put流程
![](https://ws1.sinaimg.cn/large/005L0VzSgy1g4j87ukb90j30k00fvq53.jpg)
1.8版本以前的流程：  
- 如果定位到的数组位置没有元素就直接插入
- 如果定位到的数组位置有元素，遍历以这个元素为头结点的链表，依次和插入的key比较，如果key相同就直接覆盖，不同就采用头插法插入元素  

值得注意的是，在put的执行流程中，当判断key是否存在时，使用了如下的判断方法：  
```java
if (e.hash == hash && ((k = e.key) == key || (key != null && key.equals(k))))
    break;
```
完整代码如下：  
```java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;       // 定义put操作中需要的一些变量
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;    // 记录扩容后的table数组大小
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);   // 如果头结点为空直接插入，然后结束
    else {
        Node<K,V> e; K k;
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;    // 头结点刚好为所求
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);   // 在树形结构中搜索
        else {
            for (int binCount = 0; ; ++binCount) {
                if ((e = p.next) == null) {   // 遍历到表尾，仍未找到该key，新建
                    p.next = newNode(hash, key, value, null);
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
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount;
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;
}
```

## 简述get()流程
![](https://ws1.sinaimg.cn/large/005L0VzSgy1g51p6wjsnkj30u0140417.jpg)  
完整代码如下：  
```java
final Node<K,V> getNode(int hash, Object key) {
    Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (first = tab[(n - 1) & hash]) != null) {
        if (first.hash == hash && 
            ((k = first.key) == key || (key != null && key.equals(k)))) // 先检查头结点
            return first;
        if ((e = first.next) != null) {   // 判断链表是否只有一个元素，否则分别按照树形和链表进行遍历查找
            if (first instanceof TreeNode)
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);
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

## 简述resize()的过程
resize方法是HashMap中进行扩容的方法。扩容操作会带来极大的开销，因为会伴随着一次重新Hash分配，并遍历Hash表中的所有元素。在实际使用过程中要尽力避免。  
![](https://ws1.sinaimg.cn/large/005L0VzSgy1g51psi7z9qj30u0140whf.jpg)  

这里描述下链表的迁移过程：  
```java
// 下面这段就是把原来table里面的值全部搬到新的table里面
    if (oldTab != null) {
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            if ((e = oldTab[j]) != null) {
                // 这里注意, table中存放的只是Node的引用, 这里将oldTab[j]=null只是清除旧表的引用, 但是真正的node节点还在, 只是现在由e指向它
                oldTab[j] = null;
                // 如果该存储桶里面只有一个bin, 就直接将它放到新表的目标位置
                if (e.next == null)
                    newTab[e.hash & (newCap - 1)] = e;
                // 如果该存储桶里面存的是红黑树, 则拆分树
                else if (e instanceof TreeNode)
                    //红黑树的部分以后有机会再讲吧
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                // 下面这段代码很精妙, 我们单独分一段详细来讲
                else { // preserve order
                    Node<K,V> loHead = null, loTail = null;
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    do {
                        next = e.next;
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        else {
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    if (loTail != null) {
                        loTail.next = null;
                        newTab[j] = loHead;
                    }
                    if (hiTail != null) {
                        hiTail.next = null;
                        newTab[j + oldCap] = hiHead;
                    }
                }
            }
        }
    }
```
首先，定义了两个新的链表分别是：lo链表与hi链表，其中loHead与hiHead指向头部，loTail与hiTail指向尾部。然后，从旧table的某个桶的头部开始，逐一将桶内链表上的每一个节点迁移到lo链表或hi链表中。决定进入lo链表或hi链表的依据是判断条件：(e.hash & oldCap) == 0。下面解释下为什么采用这种判断依据。首先，oldCap的大小必然是2的n次幂，newCap的大小必然是2的n+1次幂，在求桶的下标时使用公式(capacity-1)&hash，这个公式本质上取出了hash值的低n为，这个n对应于n次幂。然而，当迁移时需要计算在新table下的桶下标，这是使用的公式是(2*capacity-1)&hash，这样就会比原来多取出一位，而二进制只有0/1，这也与两个新链表hi/lo对应。(e.hash & oldCap) == 0时，这个节点应该被丢进lo链表，否则进入hi链表。在将原table上某个桶的一个链表全部拆完后，将这两个链表分别链接到新table对应的桶中，而这两个桶下标间存在一种数值上的关系，即二者差为一个原来的table长度——oldCap。  

![](https://ws1.sinaimg.cn/large/005L0VzSgy1g52w0hupuaj30m80aj76o.jpg)

## 1.4 简述HashMap中resize在多线程环境下的潜在危害

#### fail-fast
如果在使用迭代器的过程中有其他线程修改了map，那么将抛出ConcurrentModificationException，这就是所谓fail-fast策略。java.util.HashMap线程不安全，因此如果在使用迭代器的过程中有其他线程修改了map，那么将抛出ConcurrentModificationException，这就是所谓fail-fast策略。在HashMap中，有一个变量modCount来指示集合被修改的次数。在创建Iterator迭代器的时候，会给这个变量赋值给expectedModCount。当集合方法修改集合元素时，例如集合的remove()方法时，此时会修改modCount值，但不会同步修改expectedModCount值。当使用迭代器遍历元素操作时，会首先对比expectedModCount与modCount是否相等。如果不相等，则马上抛出java.util.ConcurrentModificationException异常。而通过Iterator的remove()方法移除元素时，会同时更新expectedModCount的值，将modCount的值重新赋值给expectedModCount，这样下一次遍历时，就不会发抛出java.util.ConcurrentModificationException异常。上述机制可以解释为什么HashMap通过迭代器自身的remove或add方法就不会出现迭代器失败。

#### 死链
当需要调整HashMap的大小时，如果有多个线程同时尝试调整大小，可能会导致节点间的引用构成循环链而造成死循环。  

https://www.cnblogs.com/wang-meng/p/7582532.html

![image](https://ws1.sinaimg.cn/large/005L0VzSly1g77dwat8ubj30h40c2q3r.jpg)  

![image](https://ws1.sinaimg.cn/large/005L0VzSly1g77dwswevbj30wo0mk0u5.jpg)  

![image](https://ws1.sinaimg.cn/large/005L0VzSly1g77dx2apwzj30uk0kwta2.jpg)  

![](https://ws1.sinaimg.cn/large/005L0VzSgy1g52mg06vshj30rs0ckt8s.jpg)  
假设有两个线程同时需要执行resize操作，我们原来的桶数量为2，记录数为3，需要resize桶到4，原来的记录分别为：[3,A],[7,B],[5,C]，在原来的map里面，我们发现这三个entry都落到了第二个桶里面。假设线程thread1执行到了transfer方法的Entry next=e.next这一句，然后时间片用完了，此时的e = [3,A], next = [7,B]。线程thread2被调度执行并且顺利完成了resize操作，需要注意的是，此时的[7,B]的next为[3,A]。此时线程thread1重新被调度运行，此时的thread1持有的引用是已经被thread2 resize之后的结果。线程thread1首先将[3,A]迁移到新的数组上，然后再处理[7,B]，而[7,B]被链接到了[3,A]的后面，处理完[7,B]之后，就需要处理[7,B]的next了啊，而通过thread2的resize之后，[7,B]的next变为了[3,A]，此时，[3,A]和[7,B]形成了环形链表，在get的时候，如果get的key的桶索引和[3,A]和[7,B]一样，那么就会陷入死循环。

## 1.5 HashMap在不同JDK版本下的区别

## 1.8版本与1.8版本前HashMap在hash(key)的区别
1.8版本前：  
```java
static int hash(int h) {
    // This function ensures that hashCodes that differ only by
    // constant multiples at each bit position have a bounded
    // number of collisions (approximately 8 at default load factor).
    h ^= (h >>> 20) ^ (h >>> 12);
    return h ^ (h >>> 7) ^ (h >>> 4);
}
```
1.8版本
```java
static final int hash(Object key) {
    int h;
    // key.hashCode()：返回散列值也就是hashcode
    // ^ ：按位异或
    // >>>:无符号右移，忽略符号位，空位都以0补齐
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```
相比于 JDK1.8 的 hash 方法 ，JDK 1.7 的 hash 方法的性能会稍差一点点，因为毕竟扰动了 4 次。

## HashMap在1.8和1.7的区别
- 1.7插入节点采用头插法；1.8采用尾插法
- hash(key)的计算方式不同，1.8性能较强
- 1.7是数组+链表；1.8是数组+链表+红黑树

## HashMap如何解决Hash冲突
![](https://ws1.sinaimg.cn/large/005L0VzSgy1g4j8zuq19xj31do0run1e.jpg)


## 1.6 和其他集合类的比较

## HashMap和HashTable的区别
- 线程是否安全： HashMap 是非线程安全的，HashTable 是线程安全的；HashTable 内部的方法基本都经过synchronized 修饰。（如果你要保证线程安全的话就使用 ConcurrentHashMap 吧！）；
- 效率： 因为线程安全的问题，HashMap 要比 HashTable 效率高一点。另外，HashTable 基本被淘汰，不要在代码中使用它；
- 对Null key 和Null value的支持： HashMap 中，null 可以作为键，这样的键只有一个，可以有一个或多个键所对应的值为 null。。但是在 HashTable 中 put 进的键值只要有一个 null，直接抛出 NullPointerException。
- 初始容量大小和每次扩充容量大小的不同 ： ①创建时如果不指定容量初始值，Hashtable 默认的初始大小为11，之后每次扩充，容量变为原来的2n+1。HashMap 默认的初始化大小为16。之后每次扩充，容量变为原来的2倍。②创建时如果给定了容量初始值，那么 Hashtable 会直接使用你给定的大小，而 HashMap 会将其扩充为2的幂次方大小（HashMap 中的tableSizeFor()方法保证，下面给出了源代码）。也就是说 HashMap 总是使用2的幂作为哈希表的大小,后面会介绍到为什么是2的幂次方。
- 底层数据结构： JDK1.8 以后的 HashMap 在解决哈希冲突时有了较大的变化，当链表长度大于阈值（默认为8）时，将链表转化为红黑树，以减少搜索时间。Hashtable没有这样的机制。

## HashMap和HashSet区别
![](https://ws1.sinaimg.cn/large/005L0VzSgy1g4j9751fmtj30qo076wex.jpg)

## ConcurrentHashMap和Hashtable的区别
- 底层数据结构： JDK1.7的 ConcurrentHashMap 底层采用 分段的数组+链表 实现，JDK1.8 采用的数据结构跟HashMap1.8的结构一样，数组+链表/红黑二叉树。Hashtable 和 JDK1.8 之前的 HashMap 的底层数据结构类似都是采用 数组+链表 的形式，数组是 HashMap 的主体，链表则是主要为了解决哈希冲突而存在的；
- 实现线程安全的方式（重要）：在JDK1.7的时候，ConcurrentHashMap（分段锁） 对整个桶数组进行了分割分段(Segment)，每一把锁只锁容器其中一部分数据，多线程访问容器里不同数据段的数据，就不会存在锁竞争，提高并发访问率。 到了 JDK1.8 的时候已经摒弃了Segment的概念，而是直接用 Node 数组+链表+红黑树的数据结构来实现，并发控制使用 synchronized 和 CAS 来操作。（JDK1.6以后 对 synchronized锁做了很多优化） 整个看起来就像是优化过且线程安全的 HashMap，虽然在JDK1.8中还能看到 Segment 的数据结构，但是已经简化了属性，只是为了兼容旧版本；Hashtable(同一把锁) :使用 synchronized 来保证线程安全，效率非常低下。当一个线程访问同步方法时，其他线程也访问同步方法，可能会进入阻塞或轮询状态，如使用 put 添加元素，另一个线程不能使用 put 添加元素，也不能使用 get，竞争会越来越激烈效率越低。

## ConcurrentHashMap和Hashtable/HashMap的区别
ConcurrentHashMap与HashMap和Hashtable最大的不同在于：put和 get 两次Hash到达指定的HashEntry，第一次hash到达Segment,第二次到达Segment里面的Entry,然后在遍历entry链表。

## 单线程hashmap对比concurenthashmap
1.8版本的concurentHashMap在单线程下和HashMap效率有什么区别
```xml
// 先对ConcurentHashMap测试，再对HashMap测试
loopcount            	 HashMap(ms)          	 ConcurrentHashMap(ms)
1000                 	 60                   	 2                   
1000000              	 123                  	 270                 
10000000             	 1099                 	 2014                
100000000            	 8071                 	 14197    
```
```xml
// 先对HashMap测试，再对ConcurentHashMap测试
loopcount            	 HashMap(ms)          	 ConcurrentHashMap(ms)
1000                 	 1                    	 46                  
1000000              	 96                   	 240                 
10000000             	 828                  	 2132                
100000000            	 8692                 	 20234      
```
通过两次压测表明，刨除编译优化以及代码预热的原因，在单线程场景下HashMap性能一直处于领先地位。


