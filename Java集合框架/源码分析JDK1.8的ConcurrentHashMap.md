我们都知道JDK1.7中的ConcurrentHashMap是Segment分段锁的结构，写的时候每次定位到特定的Segment，只锁住一个Segment，读的时候不加锁，相比HashTable或Collections.synchronizedMap来说，效率高了很多。

JDK1.8之后，ConcurrentHashMap又升级了，网上很多文章说是1.8的实现已经抛弃了Segment分段锁机制，利用CAS+Synchronized来保证并发更新的安全，底层采用数组+链表+红黑树的存储结构，效率又有所提升。但文章并没有继续分析下去，导致本人看完一直觉得一知半解，今天，斗胆照着1.8的源码分析下ConcurrentHashMap，本人水平有限，写的不好的地方，欢迎指正。



##几个参数

```java
//最大容量值
private static final int MAXIMUM_CAPACITY = 1 << 30;
//默认容量值，必须是2的幂次方
 private static final int DEFAULT_CAPACITY = 16;
//1.7用来表示segmemt数组默认大小的，已经弃用
private static final int DEFAULT_CONCURRENCY_LEVEL = 16;
//负载因子
private static final float LOAD_FACTOR = 0.75f;
//链表的长度超过这个值就会转为红黑树
static final int TREEIFY_THRESHOLD = 8;
//当链表长度小于这个值就由红黑树转为普通链表
 static final int UNTREEIFY_THRESHOLD = 6;
//容器可以树化的最小容量。否则如果bin中的节点太多，会调整表的大小
static final int MIN_TREEIFY_CAPACITY = 64;
//每次转移步骤的最小重组次数。 范围是细分为允许多个缩放器线程。 这个值作为下限，以避免resizer遇到过多的内存争用。 该值至少应该是DEFAULT_CAPACITY。
private static final int MIN_TRANSFER_STRIDE = 16;
/*The number of bits used for generation stamp in sizeCtl.
Must be at least 6 for 32bit arrays.*/
private static int RESIZE_STAMP_BITS = 16;
/*resize的时候帮助resize最大的线程数*/
private static final int MAX_RESIZERS = (1 << (32 - RESIZE_STAMP_BITS)) - 1;
//The bit shift for recording size stamp in sizeCtl.
private static final int RESIZE_STAMP_SHIFT = 32 - RESIZE_STAMP_BITS;
	/*
     * 节点哈希字段的编码。
     */
    static final int MOVED     = -1; // hash for forwarding nodes
    static final int TREEBIN   = -2; // hash for roots of trees
    static final int RESERVED  = -3; // hash for transient reservations
    static final int HASH_BITS = 0x7fffffff; // 正常节点哈希的可用位
```

以上这些参数后面会用到，接下来继续分析。



## 重要字段

```java
/**
 * 整个类中最重要的字段，存储元素的数组。使用懒加载，当第一次插入值时才初始化它
 * 它的大小是2的幂次方。可以直接使用迭代器遍历
*/
transient volatile Node<K,V>[] table;
/**
* resize的时候用的，其他时候为null
*/
private transient volatile Node<K,V>[] nextTable;

/*表初始化和调整大小控制字段。 当为负时，表正在初始化或调整大小：-1表示初始化，否则 - （1 +活动大小调整线程数）。 否则，当table为null时，保留要在创建时使用的初始表大小，或者默认为0。 初始化之后，保存下一个元素计数值，在该值上调整表的大小。*/
private transient volatile int sizeCtl;
//调整大小时要分割的下一个表索引（加一）。
private transient volatile int transferIndex;
```





##内部类：Nodes

Key-value entry，该类永远不会作为用户可变的Map.Entry导出，但可以用于批量任务中使用的只读遍历。

