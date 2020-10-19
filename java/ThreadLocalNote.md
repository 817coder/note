# ThreadLocal

在这里先记一下没有在这个笔记下面的知识点：
- InheritableThreadLocal
- 哈希冲突问题

### 什么是ThreadLocal
- 它能让线程拥有了自己内部独享的变量
- 每一个线程可以通过get、set方法去进行操作
- 可以覆盖initialValue方法指定线程独享的值
- 通常会用来修饰类里private static final的属性，为线程设置一些状态信息，例如user ID或者Transaction ID
- 每一个线程都有一个指向threadLocal实例的弱引用，只要线程一直存活或者该threadLocal实例能被访问到，都不会被垃圾回收清理掉

### demo
```java
public class Test {

    // Atomic integer containing the next thread ID to be assigned
    private static final AtomicInteger nextId = new AtomicInteger(0);

    // Thread local variable containing each thread's ID
    private static final ThreadLocal<Integer> threadId = new ThreadLocal<Integer>() {
        @Override
        protected Integer initialValue() {
            return nextId.getAndIncrement();
        }
    };

    // Returns the current thread's unique ID, assigning it if necessary
    public static int get() {
        return threadId.get();
    }


    public static void main(String[] args) {
        for (int i = 0; i < 5; i++) {
            new Thread(new Runnable() {
                @Override
                public void run() {
                    System.out.println("threadName=" + Thread.currentThread().getName() + ",threadId=" + threadId.get());
                }
            }).start();
        }
    }
}
```
运行结果如下：
```java
1threadName=Thread-0,threadId=0
2threadName=Thread-1,threadId=1
3threadName=Thread-2,threadId=2
4threadName=Thread-3,threadId=3
5threadName=Thread-4,threadId=4
```

### get()源码
```java
public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    return setInitialValue();
}
```
逻辑很简单：

- 获取当前线程内部的ThreadLocalMap
- map存在则获取当前ThreadLocal对应的value值
- map不存在或者找不到value值，则调用setInitialValue，进行初始化

### setInitialValue()源码
```java
private T setInitialValue() {
    T value = initialValue();
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
    return value;
}
```

逻辑也很简单：

- 调用initialValue方法，获取初始化值【调用者通过覆盖该方法，设置自己的初始化值】
- 获取当前线程内部的ThreadLocalMap
- map存在则把当前ThreadLocal和value添加到map中
- map不存在则创建一个ThreadLocalMap，保存到当前线程内部

### set(T value)源码

```java
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}
```

逻辑很简单：

- 获取当前线程内部的ThreadLocalMap
- map存在则把当前ThreadLocal和value添加到map中
- map不存在则创建一个ThreadLocalMap，保存到当前线程内部

### remove源码

```java
public void remove() {
    ThreadLocalMap m = getMap(Thread.currentThread());
    if (m != null)
     m.remove(this);
}
```

就一句话，获取当前线程内部的ThreadLocalMap，存在则从map中删除这个ThreadLocal对象。

### ThreadLocalMap

每一个线程都有一个私有变量，是ThreadLocalMap类型。当为线程添加ThreadLocal对象时，就是保存到这个map中，所以线程与线程间不会互相干扰。

- ThreadLocalMap是一个自定义的hash map，专门用来保存线程的thread local变量
- 它的操作仅限于ThreadLocal类中，不对外暴露
- 这个类被用在Thread类的私有变量threadLocals和inheritableThreadLocals上
- 为了能够保存大量且存活时间较长的threadLocal实例，hash table entries采用了WeakReferences作为key的类型
- 一旦hash table运行空间不足时，key为null的entry就会被清理掉

看一下类的的定义

```java
static class ThreadLocalMap {

    // hash map中的entry继承自弱引用WeakReference，指向threadLocal对象
    // 对于key为null的entry，说明不再需要访问，会从table表中清理掉
    // 这种entry被成为“stale entries”
    static class Entry extends WeakReference<ThreadLocal<?>> {
        /** The value associated with this ThreadLocal. */
        Object value;

        Entry(ThreadLocal<?> k, Object v) {
            super(k);
            value = v;
        }
    }

    private static final int INITIAL_CAPACITY = 16;

    private Entry[] table;

    private int size = 0;

    private int threshold; // Default to 0

    private void setThreshold(int len) {
        threshold = len * 2 / 3;
    }

    ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
        table = new Entry[INITIAL_CAPACITY];
        int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
        table[i] = new Entry(firstKey, firstValue);
        size = 1;
        setThreshold(INITIAL_CAPACITY);
    }
}
```

当创建一个ThreadLocalMap时，实际上内部是构建了一个Entry类型的数组，初始化大小为16，阈值threshold为数组长度的2/3，Entry类型为WeakReference，有一个弱引用指向ThreadLocal对象。


### 为什么Entry采用WeakReference类型？

Java垃圾回收时，看一个对象需不需要回收，就是看这个对象是否可达。什么是可达，就是能不能通过引用去访问到这个对象。(当然，垃圾回收的策略远比这个复杂，这里为了便于理解，简单给大家说一下)。

jdk1.2以后，引用就被分为四种类型：强引用、弱引用、软引用和虚引用。强引用就是我们常用的Object obj = new Object()，obj就是一个强引用，指向了对象内存空间。当内存空间不足时，
Java垃圾回收程序发现对象有一个强引用，宁愿抛出OutofMemory错误，也不会去回收一个强引用的内存空间。而弱引用，即WeakReference，意思就是当一个对象只有弱引用指向它时，垃圾回收器
不管当前内存是否足够，都会进行回收。反过来说，这个对象是否要被垃圾回收掉，取决于是否有强引用指向。ThreadLocalMap这么做，是不想因为自己存储了ThreadLocal对象，而影响到它的垃圾回收，
而是把这个主动权完全交给了调用方，一旦调用方不想使用，设置ThreadLocal对象为null，内存就可以被回收掉。

### 内存溢出问题

这里需要注意一下对象的引用     
  thread ——> ThreadLocalMap ——> threadLocal ——> entry（key和value）
entry对象是继承自WeakRefereace的，key是ThreadLocal对象，value是保存的值，如果threadLocal对象被回收了，那么entry就会被回收

例如下面的代码就会导致内存溢出：
```java
public class ThreadLocalDemo {

    static Map<Thread, ThreadLocal> map = new HashMap<>();

    public static void main(String[] args) {
        new Thread(new Runnable() {
            @Override
            public void run() {
                for (int i = 0; i < 1000; i++) {
                    TestClass t = new TestClass(i);
                    t.printId();
                    t = null;
                }
            }
        }).start();

    }

    static class TestClass {
        private int id;
        private int[] arr;
        private ThreadLocal<TestClass> threadLocal;

        TestClass(int id) {
            this.id = id;
            arr = new int[1000000];
            threadLocal = new ThreadLocal<>();
            threadLocal.set(this);
        }

        public void printId() {
            System.out.println(threadLocal.get().id);
        }
    }
}
```
究其原因就是threadLocal对象依然有强引用关联，此时可以通过以下代码手动释放threadLocal对象
```java
for (int i = 0; i < 1000; i++) {
    TestClass t = new TestClass(i);
    t.printId();
    t.threadLocal = null;    // 增加一行代码
    t = null;
}
```

除此之外，可以通过
```java
for (int i = 0; i < 1000; i++) {
    TestClass t = new TestClass(i);
    t.printId();
    t.threadLocal.remove();    // 增加一行代码，推荐
    t = null;
}
```











