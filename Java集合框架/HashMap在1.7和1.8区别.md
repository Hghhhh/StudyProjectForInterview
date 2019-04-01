区别：
（1）1.7插入元素的时候使用的是头插法，容易出现逆序且环形链表死循环问题，而1.8使用的尾插法。

（2）1.8使用了数组加链表加红黑树的结构，当链表长度大于8时，转为红黑树，小于6时转为链表，即时在hash都相同极端条件下，时间复杂度也是O（logn）

（3）在resize扩容的时候，1.7是重新进行rehash计算元素在新数组中的位置，而1.8是通过旧的hash值来计算的，如果旧hash值高位1位是0，就是原来的位置，否则就是原来的位置+旧容量。（原因是我们每次扩容都是扩2的幂次倍，这样的话原来的元素要么是在原来的位置，要么是原来的位置+2的幂次）



**HashMap可以通过下面的语句进行同步：Map m = Collections.synchronizeMap(hashMap);**

这个方法放回的是封装好的hashMap，对map的方法都加上了锁。其实和hashtable差不多



hashTable和hashMap的区别？

- hashTable是线程的安全的，它的实现是给所有方法都加上synchronized
- hash算法不一样，hashMap的hash算法是((k.hashCode>>16)^k.hashCode())&t(tab.length-1),而hashtable是(hash & 0x7FFFFFFF) % tab.length
- hashTable继承的是Dictionary类，hashMap继承Map类
- hashTable的初始容量是11，hashTable是16，原因是素数能保证进行hash的时候所有位都生效。