---
title: JAVA集合框架
date: 2018-11-20 19:57:22
tags: java集合框架
---



# JAVA集合框架

java三大集合框架 :  Set  List   Map



![20170714230605815](https://3116004636-1256103796.cos.ap-guangzhou.myqcloud.com/20170714230605815.jpg)

如上图 Set List 都属于Collection的子接口(Collection为顶层接口) Map 不属于Collection接口

<!--more-->

## Set接口:  

无序可变的数组,不允许添加重复元素,如果视图把两个相同的元素加入到同一个集合中,add方法返回false.

set判断对象是否相同使用equals方法, 就是说返回true就表示两个对象相同 set不会接受.

需要注意的是:虽然Set中元素没有顺序，但是元素在set中的位置是有由该元素的HashCode决定的，其具体位置其实是固定的。

### HashSet:

不能保证元素的排列顺序,

不是同步的(线程不安全)

集合元素可以是null,但只能放入一个null

当向HashSet集合中存入一个元素时,HashSet会调用该对象的HashCode()方法来得到该对象的HashCode值,然后根据HashCode值来决定该对象在HashSet中的储存位置.

判断两个HashSet元素相等的标准是两个对象通过equals方法比较相等,并且两个对象的HashCode()方法返回值相等

如果要一个对象放入HashSet中,重写该对象对应类的equals方法,也应该重写HashCode()方法.

另外，**HashSet底层是基于HashMap实现的**，只需把HashMap的value值固定，根据HashMap的key值来判重，添加元素的代码如下：

```java
public boolean add(E e) {
    	//PRESENT是一个固定的虚拟值
        return map.put(e, PRESENT)==null;
}
```



### LinkedHashSet:

LinkedHashSet集合同样是根据HashCode()方法来决定元素的存储位置,但是它同时使用链表维护元素的次序.这样使得元素看起来像是以插入顺序保存的,当遍历该集合的时候,LinkedHashSet将会以元素的添加顺序访问集合的元素.

通过查看LinkedHashSet的源码可以发现,其**底层是基于LinkedHashMap来实现的哦。

对于LinkedHashSet而言，它和HashSet主要区别在于LinkedHashSet中存储的元素是在哈希算法的基础上增加了链式表的结构。

LinkedHashSet在迭代访问Set中的全部元素时,性能比HashSet好,但是插入时性能稍逊色于HashSet.

### TreeSet:

TreeSet是SortedSet接口的唯一实现类，TreeSet可以确保集合元素处于排序状态。**底层算法是基于TreeMap来实现的。**

由于TreeMap需要排序,所以需要一个Comparator为键值进行大小比较.当然也是用Comparator定位的.
​	a. Comparator可以在创建TreeMap时指定

​	b. 如果创建时没有确定,那么就会使用key.compareTo()方法,这就要求key必须实现Comparable接口.

​	c. TreeMap是使用Tree数据结构实现的,所以使用compare接口就可以完成定位了.

## List接口:

一个 List 是一个元素有序的、可以重复、可以为 null 的集合（有时候我们也叫它“序列”）

###  ArrayList:

用类似数组的形式进行存储，因此它的随机访问速度极快，缺点是插入和删除的速度慢，就像数组一样，每次插入和删除都需要移动数组中的元素。

注意：

- ArrayList默认长度是10
- 扩容ensureCapaci ty的方案为“原始容量*3/2+1"
- ArrayList是线程不安全的，在多线程的情况下不要使用。



### LinkedList:

在实现中采用链表数据结构。插入和删除速度快，访问速度慢。 

注意：

- LinkedList的内部实现

  LinkedList的内部是基于双向循环链表的结构来实现的。在LinkedList中有一个类似于c语言中结构体的Entry内部类。 在Entry的内部类中包含了前一个元素的地址引用和后一个元素的地址引用类似于c语言中指针

- LinkedList不是线程安全的

​        注意LinkedList和ArrayList一样也不是线程安全的，如果在对线程下面访问可以自己重写LinkedList

​        然后在需要同步的方法上面加上同步关键字synchronized

### Vector:

Vector与ArrayList一样，也是通过数组实现的，不同的是它支持线程的同步，即某一时刻只有一个线程能够写Vector，避免多线程同时写而引起的不一致性，但实现同步需要很高的花费，因此，访问它比访问ArrayList慢。

补充:ListIterator(list特有的迭代器)提供了对List的双向遍历的方法。

Queue接口:暂时不讨论感兴趣的同学可以自行查找





## Map接口:

![20170715001340612](https://3116004636-1256103796.cos.ap-guangzhou.myqcloud.com/20170715001340612.jpg)

Map是一种把键对象和值对象映射的集合，它的每一个元素都包含一对键对象和值对象。Map接口不直接继承于Collection接口（需要注意啦），因为它包装的是一组成对的“键-值”对象的集合，而且在Map接口的集合中也不能有重复的key出现，因为每个键只能与一个成员元素相对应。 从Map集合中检索元素时，只要给出键对象，就会返回对应的值对象。 

另外前边已经说明了，Set接口的底层是基于Map接口实现的。Set中存储的值，其实就是Map中的key，它们都是不允许重复的。

### HashTable:

HashTable类实现一个哈希表，该哈希表将键映射到相应的值。默认容量为11。任何非 null 对象都可以用作键或值。为了成功地在哈希表中存储和获取对象，用作键的对象必须实现 hashCode 方法和 equals 方法。

与HashMap区别主要有三点：

1. Hashtable是基于陈旧的Dictionary实现的，而HashMap是基于Java1.2引进的Map接口实现的；
2. Hashtable是线程安全的，而HashMap是非线程安全的，我们可以使用外部同步的方法解决这个问题。
3. HashMap可以允许你在列表中放一个key值为null的元素，并且可以有任意多value为null，而Hashtable不允许键或者值为null。
4. 默认容量不同

### HashMap:

![散列桶](https://3116004636-1256103796.cos.ap-guangzhou.myqcloud.com/J7vuaqR.jpg!web)

HashMap基于散列桶的实现，散列桶的每一行相当于一个链表。

根据Entry<K,V>的K.hash()去找对应散列表table的位置，对应的方式为K.hash()&table.length-1（即K的hash值对散列表的函数取余），对于具有hash值相同的Key的Entry，插入散列桶中对应位置链表。

每一个Entry都保存了一个hash----键对象的hashcode，如果键没有按照任何特定顺序保存，查找时通过equals()逐一与每一个数组元素进行比较，那么时间复杂度为O(n)，数组长度越大，效率越低。

所以瓶颈在于键的查询速度，如何通过键来快速的定位到存储位置呢？

HashMap将键的hash值与数组下标建立映射，通过键对象的hash函数生成一个值，以此作为数组的下标，这样我们就可以通过键来快速的定位到存储位置了。如果hash函数设计的完美的话，数组的每个位置只有较少的值，那么在O(1)的时间我们就可以找到需要的元素，从而不需要去遍历链表。这样就大大提高了查询速度。

那么HashMap根据hashcode是如何得到数组下标呢？可以拆分为以下几步：

- 第一步： `h = key.hashCode()`
- 第二步： `h ^ (h >>> 16)`
- 第三步： `(length - 1) & hash`

#### 分析

第一步是得到key的hashcode值；

第二步是将键的hashcode的高16位异或低16位(高位运算)，这样即使数组table的length比较小的时候，也能保证高低Bit都参与到Hash的计算中，同时不会有太大的开销；

第三步是hash值和数组长度进行取模运算，这样元素的分布相对来说比较均匀。当length总是2的n次方时， `h & (length-1)` 运算等价于对length取模，这样模运算转化为位移运算速度更快。

源码如下：

```java
    static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }
	final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
			...
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
        	...
    }
```



插入和查询“键值对”的开销是固定的。可以通过构造器设置容量capacity和负载因子loadFactor，以调整容器的性能。 **默认的容量是16，负载因子是0.75，扩容时扩展1倍；如果传入一个容量值capacity，实际容量值为2^n。**

####负载因子的作用：

```java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
  	....添加Entry...
    //threshold=当前容量*负载因子，如果添加Entry后size值>threshold阈值,就扩容
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;
}
```

**可以看出：负载因子表示一个散列桶的空间的使用程度。**

**负载因子越大则散列桶的装填程度越高，也就是能容纳更多的元素，元素多了，链表大了，所以此时索引效率就会降低。反之，负载因子越小则链表中的数据量就越稀疏，此时会对空间造成浪费，但是此时索引效率高。**

### LinkedHashMap: 

类似于HashMap，但是迭代遍历它时，取得“键值对”的顺序是其插入次序，或者是最近最少使用(LRU)的次序。只比HashMap慢一点。而在迭代访问时反而更快，因为它使用链表维护内部次序。 

LinkedHashMap实现与HashMap的不同之处在于，后者维护着一个运行于所有条目的双重链接列表。

### TreeMap:

基于红黑树数据结构的实现。查看“键”或“键值对”时，它们会被排序(次序由Comparable或Comparator决定)。TreeMap的特点在于，你得到的结果是经过排序的。TreeMap是唯一的带有subMap()方法的Map，它可以返回一个子树。 

### ConCurrentHashMap:

HashTable容器在竞争激烈的并发环境下表现出效率低下的原因，是因为所有访问HashTable的线程都必须竞争同一把锁，那假如容器里有多把锁，每一把锁用于锁容器其中一部分数据，那么当多线程访问容器里不同数据段的数据时，线程间就不会存在锁竞争，从而可以有效的提高并发访问效率，这就是ConcurrentHashMap所使用的锁分段技术，首先将数据分成一段一段的存储，然后给每一段数据配一把锁，当一个线程占用锁访问其中一个段数据的时候，其他段的数据也能被其他线程访问。有些方法需要跨段，比如size()和containsValue()，它们可能需要锁定整个表而而不仅仅是某个段，这需要按顺序锁定所有段，操作完毕后，又按顺序释放所有段的锁。这里“按顺序”是很重要的，否则极有可能出现死锁，在ConcurrentHashMap内部，段数组是final的，并且其成员变量实际上也是final的，但是，仅仅是将数组声明为final的并不保证数组成员也是final的，这需要实现上的保证。这可以确保不会出现死锁，因为获得锁的顺序是固定的。



![img](https://images2017.cnblogs.com/blog/400827/201709/400827-20170928212457434-1134706220.png)

ConcurrentHashMap是由Segment数组结构和HashEntry数组结构组成。Segment是一种可重入锁ReentrantLock，在ConcurrentHashMap里扮演锁的角色，HashEntry则用于存储键值对数据。一个ConcurrentHashMap里包含一个Segment数组，Segment的结构和HashMap类似，是一种数组和链表结构， 一个Segment里包含一个HashEntry数组，每个HashEntry是一个链表结构的元素， 每个Segment守护者一个HashEntry数组里的元素,当对HashEntry数组的数据进行修改时，必须首先获得它对应的Segment锁。

参考：[[Java集合---ConcurrentHashMap原理分析](https://www.cnblogs.com/ITtangtang/p/3948786.html)](https://www.cnblogs.com/ITtangtang/p/3948786.html)

## 相关知识:

迭代器(Iterator)
提供一种方法访问一个容器（container）对象中各个元素，而又不需暴露该对象的内部细节。
对于遍历一个容器中所有的元素，Iterator模式是首选的方式
Collection定义了Iterator<E> iterator()方法，子类都各自实现了该方法，我们直接调用即可

Map中虽没有定义，我们可以利用map.entrySet()的iterator()方法