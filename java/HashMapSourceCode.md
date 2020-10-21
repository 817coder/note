# HashMap源码

### 重点概览
1. HashMap 是一种散列表的数据结构，底层采用数组 + 链表 + 红黑树来实现存储。
- Redis Hash 数据结构，采用数组 + 链表实现。
- Redis Zset 数据结构，采用跳表实现。
- 因为红黑树实现起来相对复杂，我们自己在实现 HashMap 可以考虑采用数组 + 链表 + 跳表来实现存储。

2. HashMap 默认容量为 16(1 << 4)，每次超过阀值时，按照两倍大小进行自动扩容，所以容量总是 2^N 次方。并且，底层的 table 数组是延迟初始化，在首次添加 key-value 键值对才进行初始化。

3. HashMap 默认加载因子是 0.75 ，如果我们已知 HashMap 的大小，需要正确设置容量和加载因子。

4. HashMap 每个槽位在满足如下两个条件时，可以进行树化成红黑树，避免槽位是链表数据结构时，链表过长，导致查找性能过慢。
- 条件一，HashMap 的 table 数组大于等于 64 。
- 条件二，槽位链表长度大于等于 8 时。选择 8 作为阀值的原因是，参考 泊松概率函数(Poisson distribution) ，概率不足千万分之一。
- 在槽位的红黑树的节点数量小于等于 6 时，会退化回链表。

5. HashMap 的查找和添加 key-value 键值对的平均时间复杂度为 O(1) 。
- 对于槽位是链表的节点，平均时间复杂度为 O(k) 。其中 k 为链表长度。
- 对于槽位是红黑树的节点，平均时间复杂度为 O(logk) 。其中 k 为红黑树节点数量。

6. 当capcity为2的幂时，hash & (length -1) 相当于取 hash值的低 n位， 结果和 hash mod length一样的）


### 简介
HashMap ，是一种散列表，用于存储 key-value 键值对的数据结构，一般翻译为“哈希表”，提供平均时间复杂度为 O(1) 的、基于 key 级别的 get/put 等操作。

### 接口
实现 java.util.Map 接口，并继承 java.util.AbstractMap 抽像类。
实现 java.io.Serializable 接口。
实现 java.lang.Cloneable 接口。

### HashMap如何实现 O(1) 是平均时间复杂度？
HashMap 其实是在数组的基础上实现的，一个“加强版”的数组。在HashMap中数组默认的初始化大小为16，一般数组的每个index称为槽位或者桶

如何定位：哈希 key
- 假设数组为tab， 长度为n， key为key，那么key在数组的位置的计算方式为：index = tab[(n - 1) & hash(key)

hash冲突解决：链表法
- 通过上述的计算公式当index冲突的时候（hash冲突），有两种方案：开放寻址法、链表法，HashMap采用链表法

冲突优化：如果所有key的hash碰巧都冲突，那复杂度不是变成了O(n)？
- 为了解决运气最差的O(n)问题，红黑树或者跳表可以它们两者的时间复杂度是 O(logN) ，这样 O(N) 就可以缓解成 O(logN) 的时间复杂度。
- HashMap采用的是红黑树

另一项优化：扩容
- 我们是希望 HashMap 尽可能能够达到 O(1) 的时间复杂度，链表法只是我们解决哈希冲突的无奈之举。而在 O(1) 的时间复杂度，基本是“一个萝卜一个坑”，所以在 HashMap 的 key-value 键值对数量达到阀值后，就会进行扩容。扩容三件套：capacity、size、loadFactor
- 扩容条件：capacity / size > loadFactor
- threshold = capacity * loadFactor， 所以当 size  > threshold 触发扩容

所以保证性能尽量趋近于 O(1) ，重点就是几处：
- 哈希 key
- 哈希冲突的解决
- 扩容

### 属性和内部类
transient Node<K,V>[] table; // 底层存储数组
transient Set<Map.Entry<K,V>> entrySet;  // 调用 `#entrySet()` 方法后的缓存
transient int size;  // key-value 的键值对数量
transient int modCount;  // HashMap 的修改次数
int threshold;  // 阀值，当 size 超过 threshold 时，会进行扩容
final float loadFactor; // 扩容因子
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16
static final float DEFAULT_LOAD_FACTOR = 0.75f; // 默认的扩容因子
static final int TREEIFY_THRESHOLD = 8;  // 树化的阈值
static final int UNTREEIFY_THRESHOLD = 6;  // 非树化阈值
static final int MIN_TREEIFY_CAPACITY = 64;  // 最小树化容量

另外还有关键性的内部类，Node和TreeNode
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;
    final K key;
    V value;
    Node<K,V> next;
}

