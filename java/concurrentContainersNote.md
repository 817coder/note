# 并发List（Vector & CopyOnWriteArrayList）
ArrayList 不是线程安全的。因此，应该尽量避免在多线程环境中使用ArrayList。如果因为某些原因必须使用的，则需要使用Collections.synchronizedList(List list)进行包装。
示例代码：
```java
List list = Collections.synchronizedList(new ArrayList());
    ...
synchronized (list) {
    Iterator i = list.iterator(); // 必须在同步块中
    while (i.hasNext())
        foo(i.next());
}
```

**Vector & CopyOnWriteArrayList 是两个线程安全的List实现**
- CopyOnWriteArrayList 的内部实现与Vector又有所不同。顾名思义，Copy-On-Write 就是 CopyOnWriteArrayList 的实现机制。即当对象进行写操作时，复制该对象；若进行的读操作，则直接返回结果，操作过程中不需要进行同步。

**CopyOnWriteArrayList**
- 很好地利用了对象的不变性，在没有对对象进行写操作前，由于对象未发生改变，因此不需要加锁。而在试图改变对象时，总是先获取对象的一个副本，然后对副本进行修改，最后将副本写回。
- 这种实现方式的核心思想是减少锁竞争，从而提高在高并发时的读取性能，但是它却在一定程度上牺牲了写的性能。
- 缺点：最明显的就是这是CopyOnWriteArrayList属于线程安全的，并发的读是没有异常的，读写操作被分离。缺点就是在写入时不止加锁，还使用了Arrays.copyOf()进行了数组复制，性能开销较大，遇到大对象也会导致内存占用较大。

**Vector**
- 在 get() 操作上，Vector 使用了同步关键字，所有的 get() 操作都必须先取得对象锁才能进行。在高并发的情况下，大量的锁竞争会拖累系统性能。反观CopyOnWriteArrayList 的get() 实现，并没有任何的锁操作。

**CopyOnWriteArrayList和Vector对比**

- 在读多写少的高并发环境中，使用 CopyOnWriteArrayList 可以提高系统的性能，但是，在写多读少的场合，CopyOnWriteArrayList 的性能可能不如 Vector。
- 在 add() 操作上，CopyOnWriteArrayList 的写操作性能不如Vector，原因也在于Copy-On-Write。
- CopyOnWriteArrayList的get(int index)方法是没有任何锁处理的，直接返回数组对象。

**Copy-On-Write源码分析**
```java
// 通过查看CopyOnWriteArrayList类的源码可知，在add操作上，是使用了Lock锁做了同步处理，内部拷贝了原数组，并在新数组上进行添加操作，最后将新数组替换掉旧数组。
public boolean add(E e) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] elements = getArray();
        int len = elements.length;
        Object[] newElements = Arrays.copyOf(elements, len + 1);
        newElements[len] = e;
        setArray(newElements);
        return true;
    } finally {
        lock.unlock();
    }
}
```
# 并发Set
Set 不是线程安全的。因此，应该尽量避免在多线程环境中使用Set。如果因为某些原因必须使用的，则需要使用Collections.synchronizedSet(new HashSet())进行包装。
示例代码：
```java
Set s = Collections.synchronizedSet(new HashSet());
    ...
synchronized (s) {
    Iterator i = s.iterator(); // 必须在同步块中
    while (i.hasNext())
        foo(i.next());
}
```

和List相似，并发Set也有一个 CopyOnWriteArraySet ，它实现了 Set 接口，并且是线程安全的。它的内部实现完全依赖于 CopyOnWriteArrayList 
# 写时复制容器

CopyOnWriteArrayList、CopyOnWriteArraySet这两种容器称为写时复制容器

通俗的理解是当我们往一个容器添加元素的时候，不直接往当前容器添加，而是先将当前容器进行Copy，复制出一个新的容器，然后新的容器里添加元素，添加完元素之后，再将原容器的引用指向新的容器。这样做的好处是我们可以对容器进行并发的读，而不需要加锁，因为当前容器不会添加任何元素。所以写时复制容器也是一种读写分离的思想，读和写不同的容器。如果读的时候有多个线程正在向容器添加数据，读还是会读到旧的数据，因为写的时候不会锁住旧的，只能保证最终一致性。

