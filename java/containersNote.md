# 容器
容器可以分为三个大类： Collection、Map、Queue


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


