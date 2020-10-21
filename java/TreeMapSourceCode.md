# TreeMap源码解析

### 简介

按照 key 的顺序的 Map 实现类，底层基于红黑树实现。

### 类图
实现 java.util.Map 接口，并继承 java.util.AbstractMap 抽像类。

直接实现 java.util.NavigableMap 接口，间接实现 java.util.NavigableMap 接口。关于这两接口的定义的操作，已经添加注释，胖友可以直接点击查看。因为 SortedMap 有很多雷同的寻找最接近 key 的操作，这里简单总结下：

- lower ：小于 ；floor ：小于等于

- higher ：大于；ceiling ：大于等于

实现 java.io.Serializable 接口。

实现 java.lang.Cloneable 接口。

### 属性

private final Comparator<? super K> comparator;  // 排序器

private transient Entry<K,V> root;  // 红黑树的根节点

private transient int size = 0;    // key和value键值对数

private transient int modCount = 0;     // 修改次数

内部类Entry，节点的数据类型

```java
static final class Entry<K,V> implements Map.Entry<K,V> {
    K key;
    V value;
    Entry<K,V> left;
    Entry<K,V> right;
    Entry<K,V> parent;
    boolean color = BLACK;
}
```

### 构造方法

TreeMap()

TreeMap(Comparator<? super K> comparator)   // 指定排序器

TreeMap(SortedMap<K, ? extends V> m)      // 内部排序使用入参的排序器，这里需要看一下灵魂代码

TreeMap(Map<? extends K, ? extends V> m)

### TreeMap(SortedMap<K, ? extends V> m) 源码解析

```java
public TreeMap(SortedMap<K, ? extends V> m) {
    // <1> 设置 comparator 属性
    comparator = m.comparator();
    try {
        // <2> 使用 m 构造红黑树
        buildFromSorted(m.size(), m.entrySet().iterator(), null, null);
    } catch (java.io.IOException | ClassNotFoundException cannotHappen) {
    }
}
```
代码如下
```java
private void buildFromSorted(int size, Iterator<?> it,
                             java.io.ObjectInputStream str,
                             V defaultVal)
    throws  java.io.IOException, ClassNotFoundException {
    // <1> 设置 key-value 键值对的数量
    this.size = size;
    // <2> computeRedLevel(size) 方法，计算红黑树的高度
    // <3> 使用 m 构造红黑树，返回根节点
    root = buildFromSorted(0, 0, size - 1, computeRedLevel(size),
                           it, str, defaultVal);
}
```
这里需要注意一下computeRedLevel方法，本质上就是求 对数+1
```java
private static int computeRedLevel(int size) {
    return 31 - Integer.numberOfLeadingZeros(size + 1);
}
```
可以看到，调用了buildFromSorted方法，其实这个方法就是核心方法，按照注释，一步一步走！！！这个方法就是生成红黑树，并且返回root节点
```java
private final Entry<K,V> buildFromSorted(int level, int lo, int hi,
                                         int redLevel,
                                         Iterator<?> it,
                                         java.io.ObjectInputStream str,
                                         V defaultVal)
    throws  java.io.IOException, ClassNotFoundException {
    /*
     * Strategy: The root is the middlemost element. To get to it, we
     * have to first recursively construct the entire left subtree,
     * so as to grab all of its elements. We can then proceed with right
     * subtree.
     *
     * The lo and hi arguments are the minimum and maximum
     * indices to pull out of the iterator or stream for current subtree.
     * They are not actually indexed, we just proceed sequentially,
     * ensuring that items are extracted in corresponding order.
     */
    // <1.1> 递归结束
    if (hi < lo) return null;

    // <1.2> 计算中间值
    int mid = (lo + hi) >>> 1;

    // <2.1> 创建左子树
    Entry<K,V> left  = null;
    if (lo < mid)
        // <2.2> 递归左子树
        left = buildFromSorted(level + 1, lo, mid - 1, redLevel,
                               it, str, defaultVal);

    // extract key and/or value from iterator or stream
    // <3.1> 获得 key-value 键值对
    K key;
    V value;
    if (it != null) { // 使用 it 迭代器，获得下一个值
        if (defaultVal == null) {
            Map.Entry<?,?> entry = (Map.Entry<?,?>)it.next(); // 从 it 获得下一个 Entry 节点
            key = (K) entry.getKey(); // 读取 key
            value = (V) entry.getValue(); // 读取 value
        } else {
            key = (K)it.next();  // 读取 key
            value = defaultVal; // 设置 default 为 value
        }
    } else { // use stream 处理 str 流的情况
        key = (K) str.readObject(); //  从 str 读取 key 值
        value = (defaultVal != null ? defaultVal : (V) str.readObject()); // 从 str 读取 value 值
    }

    
    // <3.2> 创建中间节点
    Entry<K,V> middle =  new Entry<>(key, value, null);

    // color nodes in non-full bottommost level red
    // <3.3> 如果到树的最大高度，则设置为红节点
    if (level == redLevel)
        middle.color = RED;

    // <3.4> 如果左子树非空，则进行设置
    if (left != null) {
        middle.left = left; // 当前节点，设置左子树
        left.parent = middle; // 左子树，设置父节点为当前节点
    }

    // <4.1> 创建右子树
    if (mid < hi) {
        // <4.2> 递归右子树
        Entry<K,V> right = buildFromSorted(level + 1, mid + 1, hi, redLevel,
                                           it, str, defaultVal);
        // <4.3> 当前节点，设置右子树
        middle.right = right;
        // <4.3> 右子树，设置父节点为当前节点
        right.parent = middle;
    }

    // 返回当前节点
    return middle;
}
```