```java
static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;
        final K key;
        volatile V val;
        volatile Node<K,V> next;
        Node(int hash, K key, V val, Node<K,V> next) {
            this.hash = hash;
            this.key = key;
            this.val = val;
            this.next = next;
        }
        public final K getKey()       { return key; }
        public final V getValue()     { return val; }
        public final int hashCode()   { return key.hashCode() ^ val.hashCode(); }
        public final String toString(){ return key + "=" + val; }
    	//不允许setValue
        public final V setValue(V value) {
            throw new UnsupportedOperationException();
        }
        public final boolean equals(Object o) {
            Object k, v, u; Map.Entry<?,?> e;
            return ((o instanceof Map.Entry) &&
                    (k = (e = (Map.Entry<?,?>)o).getKey()) != null &&
                    (v = e.getValue()) != null &&
                    (k == key || k.equals(key)) &&
                    (v == (u = val) || v.equals(u)));
        }
        /**
         * Virtualized support for map.get(); overridden in subclasses.
         */
        Node<K,V> find(int h, Object k) {
            Node<K,V> e = this;
            if (k != null) {
                do {
                    K ek;
                    if (e.hash == h &&
                        ((ek = e.key) == k || (ek != null && k.equals(ek))))
                        return e;
                } while ((e = e.next) != null);
            }
            return null;
        }
    }
```



## 构造方法

```java
/**
* 创建默认大小为16的ConcurrentHashMap
*/
public ConcurrentHashMap() {
}

/**
根据指定容量来创建对象
**/
public ConcurrentHashMap(int initialCapacity) {
        //指定容量为负数抛出异常
        if (initialCapacity < 0)
            throw new IllegalArgumentException();
        //因为table的数组必须是2的幂次方，所以容量为大于initialCapacity的最近的2次幂
        int cap = ((initialCapacity >= (MAXIMUM_CAPACITY >>> 1)) ?
                   //若超过了最大容量，则为只能为最大容量
                   MAXIMUM_CAPACITY :
                   /*否则，使用tableSizeFor计算实际容量，
                   这里有点不明白为什么要加上1和0.5倍initialCapacity，而不像HashMap那样直接使用					initialCapacity呢？
                   对照下面的方法，这个值相当于(long)(1.0 + (long)initialCapacity / 0.5)，也就是loadFactor为0.5的情况，让容量更大一点，目的应该是为了减少hashcrash保证查询的效率吧;
                   */
                   tableSizeFor(initialCapacity + (initialCapacity >>> 1) + 1));
       //设置表初始化大小
        this.sizeCtl = cap;
}

	/**根据initialCapacity*1.5+1返回2的幂次方，这个方法HashMap里面也有**/
	private static final int tableSizeFor(int c) {
       	//下面的操作将n变成从第一个1开始后面全是1，之后执行加1操作，就返回了一个>=c的2的幂次方
        int n = c - 1;
        n |= n >>> 1;
        n |= n >>> 2;
        n |= n >>> 4;
        n |= n >>> 8;
        n |= n >>> 16;
        return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
    }

public ConcurrentHashMap(int initialCapacity, float loadFactor) {
        this(initialCapacity, loadFactor, 1);
}
/**
这个方法可以根据initialCapacity,loadFactor,concurrencyLevel来决定table数组的实际大小
loadFactor：用于建立初始表大小的加载因子（表密度），这个值仅仅在初始化的时候使用，跟LOAD_FACTOR没啥关系
concurrencyLevel：估计下会有多少线程使用这个容器，如果concurrencyLevel>initialCapacity,
因为每次更新操作会锁住一个元素，那如果有n个线程同时在使用，而容量只有m(m<n)，如果这些线程同时进行更新操作，必定有一些线程要等待锁释放，所以，让initialCapacity>=concurrencyLevel
这个值在JDK1.7中指定的是segment数组的长度
**/
public ConcurrentHashMap(int initialCapacity,
                             float loadFactor, int concurrencyLevel) {
        if (!(loadFactor > 0.0f) || initialCapacity < 0 || concurrencyLevel <= 0)
            throw new IllegalArgumentException();
        if (initialCapacity < concurrencyLevel)   // Use at least as many bins
            initialCapacity = concurrencyLevel;   // as estimated threads
        long size = (long)(1.0 + (long)initialCapacity / loadFactor);
        int cap = (size >= (long)MAXIMUM_CAPACITY) ?
            MAXIMUM_CAPACITY : tableSizeFor((int)size);
        this.sizeCtl = cap;
}
```



