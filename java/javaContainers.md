# 容器

java的容器从最基本接口方面可以分为三大类： Map、Collection、Queue
同一大类下的容器又可以通过继承不同的接口，通过不同的底层数据结构和代码实现在性能、线程安全等方面表现不同的特性

### Map的类关系为：
![image](https://github.com/wangjunjie0817/note/blob/master/images/map.png)

### Collection的类关系如下：
![image](https://github.com/wangjunjie0817/note/blob/master/images/collection.png)

Map：
- HashMap： 基于数组+链表实现，[详见](https://github.com/wangjunjie0817/note/blob/master/java/HashMapSourceCode.md)
- LinkedHashMap： 基于HashMap + LinkedList 实现，[详见](https://github.com/wangjunjie0817/note/blob/master/java/LinkedHashMapSourceCode.md)
- HashTable： 早期的哈希表实现版本，基于Entry数组+链表实现，方法都是同步的，key和value都不能是null；继承自Dictionary。
- TreeMap： 基于红黑树实现key的排序， [详见](https://github.com/wangjunjie0817/note/blob/master/java/TreeMapSourceCode.md)
- ConcurrentHashMap
- ConcurrentSkiplistMap

Collection:
- Vector： 底层基于char[]， 同步访问的线性存储结构
- Stack： stack底层基于Vector 整个类也比较简单，就三个主要方法 push 、pop 、peek
- ArrayList： 基于数组实现，[详见](https://github.com/wangjunjie0817/note/blob/master/java/ArrayListSourceCode.md)
- LinkedList： 基于链表实现，[详见](https://github.com/wangjunjie0817/note/blob/master/java/LinkedListSourceCode.md)
- CopyOnWriteArrayList

Set
- HashSet:  源码很简单，基于HashMap实现，只有一个成员变量HashMap，set中存储的对象存储为HashMap的key， Value采用一个PRESENT对象填充。提供的API都是通过封装HashMap的方法实现的
- LinkedHashSet： 底层基于LinkedHashMap实现，继承自HashSet，所有的方法和HashSet一样，只不过HashSet是基于HashMap实现的
- TreeSet: 提供 双排 （排序 + 排重）功能，源码简单，基于TreeMap实现，value固定为Present对象，通过封装TreeMap方法，实现Set的功能
- CopyOnWriteArraySet
- ConcurrentSkiplistSet

Queue

详见：https://github.com/wangjunjie0817/note/blob/master/java/queue.md

### 最后补充知乎上关于容器初始化容积的问题

1. HashMap 和 HashSet 的默认大小是16。
2. Hashtable 的默认大小是11。
3. ArrayList 和 Vector 的默认大小是10。
4. ArrayDeque 的默认大小是8。
5. PriorityQueue 的默认大小是11。

问题：
1. HashMap 的默认大小为16，有什么特别的理由吗？
已解决。
HashMap 的容量大小需要保证为 2^n。
HashMap 定位桶时是对桶容量( HashMap 容量)取余，（无论因果），该取余函数是用 h & (length - 1) 实现。@RednaxelaFX
2. HashTable 的默认大小为11，有什么特别的理由吗？
已解决。
HashTable 的容量增加逻辑是乘2+1，保证奇数。
在应用数据随机分布时，容量越大越好。Hash时取模一定要模质数吗？ - 编程
在应用数据分布在等差数据集合(如偶数)上时，如果公差与桶容量有公约数n，则至少有(n-1)/n数量的桶是利用不到的。
实际上 HashMap 也会有此问题，并且不能指定桶容量。所以 HashMap 会在取模哈希前先做一次哈希，@RednaxelaFX。
h ^= (h >>> 20) ^ (h >>> 12);
h ^ (h >>> 7) ^ (h >>> 4);
3. ArrayList 默认大小为10，有什么特别的理由吗？
ArrayList 的容量增长逻辑是乘 1.5 + 1，逻辑比较随意，看不出有什么特别含义。
4. ArrayDeque 默认大小为8，注释明确说明容量需要取 2^n。但是 ArrayDeque 内部存储结构是数组(没有哈希操作)，并没有与问题1类似的需求。
已解决
ArrayDeque 的容量大小需要保证为 2^n。
ArrayDeque 使用 (tail - head) & (elements.length - 1) 计算 size()
5. PriorityQueue 默认大小为11，内部存储结构是数组(最小堆)，却没有使用 2^n 而是11。
PriorityQueue 的容量增长逻辑是 乘 2 或 1.5，逻辑比较随意，看不出有什么特别含义。
