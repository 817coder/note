# Synchronized原理解析

### 简介
synchronized关键字可以保证方法或者代码块在执行的之后，同一时刻只有一个线程可以进入到临界区，同时他还可以保证共享变量的内存可见性。

synchronized 是一个重量级锁，但是在随着 Javs SE 1.6 对 synchronized 进行的各种优化后，synchronized 并不会显得那么重了。

Java 中每一个对象都可以作为锁，这是 synchronized 实现同步的基础：

- 普通同步方法，锁是当前实例对象
- 静态同步方法，锁是当前类的 class 对象
- 同步方法块，锁是括号里面的对象

当线程访问同步代码块时，首先需要获得锁才能继续执行，当退出或者抛出异常时必须要释放锁。那么，synchronized是如何实现的呢？

我们可以反编译class文件，可以看到
- 同步代码块是使用 monitorenter 和 monitorexit 指令实现的。monitorenter 指令插入到同步代码块的开始位置，monitorexit 指令插入到同步代码块的结束位置，JVM 需要保证每一个 monitorenter 都有一个 monitorexit 与之相对应。任何对象都有一个 Monitor 与之相关联，当且一个 Monitor 被持有之后，他将处于锁定状态。线程执行到 monitorenter 指令时，将会尝试获取对象所对应的 Monitor 所有权，即尝试获取对象的锁。
- 同步方法是依靠的是方法修饰符上的ACC_SYNCHRONIZED 实现。synchronized 方法则会被翻译成普通的方法调用和返回指令如：invokevirtual、areturn 指令，在 VM 字节码层面并没有任何特别的指令来实现被synchronized 修饰的方法，而是在 Class 文件的方法表中将该方法的 access_flags 字段中的 synchronized 标志位置设置为 1，表示该方法是同步方法，并使用调用该方法的对象或该方法所属的 Class 在 JVM 的内部对象表示 Klass 作为锁对象。

### 为什么重量锁那么重？

简单来说，在 JVM 中 monitorenter 和 monitorexit 字节码依赖于底层的操作系统的Mutex Lock 来实现的，但是由于使用 Mutex Lock 需要将当前线程挂起并从用户态切换到内核态来执行，这种切换的代价是非常昂贵的。然而，在现实中的大部分情况下，同步方法是运行在单线程环境（无锁竞争环境），如果每次都调用 Mutex Lock 那么将严重的影响程序的性能。

### 对象头

Java对象结构中的对象头描述部分是实现锁机制的关键，实际上在HotSpot JVM 虚拟机的工作中将对象结构分为三大块区域：对象头（Header）、实例数据（Instance Data）和对齐填充区域（可能存在）

对象头主要包括两部分数据：Mark Word（标记字段）、Klass Pointer（类型指针）。

Mark Word 用于存储对象自身的运行时数据，如哈希码（HashCode）、GC 分代年龄、锁状态标志、线程持有的锁、偏向线程 ID、偏向时间戳等等，是对象关键的运行时数据。

