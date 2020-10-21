# 容器

java的容器从最基本接口方面可以分为三大类： Map、Collection、Queue
同一大类下的容器又可以通过继承不同的接口，通过不同的底层数据结构和代码实现在性能、线程安全等方面表现不同的特性

Map：
- HashMap： 基于数组+链表实现，[详见](https://github.com/wangjunjie0817/note/blob/master/java/HashMapSourceCode.md)
- LinkedHashMap
- HashTable
- TreeMap
- ConcurrentHashMap
- ConcurrentSkiplistMap

Collection:
- Vector
- Stack
- ArrayList： 基于数组实现，[详见](https://github.com/wangjunjie0817/note/blob/master/java/ArrayListSourceCode.md)
- LinkedList： 基于链表实现，[详见](https://github.com/wangjunjie0817/note/blob/master/java/LinkedListSourceCode.md)
- CopyOnWriteArrayList

Set
- HashSet
- LinkedHashSet
- TreeSet
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



