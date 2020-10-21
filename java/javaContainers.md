# 容器

java的容器从最基本接口方面可以分为三大类： Map、Collection、Queue
同一大类下的容器又可以通过继承不同的接口，通过不同的底层数据结构和代码实现在性能、线程安全等方面表现不同的特性

Map：
- HashMap
- LinkedHashMap
- HashTable
- TreeMap
- ConcurrentHashMap
- ConcurrentSkiplistMap

Collection:
- Vector
- Stack
- ArrayList
- LinkedList
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