这里我们重点讨论和synchronized加锁过程有关的markword区域，首先需要说明几点：
根据对象所处的锁状态的不同，markword区域的存储结构会发生变动。例如当对象处于轻量级锁状态的情况下，markword区域的存储结构是一种定义；而当对象锁级别处于偏向锁状态的情况下，markword区域的存储结构又是另一种定义
![image](https://github.com/wangjunjie0817/note/blob/master/images/sync.png)

### MonitorRecord


### Object Monitor

什么是 Monitor ？我们可以把它理解为一个同步工具，也可以描述为一种同步机制，它通常被描述为一个对象。与一切皆对象一样，所有的 Java 对象是天生的 Monitor ，每一个 Java 对象都有成为Monitor 的潜质，因为在 Java 的设计中 ，每一个 Java 对象自打娘胎里出来就带了一把看不见的锁，它叫做内部锁或者 Monitor 锁。

看一下Object Monitor的定义
![image](https://github.com/wangjunjie0817/note/blob/master/images/monitorObject.png)

首先在HotSpot虚拟机中，monitor采用ObjectMonitor实现，每个线程都具有两个队列，分别为free和used，用来存放ObjectMonitor。如果当前free列表为空，线程将向全局global list请求分配ObjectMonitor。

ObjectMonitor对象中有两个队列，都用来保存ObjectWaiter对象，分别是_WaitSet 和 _EntrySet。_owner用来指向获得ObjectMonitor对象的线程

ObjectWaiter对象是双向链表结构，保存了_thread（当前线程）以及当前的状态TState等数据， 每个等待锁的线程都会被封装成ObjectWaiter对象。

![image](https://github.com/wangjunjie0817/note/blob/master/images/monitorObject2.jpeg)

ObjectMonitor的关键属性

- _owner：指向持有ObjectMonitor对象的线程

- _WaitSet：存放处于wait状态的线程队列

- _EntryList：存放处于等待锁block状态的线程队列

- _recursions：锁的重入次数

- _count：用来记录该线程获取锁的次数


ObjectWaiter对象是双向链表结构，保存了_thread（当前线程）以及当前的状态TState等数据， 每个等待锁的线程都会被封装成ObjectWaiter对象。

wait方法实现: lock.wait()方法最终通过ObjectMonitor的void wait(jlong millis, bool interruptable, TRAPS);

1、将当前线程封装成ObjectWaiter对象node；

2、通过ObjectMonitor::AddWaiter方法将node添加到_WaitSet列表中；

3、通过ObjectMonitor::exit方法释放当前的ObjectMonitor对象，这样其它竞争线程就可以获取该ObjectMonitor对象。

4、最终底层的park方法会挂起线程；

notify方法实现: lock.notify()方法最终通过ObjectMonitor的void notify(TRAPS)实现：

1、如果当前_WaitSet为空，即没有正在等待的线程，则直接返回；
2、通过ObjectMonitor::DequeueWaiter方法，获取_WaitSet列表中的第一个ObjectWaiter节点，实现也很简单。这里需要注意的是，在jdk的notify方法注释是随机唤醒一个线程，其实是第一个ObjectWaiter节点
3、根据不同的策略，将取出来的ObjectWaiter节点，加入到_EntryList或则通过Atomic::cmpxchg_ptr指令进行自旋操作cxq，具体代码实现有点长，这里就不贴了，有兴趣的同学可以看objectMonitor::notify方法；

notifyAll方法实现

lock.notifyAll()方法最终通过ObjectMonitor的void notifyAll(TRAPS)实现：
通过for循环取出_WaitSet的ObjectWaiter节点，并根据不同策略，加入到_EntryList或则进行自旋操作。
从JVM的方法实现中，可以发现：notify和notifyAll并不会释放所占有的ObjectMonitor对象，其实真正释放ObjectMonitor对象的时间点是在执行monitorexit指令，一旦释放ObjectMonitor对象了，entry set中ObjectWaiter节点所保存的线程就可以开始竞争ObjectMonitor对象进行加锁操作了。

### 锁优化
JDK 1.6 对锁的实现引入了大量的优化，如自旋锁、适应性自旋锁、锁消除、锁粗化、偏向锁、轻量级锁等技术来减少锁操作的开销。

自旋锁：
  - 自旋锁实际上是一种基于CAS原理的实现方式（关于CAS原理在本专题之前的文章中已经介绍过，从根本上来说这是一个“乐观锁”设计思想的具体实现）。自旋锁就是在满足一定条件下，让当前还没有获得对象操作权的线程进入一种“自循环”的等待状态，而不是真正让这个线程释放CPU资源。这个等待状态的尝试时间和自旋的次数非常短，如果在这个非常短的时间内该对象还没有获得对象操作权，则锁状态就会升级。
  - 自旋锁的特点是，由于等待线程并没有切换出CPU资源，而是使用“自循环”的方式将自己保持在CPU L1/L2缓存中，这样就避免了线程在CPU的切换过程，在实际上并没有什么并发量的执行环境下减少了程序的处理时间。当基于当前对象的synchronized控制还处于“自旋锁”状态时，实际上并没有真正开启Object Monitor控制机制，所以这个自旋锁状态（包括偏向锁、轻量级锁）并不会反映在对象头的数据结构中。
  
适应自旋锁
- JDK 1.6 引入了更加聪明的自旋锁，即自适应自旋锁。所谓自适应就意味着自旋的次数不再是固定的，它是由前一次在同一个锁上的自旋时间及锁的拥有者的状态来决定。它怎么做呢？有了自适应自旋锁，随着程序运行和性能监控信息的不断完善，虚拟机对程序锁的状况预测会越来越准确，虚拟机会变得越来越聪明。
  - 线程如果自旋成功了，那么下次自旋的次数会更加多，因为虚拟机认为既然上次成功了，那么此次自旋也很有可能会再次成功，那么它就会允许自旋等待持续的次数更多。
  - 反之，如果对于某个锁，很少有自旋能够成功的，那么在以后要或者这个锁的时候自旋的次数会减少甚至省略掉自旋过程，以免浪费处理器资源。
  
锁消除
- 锁削除是指虚拟机即时编译器在运行时，对一些代码上要求同步，但是被检测到不可能存在共享数据竞争的锁进行削除。锁削除的主要判定依据来源于逃逸分析的数据支持，如果判断到一段代码中，在堆上的所有数据都不会逃逸出去被其他线程访问到，那就可以把它们当作栈上数据对待，认为它们是线程私有的，同步加锁自然就无须进行。 但是变量是否逃逸，对于虚拟机来说需要使用数据流分析来确定，但是程序员自己应该是很清楚的，怎么会在明知道不存在数据争用的情况下要求同步呢？答案是有许多同步措施并不是程序员自己加入的，如 StringBuffer、Vector、HashTable 等，这个时候会存在隐性的加锁操作。

锁粗化
- 锁粗话概念比较好理解，就是将多个连续的加锁、解锁操作连接在一起，扩展成一个范围更大的锁。

### 锁升级
通常情况下，我们在代码中使用synchronized关键字协调多个线程的工作过程，实际上就是使用Object Monitor控制对多个线程的工作过程进行协调。synchronized关键字的执行过程从传统的理解上就是“悲观锁”设计思想的一种实现。但实际上synchronized关键字的执行过程还涉及到锁机制的升级过程，升级顺序为 自旋锁、偏向锁、轻量级锁、重量级锁。

偏向锁：
- 偏向锁实际上是在没有多线程对指定对象进行操作权抢占的情况下，完全取消针对这个指定对象的同步操作元语。而当前唯一请求对象操作权的线程，将被对象记录到对象头中。这样一来如果一直没有出现其它线程抢占对象操作权的情况下，则当前同步代码块就基本上不会出现针对锁的额外处理。

- 举一个栗子，当前进程中只有线程A在请求同步代码块X的对象操作权（对象记为Y），这时synchronized控制机制就会将Y对象的对象头记为“偏向锁”。这时线程A依然在执行同步代码块X时，又有另一个线程B试图抢占对象Y的操作权。如果线程B通过“自旋”操作等待后依然没有获取到对象Y的操作权，则锁升级为轻量级锁。

轻量级锁：
- 按照之前对偏向锁的描述，偏向锁主要解决在没有对象抢占的情况下，由单个线程进度同步块时的加锁问题。一旦出现了两个或多个线程抢占对象操作时，偏向锁就会升级为轻量级锁。轻量级锁同样使用CAS技术进行实现，它主要说的是多个需要抢占对象操作权的线程，通过CAS的是实现技术持续尝试获得对象的操作权的过程。

按照轻量级锁的定义，我们将以上的栗子继续下去。当前对象Y的锁级别升级为轻量级锁后，JVM将在线程A、线程B和之后请求获得对象Y操作的若干线程的当前栈帧中，添加一个锁记录空间（记为Key空间），并将对象头中的Mark Word复制到锁记录中。然后线程会持续尝试使用CAS原理将对象头中的Mark Word部分替换为指向本线程锁记录空间的指针。如果替换成功则当前线程获得这个对象的操作权；如果多次CAS持续失败，说明当前对象的多线程抢占现象很严重，这是对象锁升级为重量锁状态，并使用操作系统层面的Mutex Lock（互斥锁）技术进行实现。

重量级锁：
- 当对象的锁级别升级为“重量级锁”时，JVM就开始采用Object Monitor机制控制各线程抢占对象的过程了。实际上这是JVM对操作系统级别Mutex Lock（互斥锁）的管理过程。

### 写在最后
不清楚面试对于synchronized考察的深度要求，更具体的概念例如Monitor Record等信息看芋艿的源码解析吧！！！
