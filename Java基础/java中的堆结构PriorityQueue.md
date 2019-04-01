PriorityQueue：是java提供的一个堆结构，默认是小顶堆,容量默认是11

主要方法：

- poll()：弹出堆顶元素
- peek():返回但不删除此队列的头
- remove(Object o):删除指定元素
- add()或 offer():添加元素

实现方法就是堆调整的方法，删除元素的时候和最后的元素交换，然后进行堆调整



PriorityQueue实现大顶堆，只需要传入一个Comparator：

```java
PriorityQueue<Integer> maxHeap=new PriorityQueue<Integer>(new Comparator<Integer>() {
        @Override
        public int compare(Integer o1, Integer o2) {                
            return o2-o1;
        }
    });
```