适用读多写少的并发场景，常见应用：白名单/黑名单， 商品类目的访问和更新场景。

缺点：存在内存占用问题。
# 并发Map
在多线程环境下使用Map，一般也可以使用 Collections.synchronizedMap()方法得到一个线程安全的 Map
```java
Map m = Collections.synchronizedMap(new HashMap());
    ...
Set s = m.keySet();  // 不需要同步块
    ...
synchronized (m) {  // 同步在m上，而不是s上!!
    Iterator i = s.iterator(); // 必须在同步块中
    while (i.hasNext())
        foo(i.next());
}
```
但是在高并发的情况下，这个Map的性能表现不是最优的。由于 Map 是使用相当频繁的一个数据结构，因此 JDK 中便提供了专用于高并发的 Map 实现 ConcurrentHashMap。

### 为什么不能在高并发下使用HashMap？
- 数据丢失
- JDK1.7以前，死循环
- ... ...

详细了解？ https://blog.csdn.net/wangjunjie0817/article/details/96740735


### 为什么不使用线程安全的HashTable？

HashTable容器使用synchronized来保证线程安全，但在线程竞争激烈的情况下HashTable的效率非常低下。因为当一个线程访问HashTable的同步方法时，其他线程访问HashTable的同步方法时，可能会进入阻塞或轮询状态。如线程1使用put进行添加元素，线程2不但不能使用put方法添加元素，并且也不能使用get方法来获取元素，所以竞争越激烈效率越低。

### ConcurrentHashMap的优势
ConcurrentHashMap的内部实现进行了锁分离（或锁分段），所以它的锁粒度小于同步的 HashMap；同时，ConcurrentHashMap的 get() 操作也是无锁的。除非读到的值是空的才会加锁重读，我们知道HashTable容器的get方法是需要加锁的，那么ConcurrentHashMap的get操作是如何做到不加锁的呢？原因是它的get方法里将要使用的共享变量都定义成volatile。
锁分离：首先将数据分成一段一段的存储，然后给每一段数据配一把锁，当一个线程占用锁访问其中一个段数据的时候，其他段的数据也能被其他线程访问。有些方法需要跨段，比如size()和containsValue()，它们可能需要锁定整个表而而不仅仅是某个段，这需要按顺序锁定所有段，操作完毕后，又按顺序释放所有段的锁。
详细了解？ https://blog.csdn.net/wangjunjie0817/article/details/96775010    https://juejin.im/post/6844903618143846408