## get方法

如果此映射包含从key{@code k}到value{@code v}的映射，使得{@code key.equals（k）}，则此方法返回{@code v}; 否则返回{@code null}。 （最多可以有一个这样的映射。）同时key不能为null，否则会报错

```java
public V get(Object key) {
    Node<K,V>[] tab; 
    Node<K,V> e, p; 
    int n, eh; K ek;
    int h = spread(key.hashCode());//得到key的hash值
    //如果table不为空，长度大于0，且通过 (n - 1) & h能定位到一个Node，就继续从这个Node开始找，
    //否则返回null
    //(n - 1) & h中n是table数组的长度，定位方式和HashMap的hash^(hash>>>16)&(table.length-1)是几乎是一样的
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (e = tabAt(tab, (n - 1) & h)) != null) {
        if ((eh = e.hash) == h) {
            if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                return e.val;
        }
        // 如果头结点的 hash 小于 0，说明 正在扩容，或者该位置是红黑树
        else if (eh < 0)
          // 参考 ForwardingNode.find(int h, Object k) 和 TreeBin.find(int h, Object k)
            return (p = e.find(h, key)) != null ? p.val : null;
        //遍历Node链表去找值，这个值满足hash值相等且key.equals(ek)
        while ((e = e.next) != null) {
            if (e.hash == h &&
                ((ek = e.key) == key || (ek != null && key.equals(ek))))
                return e.val;
        }
    }
    return null;
}
	/**HASH_BITS=0x7fffffff;
	这个函数的目的是加入高位16位的影响，同时让最高位为0,和hashmap一样
	**/
    static final int spread(int h) {
        return (h ^ (h >>> 16)) & HASH_BITS;
    }
	/**
	最主要的方法，get不加锁就是通过它来实现的
	U是Unsafe类，这个方法定位到了table数组中的某个元素
	**/
	static final <K,V> Node<K,V> tabAt(Node<K,V>[] tab, int i) {
        return (Node<K,V>)U.getObjectVolatile(tab, ((long)i << ASHIFT) + ABASE);
    }
/**
这里Doug Lea采用了Unsafe.getObjectVolatile来获取，那直接table[index]不可以么，为什么要这么复杂？
 在java内存模型中，我们已经知道每个线程都有一个工作内存，里面存储着table的副本，虽然table是volatile修饰的，但volatile不能保证线程每次都拿到table中的最新元素，Unsafe.getObjectVolatile可以直接获取指定内存的数据，保证了每次拿到数据都是最新的。
 **/
```



## put方法

key和value都不能为null，否则会报错

整个put方法使用CAS+synchronized的方法，实现了添加元素时候只对table数组中的某个元素加锁

