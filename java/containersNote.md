# 容器
容器可以分为三个大类： Collection、Map、Queue
### Map的类关系为：
![image](https://github.com/wangjunjie0817/note/blob/master/images/map.png)

#### HashMap
初始化
扩容
hash
#### HashTable
早期的哈希表实现版本，方法都是同步的，key和value都不能是null
继承Dictionary
#### LinkedHashMap
继承HashMap，通过LinkedList维护key插入或者访问的顺序，可以用作LRU缓存
#### TreeMap
按照key的顺序的 Map 实现类


### Collection的类关系如下：
![image](https://github.com/wangjunjie0817/note/blob/master/images/collection.png)

#### Vector
底层基于char\[]， 同步访问的线性存储结构
#### ArrayList
底层基于char\[]， 非同步随机访问的线性存储结构
#### LinkedList
底层基于char\[]， 线性链式存储结构
#### HashSet
基于HashMap，key为 set 的元素，value是 PRESENT 对象
#### TreeSet
基于TreeMap实现

### Queue
#### LinkedList
底层基于链表实现的双端队列
#### ArrayDeque
底层基于数组存储实现的双端队列
#### PriorityQueue
优先队列底层基于数组实现的大顶堆、小顶堆

### Stack
最后补充一个比较奇葩的数据结构   Stack
stack底层基于Vector   整个类也比较简单，就三个主要方法   push 、pop 、peek

### 最后补充知乎上关于容器初始化容积的问题

JDK1.7的实现中：
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