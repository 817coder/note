#  LinkedList源码

### 概述
LinkedList ，基于节点实现的双向链表的 List ，每个节点都指向前一个和后一个节点从而形成链表。

### 类图
- java.util.List 接口
- java.io.Serializable 接口
- java.lang.Cloneable 接口
- java.util.Deque 接口，提供了双端队列的功能

### 属性
transient int size = 0; // 链表大小
transient Node<E> first; // 头节点
transient Node<E> last; // 尾节点

### 内部类Node
private static class Node<E> {
    E item;
    Node<E> next;
    Node<E> prev;
    Node(Node<E> prev, E element, Node<E> next) {
        this.item = element;
        this.next = next;
        this.prev = prev;
    }
}

### 添加单个元素
add(E e) 添加到末尾
add(int index, E element) 添加到指定位置

实现了Deque的以下方法：
public void addFirst(E e) {
    linkFirst(e);
}
public boolean offerFirst(E e) {
    addFirst(e); // 调用上面的方法
    return true;
}

public void addLast(E e) {
    linkLast(e);
}
public boolean offerLast(E e) {
    addLast(e); // 调用上面的方法
    return true;
}

实现了Queue的方法
public void push(E e) {
    addFirst(e);
}

public boolean offer(E e) {
    return add(e);
}

### 添加多个元素
addAll(Collection<? extends E> c)
addAll(int index, Collection<? extends E> c)
简单

### 移除单个元素
remove(int index) 简单
remove(Object o) 简单

-------实现 Queue 接口--------
poll() 移除头并返回
pop() 移除头并返回，如果队列为空，抛出异常
remove() 移除首个节点
removeFirst() 移除首个节点，列表为空，报错
-------实现 Deque 接口--------
removeFirstOccurrence(Object o)  移除元素首个出现的节点
removeLastOccurrence(Object o)  移除元素最后出现的节点
removeLast()  移除最后节点，列表为空，报错
pollFirst()   移除头
pollLast()    移除尾

### 移除多个元素
removeAll(Collection<?> c)     移除c中的元素
retainAll(Collection<?> c)     移除不在c中的元素

### 查找单个元素
indexOf(Object o)
contains(Object o) 基于indexOf(Object o)实现

### 获得指定位置的元素
get(int index)
-------实现 Deque 接口--------
peekFirst()   不报错，链表为空返回null
peekLast()   不报错，链表为空返回null
peek()    同peekFirst()
element()   链表为空报错
getFirst()   链表为空报错

### 设置指定位置的元素
set(int index, E element) 简单

### 转换成数组
toArray()
toArray(T[] a)
也比较简单，当a的长度不足，反射生成一个新数组
```java
if (a.length < size){
  a = (T[])java.lang.reflect.Array.newInstance(a.getClass().getComponentType(), size);
}
```

### 求哈希值
hashCode = 31*hashCode + (e==null ? 0 : e.hashCode());

### 判断相等
== instanceof 遍历 一键三连经典判等！！！

### 清空链表
遍历每个节点，前后节点指向为null，help GC

### 序列化链表

### 反序列化链表
