```javascript
 public V put(K key, V value) {
        return putVal(key, value, false);
    }

 /** Implementation for put and putIfAbsent */
    final V putVal(K key, V value, boolean onlyIfAbsent) {
        if (key == null || value == null) throw new NullPointerException();
        int hash = spread(key.hashCode());
        int binCount = 0;
        //自旋
        for (Node<K,V>[] tab = table;;) {
            Node<K,V> f; int n, i, fh;
            //1.如果table数组为空，初始化，自旋插入操作，这里看出懒加载
            if (tab == null || (n = tab.length) == 0)
                tab = initTable();
            else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
                //2.如果table里面对应的链表为null，就新建一个Node，通过CAS方法设置，如果CAS设置成功就可以break了，否则自旋插入操作
                if (casTabAt(tab, i, null,
                             new Node<K,V>(hash, key, value, null)))
                    break;                   // no lock when adding to empty bin
            }
            //3.如果f的hash值为-1，说明当前f是ForwardingNode节点，意味有其它线程正在扩容，则一起				进行扩容操作。
            else if ((fh = f.hash) == MOVED)
                tab = helpTransfer(tab, f);
            //4.如果以上都没有，说明我们要把节点插入到table数组的某个元素链表中
            else {
                V oldVal = null;
                //5.synchronized锁住链表的头节点，其实就是锁住table数组的一个元素
                synchronized (f) {
                    //6.节点插入之前，再次利用tabAt(tab, i) == f判断，防止被其它线程修改。
                    if (tabAt(tab, i) == f) {
                        //7.如果f的hash值大于0，说明是普通链表
                        if (fh >= 0) {
                            binCount = 1;
                            for (Node<K,V> e = f;; ++binCount) {
                                K ek;
                                //如果遍历链表的时候发现链表中已经有这个key了，根据onlyIfAbsent								判断是否要设置为新值
                                if (e.hash == hash &&
                                    ((ek = e.key) == key ||
                                     (ek != null && key.equals(ek)))) {
                                    oldVal = e.val;
                                    if (!onlyIfAbsent)
                                        e.val = value;
                                    break;
                                }
                                Node<K,V> pred = e;
                                //遍历到结尾，插入节点，和HashMap一样是尾插法
                                if ((e = e.next) == null) {
                                    pred.next = new Node<K,V>(hash, key,
                                                              value, null);
                                    break;
                                }
                            }
                        }
                        //8.如果f是红黑树根节点，则在树结构上遍历元素，更新或增加节点。
                        else if (f instanceof TreeBin) {
                            Node<K,V> p;
                            binCount = 2;
                            if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                           value)) != null) {
                                oldVal = p.val;
                                if (!onlyIfAbsent)
                                    p.val = value;
                            }
                        }
                    }
                }
                if (binCount != 0) {
                 	//9.如果增加节点后binCout>=8，就转变为红黑树
                    if (binCount >= TREEIFY_THRESHOLD)
                        treeifyBin(tab, i);
                    if (oldVal != null)
                        //如果有旧值，返回旧值
                        return oldVal;
                    break;
                }
            }
        }
        //10.判断是否要扩容
        addCount(1L, binCount);
        return null;
    }

	//通过Unsafe的compareAndSwapObject设置tab数组的元素
    static final <K,V> boolean casTabAt(Node<K,V>[] tab, int i,
                                        Node<K,V> c, Node<K,V> v) {
        return U.compareAndSwapObject(tab, ((long)i << ASHIFT) + ABASE, c, v);
    }

	/**
     * Initializes table, using the size recorded in sizeCtl.
     */
    private final Node<K,V>[] initTable() {
        Node<K,V>[] tab; int sc;
        while ((tab = table) == null || tab.length == 0) {
            //1.如果sizeCtl<0，说明其他线程正在初始化，当前线程让出cpu
            if ((sc = sizeCtl) < 0)
                Thread.yield(); // lost initialization race; just spin
            else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
                //2.自旋让sizeCtl从原来的值变成-1，如果自旋成功就进行下面的操作
                try {
                    if ((tab = table) == null || tab.length == 0) {
                        //3.如果使用了构造函数，给sizeCtl设置值，就使用sizeCtl，否则默认是16
                        int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                        @SuppressWarnings("unchecked")
                        //4.初始化
                        Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                        table = tab = nt;
                        sc = n - (n >>> 2);
                    }
                } finally {
                    sizeCtl = sc;
                }
                break;
            }
        }
        return tab;
    }
```

##扩容

```java
private final void addCount(long x, int check) {
    CounterCell[] as; long b, s;
    if ((as = counterCells) != null ||
        !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {
        CounterCell a; long v; int m;
        boolean uncontended = true;
        if (as == null || (m = as.length - 1) < 0 ||
            (a = as[ThreadLocalRandom.getProbe() & m]) == null ||
            !(uncontended =
              U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))) {
            fullAddCount(x, uncontended);
            return;
        }
        if (check <= 1)
            return;
        s = sumCount();
    }
    if (check >= 0) {
        Node<K,V>[] tab, nt; int n, sc;
        while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
               (n = tab.length) < MAXIMUM_CAPACITY) {
            int rs = resizeStamp(n);
            if (sc < 0) {
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                    sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                    transferIndex <= 0)
                    break;
                if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                    transfer(tab, nt);
            }
            else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                         (rs << RESIZE_STAMP_SHIFT) + 2))
                transfer(tab, null);
            s = sumCount();
        }
    }
}
```