static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V> {
    TreeNode<K,V> parent;  // red-black tree links
    TreeNode<K,V> left;
    TreeNode<K,V> right;
    TreeNode<K,V> prev;    // needed to unlink next upon deletion
    boolean red;
}

### 构造方法
HashMap() // 创建一个初始化容量为 16 的 HashMap 对象， 延迟初始化，在我们开始往 HashMap 中添加 key-value 键值对时，在 #resize() 方法中才真正初始化。
HashMap(int initialCapacity) // 手动指定初始化容量，调用第三个初始化方法
HashMap(int initialCapacity, float 







)  // 指定初始化方法和扩容因子，参数不合法会报错，这里有个关键的函数 tableSizeFor，返回大于 cap 的最小 2 的 N 次方
> static final int tableSizeFor(int cap) {
        int n = cap - 1;
        n |= n >>> 1;
        n |= n >>> 2;
        n |= n >>> 4;
        n |= n >>> 8;
        n |= n >>> 16;
        return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
    }
HashMap(Map<? extends K, ? extends V> m)  // 创建 length = tableSizeFor( m.size() / loadFactor ) 的数组存储数据， 这个看一下源码一眼就明白了

### 哈希函数

(key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);  // h = key.hashCode() 计算哈希值，  ^ (h >>> 16) 高 16 位与自身进行异或计算，保证计算出来的 hash 更加离散

### 添加单个元素
put(K key, V value) 方法，添加单个元素。代码如下：
```java
// HashMap.java

public V put(K key, V value) {
    // hash(key) 计算哈希值
    return putVal(hash(key), key, value, false, true);
}

final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    Node<K,V>[] tab; // tables 数组
    Node<K,V> p; // 对应位置的 Node 节点
    int n; // 数组大小
    int i; // 对应的 table 的位置
    // <1> 如果 table 未初始化，或者容量为 0 ，则进行扩容
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize() /*扩容*/ ).length;
    // <2> 如果对应位置的 Node 节点为空，则直接创建 Node 节点即可。
    if ((p = tab[i = (n - 1) & hash] /*获得对应位置的 Node 节点*/) == null)
        tab[i] = newNode(hash, key, value, null);
    // <3> 如果对应位置的 Node 节点非空，则可能存在哈希冲突
    else {
        Node<K,V> e; // key 在 HashMap 对应的老节点
        K k;
        // <3.1> 如果找到的 p 节点，就是要找的，则则直接使用即可
        if (p.hash == hash && // 判断 hash 值相等
            ((k = p.key) == key || (key != null && key.equals(k)))) // 判断 key 真正相等
            e = p;
        // <3.2> 如果找到的 p 节点，是红黑树 Node 节点，则直接添加到树中
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        // <3.3> 如果找到的 p 是 Node 节点，则说明是链表，需要遍历查找
        else {
            // 顺序遍历链表
            for (int binCount = 0; ; ++binCount) {
                // `(e = p.next)`：e 指向下一个节点，因为上面我们已经判断了最开始的 p 节点。
                // 如果已经遍历到链表的尾巴，则说明 key 在 HashMap 中不存在，则需要创建
                if ((e = p.next) == null) {
                    // 创建新的 Node 节点
                    p.next = newNode(hash, key, value, null);
                    // 链表的长度如果数量达到 TREEIFY_THRESHOLD（8）时，则进行树化。
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break; // 结束
                }
                // 如果遍历的 e 节点，就是要找的，则则直接使用即可
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break; // 结束
                // p 指向下一个节点
                p = e;
            }
        }
        // <4.1> 如果找到了对应的节点
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            // 修改节点的 value ，如果允许修改
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            // 节点被访问的回调
            afterNodeAccess(e);
            // 返回老的值
            return oldValue;
        }
    }
    // <4.2>
    // 增加修改次数
    ++modCount;
    // 如果超过阀值，则进行扩容
    if (++size > threshold)
        resize();
    // 添加节点后的回调
    afterNodeInsertion(evict);
    // 返回 null
    return null;
}
```
基本的思路是：
- 首先判断数组是否为null， 如果为null就初始化
- 然后获取槽位对应的Node对象，如果对应位置的 Node 节点为空，则直接创建 Node 节点即可。如果对应位置的 Node 节点非空，则可能存在哈希冲突
- 当存在Hash冲突时，如果节点是Node，则遍历查找是否有相同key的Node；如果节点是TreeNode，则putTreeVal
- 如果查找到了key相同的Node，如果满足 （!onlyIfAbsent || oldValue == null） 则更新节点的值

