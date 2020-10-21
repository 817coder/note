# LinkedHashMap源码解析

### 重点概览

LinkedHashMap 是 HashMap 的子类，增加了顺序访问的特性。
- 【默认】当 accessOrder = false 时，按照 key-value 的插入顺序进行访问。
- 当 accessOrder = true 时，按照 key-value 的读取顺序进行访问。
LinkedHashMap 的顺序特性，通过内部的双向链表实现，所以我们把它看成是 LinkedList + LinkedHashMap 的组合。
LinkedHashMap 通过重写 HashMap 提供的回调方法，从而实现其对顺序的特性的处理。同时，因为 LinkedHashMap 的顺序特性，需要重写 #keysToArray(T[] a) 等遍历相关的方法。
LinkedHashMap 可以方便实现 LRU 算法的缓存

### 概述
HashMap是访问无序的，同时key也是无序的
所以java提供了两个类
- LinkedHashMap   访问或者插入顺序有序
- TreeMap      key有序
实际上，LinkedHashMap 可以理解成是 LinkedList + HashMap 的组合。

### 类图
实现 Map 接口。
继承 HashMap 类

### 属性

```java
transient LinkedHashMap.Entry<K,V> head;  // 头节点。越老的节点，放在越前面。所以头节点，指向链表的开头
transient LinkedHashMap.Entry<K,V> tail;  // 越新的节点，放在越后面。所以尾节点，指向链表的结尾
/**
 * 是否按照访问的顺序。
 * true ：按照 key-value 的访问顺序进行访问。
 * false ：按照 key-value 的插入顺序进行访问。
 */
final boolean accessOrder;

// 内部定义了Entry类，继承自HashMap的Node
static class Entry<K,V> extends HashMap.Node<K,V> {
    Entry<K,V> before, // 前一个节点
            after; // 后一个节点
    Entry(int hash, K key, V value, Node<K,V> next) {
        super(hash, key, value, next);
    }
}
```

### 构造方法
LinkedHashMap 一共有 5 个构造方法，其中四个和 HashMap 相同，只是多初始化 accessOrder = false 。所以，默认使用插入顺序进行访问。

另外一个 #LinkedHashMap(int initialCapacity, float loadFactor, boolean accessOrder) 构造方法，允许自定义 accessOrder 属性。代码如下：

```java
public LinkedHashMap(int initialCapacity, float loadFactor, boolean accessOrder) {
    super(initialCapacity, loadFactor);
    this.accessOrder = accessOrder;
}
```

### 创建节点
当调用put(K key, V value)方法添加元素的时候，回调用newNode方法创建新的节点，LinkedHashMap重写了这个方法
```java
Node<K,V> newNode(int hash, K key, V value, Node<K,V> e) {
    // <1> 创建 Entry 节点
    LinkedHashMap.Entry<K,V> p =
        new LinkedHashMap.Entry<>(hash, key, value, e);
    // <2> 添加到结尾
    linkNodeLast(p);
    // 返回
    return p;
}
```
当生成新节点的时候，调用linkNodeLast将节点加到链表的末尾，这样，符合越新的节点，放到链表的越后面。

### 节点操作回调
在 HashMap 的读取、添加、删除时，分别提供了 #afterNodeAccess(Node<K,V> e)、#afterNodeInsertion(boolean evict)、#afterNodeRemoval(Node<K,V> e) 回调方法。这样，LinkedHashMap 可以通过它们实现自定义拓展逻辑。

- afterNodeAccess(Node<K,V> e)    将节点加到列表的末尾或者移动到列表的末尾。如果HashMap的方法调用了这个方法，就不用重写，否则LinkedHashMap重写了afterNodeAccess方法
- afterNodeInsertion(boolean evict)     删除节点，当插入节点的时候会回调这个方法。删除节点的条件是 evict 参数为 true 和 removeEldestEntry(Map.Entry<K,V> eldest)方法的结果为true。这个方法在LinkedHashMap中为空实现，所以如果需要基于LinkedHashMap实现LRU缓存需要重写这个方法。
```java
class LRUCache<K, V> extends LinkedHashMap<K, V> {

    private final int CACHE_SIZE;

    /**
     * 传递进来最多能缓存多少数据
     *
     * @param cacheSize 缓存大小
     */
    public LRUCache(int cacheSize) {
        // true 表示让 LinkedHashMap 按照访问顺序来进行排序，最近访问的放在头部，最老访问的放在尾部。
        super((int) Math.ceil(cacheSize / 0.75) + 1, 0.75f, true);
        CACHE_SIZE = cacheSize;
    }

    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        // 当 map 中的数据量大于指定的缓存个数的时候，就自动删除最老的数据。
        return size() > CACHE_SIZE;
    }

}
```

- afterNodeRemoval
在节点被移除时，LinkedHashMap 需要将节点也从链表中移除，所以重写 #afterNodeRemoval(Node<K,V> e) 方法，实现该逻辑。

### 转换成数组

### 转换成 Set/Collection

### 清空
```java
public void clear() {
    // 清空
    super.clear();
    // 标记 head 和 tail 为 null
    head = tail = null;
}
```






