# 更多的并发容器
### 跳表
SkipList，以空间换时间，在原链表的基础上形成多层索引，但是某个节点在插入时，是否成为索引，随机决定，所以跳表又称为概率数据结构。
![image](https://github.com/wangjunjie0817/note/blob/master/images/skip.png)
### ConcurrentSkipListMap 
TreeMap的并发版本
### ConcurrentSkipListSet
TreeSet的并发版本
# 并发Queue
在并发队列上，JDK提供了两套实现，一个是以 ConcurrentLinkedQueue 为代表的高性能队列，一个是以 BlockingQueue 接口为代表的阻塞队列。不论哪种实现，都继承自 Queue 接口。

### ConcurrentLinkedQueue
ConcurrentLinkedQueue是LinkList的并发版本，是一个适用于高并发场景下的队列。它通过无锁的方式，采用CAS+volatile实现并发安全和数据一致性，实现了高并发状态下的高性能。通常，ConcurrentLinkedQueue 的性能要好于 BlockingQueue 。

ConcurrentLinkedQueue的方法：
- add,offer将元素插入到尾部
- peek（拿头部的数据，但是不移除）
- poll（拿头部的数据，但是移除）
### 阻塞队列
阻塞队列遵循生产者消费者模式 
- 当队列满的时候，插入元素的线程被阻塞，直达队列不满。
- 队列为空的时候，获取元素的线程被阻塞，直到队列不空。

生产者和消费者模式
- 生产者就是生产数据的线程，消费者就是消费数据的线程。在多线程开发中，如果生产者处理速度很快，而消费者处理速度很慢，那么生产者就必须等待消费者处理完，才能继续生产数据。同样的道理，如果消费者的处理能力大于生产者，那么消费者就必须等待生产者。为了解决这种生产消费能力不均衡的问题，便有了生产者和消费者模式。生产者和消费者模式是通过一个容器来解决生产者和消费者的强耦合问题。生产者和消费者彼此之间不直接通信，而是通过阻塞队列来进行通信，所以生产者生产完数据之后不用等待消费者处理，直接扔给阻塞队列，消费者不找生产者要数据，而是直接从阻塞队列里取，阻塞队列就相当于一个缓冲区，平衡了生产者和消费者的处理能力。

常用方法
![image](https://github.com/wangjunjie0817/note/blob/master/images/method.png)
- 抛出异常：当队列满时，如果再往队列里插入元素，会抛出IllegalStateException（"Queuefull"）异常。当队列空时，从队列里获取元素会抛出NoSuchElementException异常。
- 返回特殊值：当往队列插入元素时，会返回元素是否插入成功，成功返回true。如果是移除方法，则是从队列里取出一个元素，如果没有则返回null。
- 一直阻塞：当阻塞队列满时，如果生产者线程往队列里put元素，队列会一直阻塞生产者线程，直到队列可用或者响应中断退出。当队列空时，如果消费者线程从队列里take元素，队列会阻塞住消费者线程，直到队列不为空。
- 超时退出：当阻塞队列满时，如果生产者线程往队列里插入元素，队列会阻塞生产者线程一段时间，如果超过了指定的时间，生产者线程就会退出。

常用阻塞队列 
- ArrayBlockingQueue：一个由数组结构组成的有界阻塞队列。
按照先进先出原则，要求设定初始大小
- LinkedBlockingQueue：一个由链表结构组成的有界阻塞队列。
按照先进先出原则，可以不设定初始大小，Integer.Max_Value
- PriorityBlockingQueue：一个支持优先级排序的无界阻塞队列。
默认情况下，按照自然顺序，要么实现compareTo()方法，指定构造参数Comparator
- DelayQueue：一个使用优先级队列实现的无界阻塞队列。
支持延时获取的元素的阻塞队列，元素必须要实现Delayed接口,实现getDelay和compareTo接口即可。适用场景：实现自己的缓存系统，订单到期，限时支付等等。
- SynchronousQueue：一个不存储元素的阻塞队列。
每一个put操作都要等待一个take操作
- LinkedTransferQueue：一个由链表结构组成的无界阻塞队列。
transfer()，必须要消费者消费了以后方法才会返回，tryTransfer()无论消费者是否接收，方法都立即返回。

ArrayBlockingQueue和LinkedBlockingQueue不同：
- 锁上面：ArrayBlockingQueue只有一个锁，LinkedBlockingQueue用了两个锁，
实现上：ArrayBlockingQueue直接插入元素，LinkedBlockingQueue需要转换。

# 并发Deque
在JDK1.6中，还提供了一种双端队列（Double-Ended Queue），简称Deque。Deque允许在队列的头部或尾部进行出队和入队操作。与Queue相比，具有更加复杂的功能。

Deque 接口的实现类：LinkedList、ArrayDeque和LinkedBlockingDeque。

- LinkedBlockingDeque：一个由链表结构组成的双向阻塞队列。
可以从队列的头和尾都可以插入和移除元素，实现工作密取，方法名带了First对头部操作，带了last从尾部操作，另外：add=addLast;	remove=removeFirst;	take=takeFirst

它们都实现了双端队列Deque接口。其中LinkedList使用链表实现了双端队列，ArrayDeque使用数组实现双端队列。通常情况下，由于ArrayDeque基于数组实现，拥有高效的随机访问性能，因此ArrayDeque具有更好的遍性能。但是当队列的大小发生变化较大时，ArrayDeque需要重新分配内存，并进行数组复制，在这种环境下，基于链表的 LinkedList 没有内存调整和数组复制的负担，性能表现会比较好。但无论是LinkedList或是ArrayDeque，它们都不是线程安全的。

LinkedBlockingDeque 是一个线程安全的双端队列实现。可以说，它已经是最为复杂的一个队列实现。在内部实现中，LinkedBlockingDeque 使用链表结构。每一个队列节点都维护了一个前驱节点和一个后驱节点。LinkedBlockingDeque 没有进行读写锁的分离，因此同一时间只能有一个线程对其进行操作。因此，在高并发应用中，它的性能表现要远远低于 LinkedBlockingQueue，更要低于 ConcurrentLinkedQueue 。