这里注意两个点：
1. 在put新数据的时候，是先放数据然后判断扩容
2. 注意afterNodeAccess(e)和afterNodeInsertion(evict);两个回调方法，LinkedHashMap就是基于回调方法实现的key插入顺序或者访问顺序有序的

### 扩容
resize()，两个作用：当table == null的时候，初始化table；table != null的时候，两倍扩容HashMap
```java
// HashMap.java

final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;
    // <1> 开始：
    // <1.1> oldCap 大于 0 ，说明 table 非空
    if (oldCap > 0) {
        // <1.1.1> 超过最大容量，则直接设置 threshold 阀值为 Integer.MAX_VALUE ，不再允许扩容
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        // <1.1.2> newCap = oldCap << 1 ，目的是两倍扩容
        // 如果 oldCap >= DEFAULT_INITIAL_CAPACITY 满足，说明当前容量大于默认值（16），则 2 倍阀值。
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                 oldCap >= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr << 1; // double threshold
    }
    // <1.2.1>【非默认构造方法】oldThr 大于 0 ，则使用 oldThr 作为新的容量
    else if (oldThr > 0) // initial capacity was placed in threshold
        newCap = oldThr;
    // <1.2.2>【默认构造方法】oldThr 等于 0 ，则使用 DEFAULT_INITIAL_CAPACITY 作为新的容量，使用 DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY 作为新的容量
    else {               // zero initial threshold signifies using defaults
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    // 1.3 如果上述的逻辑，未计算新的阀值，则使用 newCap * loadFactor 作为新的阀值
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                  (int)ft : Integer.MAX_VALUE);
    }
    // <2> 开始：
    // 将 newThr 赋值给 threshold 属性
    threshold = newThr;
    // 创建新的 Node 数组，赋值给 table 属性
    @SuppressWarnings({"rawtypes","unchecked"})
    Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
    // 如果老的 table 数组非空，则需要进行一波搬运
    if (oldTab != null) {
        for (int j = 0; j < oldCap; ++j) {
            // 获得老的 table 数组第 j 位置的 Node 节点 e
            Node<K,V> e;
            if ((e = oldTab[j]) != null) {
                // 置空老的 table 数组第 j 位置
                oldTab[j] = null;
                // <2.1> 如果 e 节点只有一个元素，直接赋值给新的 table 即可
                if (e.next == null)
                    newTab[e.hash & (newCap - 1)] = e;
                // <2.2> 如果 e 节点是红黑树节点，则通过红黑树分裂处理
                else if (e instanceof TreeNode)
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                // <2.3> 如果 e 节点是链表
                else { // preserve order
                    // HashMap 是成倍扩容，这样原来位置的链表的节点们，会被分散到新的 table 的两个位置中去
                    // 通过 e.hash & oldCap 计算，根据结果分到高位、和低位的位置中。
                    // 1. 如果结果为 0 时，则放置到低位
                    // 2. 如果结果非 1 时，则放置到高位
                    Node<K,V> loHead = null, loTail = null;
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    // 这里 do while 的原因是，e 已经非空，所以减少一次判断。细节~
                    do {
                        // next 指向下一个节点
                        next = e.next;
                        // 满足低位
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        // 满足高位
                        else {
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    // 设置低位到新的 newTab 的 j 位置上
                    if (loTail != null) {
                        loTail.next = null;
                        newTab[j] = loHead;
                    }
                    // 设置高位到新的 newTab 的 j + oldCap 位置上
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
这个方法重点就在于理解2.3数据搬运的代码，在java8中数据迁移到新table之后，链表的顺序不会发生改变，而在java7中，由于顺序会倒序，则可能会发生死锁！！！
另外的一个难点在于高位和地位的理解，代码就是(e.hash & oldCap) 结果是否 == 0

### 树化
树化的方法是 treeifyBin(Node<K,V>[] tab, int hash) 方法
树化的条件有两个  tab.length >= MIN_TREEIFY_CAPACITY 并且 binCount >= TREEIFY_THRESHOLD
树化的过程是，首先将每个节点转化为TreeNode，然后调用 TreeNode.treeify 方法，具体的可以看代码

### 添加多个元素
putAll(Map<? extends K, ? extends V> m) 方法，添加多个元素到 HashMap 中， 和 HashMap(Map<? extends K, ? extends V> m) 构造方法一样，都调用 putMapEntries(Map<? extends K, ? extends V> m, boolean evict) 方法。

### 移除单个元素
```java
public V remove(Object key) {
    Node<K,V> e;
    // hash(key) 求哈希值
    return (e = removeNode(hash(key), key, null, false, true)) == null ?
        null : e.value;
}

