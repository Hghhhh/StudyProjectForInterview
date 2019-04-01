1. final int hash(Object k) {`
2. `int h = hashSeed;`
3. `if (0 != h && k instanceof String) {`
4. `return sun.misc.Hashing.stringHash32((String) k);`
5. `}`
6.  
7. `h ^= k.hashCode();`
8. `h ^= (h >>> 20) ^ (h >>> 12);`
9. `return h ^ (h >>> 7) ^ (h >>> 4);`
10. `}`
11.  
12. `static int indexFor(int h, int length) {`
13. `return h & (length-1);`
14. `}`

**HashMap的数据结构是一个Node数组（这个Node是一个链表），请问一个元素怎么定位到数组的下标？**

答：拿到这个元素key的hashcode，将这个hashcode右移16位后与原来的hashcode异或得到的数作为hash码（hash = (hashcode>>16)^hashcode），这样是为了把高位的特征和低位的特征组合起来，减少hash冲突。再根据这个hash码对数组长度取余，取余使用的是位运算，hash按位与Node数组的长度减一（hash&（node-1）），得到下标