### TreeMap(Map<? extends K, ? extends V> m)源码解析

底层就是调用putAll方法，如果是SortedMap类型，则调用buildFromSorted方法构建红黑树，否则遍历每个Entry，单个添加

```java
public void putAll(Map<? extends K, ? extends V> map) {
    // <1> 路径一，满足如下条件，调用 buildFromSorted 方法来优化处理
    int mapSize = map.size();
    if (size == 0 // 如果 TreeMap 的大小为 0
            && mapSize != 0 // map 的大小非 0
            && map instanceof SortedMap) { // 如果是 map 是 SortedMap 类型
        if (Objects.equals(comparator, ((SortedMap<?,?>)map).comparator())) { // 排序规则相同
            // 增加修改次数
            ++modCount;
            // 基于 SortedMap 顺序迭代插入即可
            try {
                buildFromSorted(mapSize, map.entrySet().iterator(),
                                null, null);
            } catch (java.io.IOException | ClassNotFoundException cannotHappen) {
            }
            return;
        }
    }
    // <2> 路径二，直接遍历 map 来添加
    super.putAll(map);
}
```

### 添加单个元素

```java
public V put(K key, V value) {
    // 记录当前根节点
    Entry<K,V> t = root;
    // <1> 如果无根节点，则直接使用 key-value 键值对，创建根节点
    if (t == null) {
        // <1.1> 校验 key 类型。
        compare(key, key); // type (and possibly null) check

        // <1.2> 创建 Entry 节点
        root = new Entry<>(key, value, null);
        // <1.3> 设置 key-value 键值对的数量
        size = 1;
        // <1.4> 增加修改次数
        modCount++;
        return null;
    }
    // <2> 遍历红黑树
    int cmp; // key 比父节点小还是大
    Entry<K,V> parent; // 父节点
    // split comparator and comparable paths
    Comparator<? super K> cpr = comparator;
    if (cpr != null) { // 如果有自定义 comparator ，则使用它来比较
        do {
            // <2.1> 记录新的父节点
            parent = t;
            // <2.2> 比较 key
            cmp = cpr.compare(key, t.key);
            // <2.3> 比 key 小，说明要遍历左子树
            if (cmp < 0)
                t = t.left;
            // <2.4> 比 key 大，说明要遍历右子树
            else if (cmp > 0)
                t = t.right;
            // <2.5> 说明，相等，说明要找到的 t 就是 key 对应的节点，直接设置 value 即可。
            else
                return t.setValue(value);
        } while (t != null);  // <2.6>
    } else { // 如果没有自定义 comparator ，则使用 key 自身比较器来比较
        if (key == null) // 如果 key 为空，则抛出异常
            throw new NullPointerException();
        @SuppressWarnings("unchecked")
        Comparable<? super K> k = (Comparable<? super K>) key;
        do {
            // <2.1> 记录新的父节点
            parent = t;
            // <2.2> 比较 key
            cmp = k.compareTo(t.key);
            // <2.3> 比 key 小，说明要遍历左子树
            if (cmp < 0)
                t = t.left;
            // <2.4> 比 key 大，说明要遍历右子树
            else if (cmp > 0)
                t = t.right;
            // <2.5> 说明，相等，说明要找到的 t 就是 key 对应的节点，直接设置 value 即可。
            else
                return t.setValue(value);
        } while (t != null); // <2.6>
    }
    // <3> 创建 key-value 的 Entry 节点
    Entry<K,V> e = new Entry<>(key, value, parent);
    // 设置左右子树
    if (cmp < 0) // <3.1>
        parent.left = e;
    else // <3.2>
        parent.right = e;
    // <3.3> 插入后，进行自平衡
    fixAfterInsertion(e);
    // <3.4> 设置 key-value 键值对的数量
    size++;
    // <3.5> 增加修改次数
    modCount++;
    return null;
}
```

剩余的全部源码，见芋道源码 http://svip.iocoder.cn/JDK/Collection-TreeMap/
