final Node<K,V> removeNode(int hash, Object key, Object value,
                           boolean matchValue, boolean movable) {
    Node<K,V>[] tab; // table 数组
    Node<K,V> p; // hash 对应 table 位置的 p 节点
    int n, index;
    // <1> 查找 hash 对应 table 位置的 p 节点
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (p = tab[index = (n - 1) & hash]) != null) {
        Node<K,V> node = null, // 如果找到 key 对应的节点，则赋值给 node
                e;
        K k; V v;
        // <1.1> 如果找到的 p 节点，就是要找的，则则直接使用即可
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            node = p;
        else if ((e = p.next) != null) {
            // <1.2> 如果找到的 p 节点，是红黑树 Node 节点，则直接在红黑树中查找
            if (p instanceof TreeNode)
                node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
            // <1.3> 如果找到的 p 是 Node 节点，则说明是链表，需要遍历查找
            else {
                do {
                    // 如果遍历的 e 节点，就是要找的，则则直接使用即可
                    if (e.hash == hash &&
                        ((k = e.key) == key ||
                         (key != null && key.equals(k)))) {
                        node = e;
                        break; // 结束
                    }
                    p = e; // 注意，这里 p 会保存找到节点的前一个节点
                } while ((e = e.next) != null);
            }
        }
        // <2> 如果找到 node 节点，则进行移除
        // 如果有要求匹配 value 的条件，这里会进行一次判断先移除
        if (node != null && (!matchValue || (v = node.value) == value ||
                             (value != null && value.equals(v)))) {
            // <2.1> 如果找到的 node 节点，是红黑树 Node 节点，则直接在红黑树中删除
            if (node instanceof TreeNode)
                ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
            // <2.2.1> 如果查找到的是链表的头节点，则直接将 table 对应位置指向 node 的下一个节点，实现删除
            else if (node == p)
                tab[index] = node.next;
            // <2.2.2> 如果查找到的是链表的中间节点，则将 p 指向 node 的下一个节点，实现删除
            else
                p.next = node.next;
            // 增加修改次数
            ++modCount;
            // 减少 HashMap 数量
            --size;
            // 移除 Node 后的回调
            afterNodeRemoval(node);
            // 返回 node
            return node;
        }
    }
    // 查找不到，则返回 null
    return null;
}
```
这段代码也比较精彩，具体见代码
这里注意一下 afterNodeRemoval(node) 的删除节点的回调，LinkedHashMap会重写这个方法，将key从LinkedList中删除

### 查找单个元素
这个方法就比较简单了，上面删除的代码中已经包含了全部精髓

containsKey(Object key)  // 基于这个方法实现
containsValue(Object value)  // wow，这个方法写的好low，虽然我也没好办法

### 清空
简单粗暴
```java
for (int i = 0; i < tab.length; ++i)
  tab[i] = null;
```

### 序列化反序列化
浅拷贝，具体看代码，比较简单

### clone
浅拷贝，也比较简单

### java1.8和java1.7 HashMap的区别












