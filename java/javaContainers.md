# 容器

java的容器从最基本接口方面可以分为三大类： Map、Collection、Queue
同一大类下的容器又可以通过继承不同的接口，通过不同的底层数据结构和代码实现在性能、线程安全等方面表现不同的特性

Map：
- HashMap： 基于数组+链表实现，[详见](https://github.com/wangjunjie0817/note/blob/master/java/HashMapSourceCode.md)
- LinkedHashMap： 基于HashMap + LinkedList 实现，[详见](https://github.com/wangjunjie0817/note/blob/master/java/LinkedHashMapSourceCode.md)
- HashTable
- TreeMap： 基于红黑树实现key的排序， [详见](https://github.com/wangjunjie0817/note/blob/master/java/TreeMapSourceCode.md)
- ConcurrentHashMap
- ConcurrentSkiplistMap

Collection:
- Vector
- Stack
- ArrayList： 基于数组实现，[详见](https://github.com/wangjunjie0817/note/blob/master/java/ArrayListSourceCode.md)
- LinkedList： 基于链表实现，[详见](https://github.com/wangjunjie0817/note/blob/master/java/LinkedListSourceCode.md)
- CopyOnWriteArrayList

Set
- HashSet:  源码很简单，基于HashMap实现，只有一个成员变量HashMap，set中存储的对象存储为HashMap的key， Value采用一个PRESENT对象填充。提供的API都是通过封装HashMap的方法实现的
- LinkedHashSet： 
- TreeSet: 提供 双排 （排序 + 排重）功能，源码简单，基于TreeMap实现，value固定为Present对象，通过封装TreeMap方法，实现Set的功能
- CopyOnWriteArraySet
- ConcurrentSkiplistSet

Queue
- ArrayQueue
- PriorityQueue
- ConcurrentLinkedQueue

Deque
- ArrayDeque
- LinkedList
- ConcurrentLinkedQueue

BlockingQueue
- ArrayBlockingQueue
- LinkedBlockingQueue
- PriorityBlockingQueue
- DelayQueue
- SynchronousQueue
- LinkedTransferQueue

BlockingDeque
- LinkedBlockingDeque



