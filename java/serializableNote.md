序列化是将对象变为可传输内容的过程, 反序列化则是将可传输内容转化为对象的过程.

Java原生序列化方式是通过实现Serializable接口实现的. 不实现该接口会导致无法序列化, 抛出异常如下:

> java.io.NotSerializableException

序列化的应用场景：

> 将对象转换为字节流, 用于网络传输, 例如用于RPC远程调用。

> 将对象保存到磁盘, 例如tomcat的钝化和活化.

Java原生序列化是通过IO包中的ObjectInputStream和ObjectOutputStream实现的。ObjectOutputStream类负责实现序列化, ObjectInputStream类负责实现反序列化。

### Java序列化反序列化实例
```java
public class Test {
    
    public static void main(String[] args) throws Exception {
        File file = new File("person.txt");
        
        ObjectOutputStream objectOutputStream = new ObjectOutputStream(new FileOutputStream(file));
        Person person = new Person(3, "abc");
        objectOutputStream.writeObject(person);
        objectOutputStream.close();

        ObjectInputStream objectInputStream = new ObjectInputStream(new FileInputStream(file));
        Object readObject = objectInputStream.readObject();
        objectInputStream.close();
        
        Person newPerson = (Person) readObject;
        System.out.println(person == newPerson); // Person [age=3, name=abc]
        System.out.println(newPerson);
    }
}

/**
 * Person类
 */
class Person implements Serializable {

    private static final long serialVersionUID = 1L;

    private Integer age;
    private String name;
    
    ... // 省略构造函数/getter/setter/toString方法
}
```
### transient
出于安全问题，有时候不会把指定字段进行序列化保存，例如密码, 此处以Person对象的age字段为例, 为age字段添加transient修饰后，默认序列化机制会忽略它。如下:
```java
public class Test {
    
    public static void main(String[] args) throws Exception {
        File file = new File("person.txt");
        
        ObjectOutputStream objectOutputStream = new ObjectOutputStream(new FileOutputStream(file));
        Person person = new Person(3, "abc");
        objectOutputStream.writeObject(person);
        objectOutputStream.close();

        ObjectInputStream objectInputStream = new ObjectInputStream(new FileInputStream(file));
        Person newPerson = (Person)objectInputStream.readObject();
        objectInputStream.close();
        
        System.out.println(newPerson); // Person [age=null, name=abc]
    }
}

class Person implements Serializable {

    private static final long serialVersionUID = 1L;

    private transient Integer age;
    private String name;
    
    ... // 省略构造函数/getter/setter/toString方法
}
```
如果transient修饰的字段也需要序列化和反序列化，可以使用readObject和writeObject方法。

我们只需要在当前 Person 类中添加 readObject() 和 writeObject() 方法，在 writeObject 方法中实现对 age 的字段赋值，就可以使age字段被序列化到字节流中；在 readObject 方法中实现对 age 字段读取，并赋值给Person对象即可。
如下：
```java
class Person implements Serializable {

    private static final long serialVersionUID = 1L;

    private transient Integer age;
    private String name;
    
      private void readObject(ObjectInputStream s) throws Exception {
        s.defaultReadObject();
        this.age = (Integer)s.readObject();
    }

    private void writeObject(ObjectOutputStream s) throws Exception {
        s.defaultWriteObject();
        this.age = 10000;
             s.writeObject(this.age);
    }
    ... // 省略构造函数/getter/setter/toString方法
}
```
readObject和writeObject在本文后段内容会讲解.

这里的defaultWriteObject方法作用是序列化非transient字段以及非静态字段; 这里的defaultReadObject方法作用是反序列化非transient字段以及非静态字段; 当类中含有非transient字段时, 一定要加上这两个方法.

如果被序列化的类中存在多个transient的字段, 序列化时需要如下操作:
```java
class Person implements Serializable {

    private static final long serialVersionUID = 1L;

    private transient Integer age;
    private transient String name;
    private transient Integer money;
    
      private void readObject(ObjectInputStream s) throws Exception {
        s.defaultReadObject();
        this.age = (Integer) stream.readObject();
        this.money = (Integer) stream.readObject();
        this.name = (String) stream.readObject();
    }

    private void writeObject(ObjectOutputStream s) throws Exception {
        s.defaultWriteObject();
        stream.writeObject(this.age);
        stream.writeObject(this.money);
        stream.writeObject(this.name);
    }
    ... // 省略构造函数/getter/setter/toString方法
}
```
> 这里writeObject和readObject的使用是有顺序的, 例如第一次writeObject是将age作为Object写入, 所以第一次调用readObject读到的对象就一定是age; 所以, 写入的顺序是age,money,name, 读取时候的顺序一定也要是age,money,name.

> 上面理论上会写入了四个对象, 第一个是defaultWriteObject写入的Person对象, 之后写入的是age(Integer对象), money(Integer), name(String).

> 当然这里defaultWriteObject没有写入, 因为所有成员字段都是transient修饰, 所以实际上只有三个对象(age,money,name). 换句话说, 当所有字段被transient修饰时, 可以不用defaultWriteObject和defaultReadObject.

### serialVersionUID
当person对象被序列化保存到person.txt文件时, 在Person对象中添加新的属性address, 只执行反序列化代码, 如下:
```java
public class Test {
    
    public static void main(String[] args) throws Exception {
        File file = new File("person.txt");
        
        ObjectInputStream objectInputStream = new ObjectInputStream(new FileInputStream(file));
        Person newPerson = (Person)objectInputStream.readObject();
        objectInputStream.close();
        
        System.out.println(newPerson); // Person [age=null, name=abc]
    }
}

class Person implements Serializable {

    private static final long serialVersionUID = 1L;

    private transient Integer age;
    private String name;
    private String address; // 新添加的属性
    
    ... // 省略构造函数/getter/setter/toString方法
}
```
输出依然是:

> Person [age=null, name=abc]
但是我们删除serialVersionUID字段后再次执行, 如下:
```java
public class Test {
    
    public static void main(String[] args) throws Exception {
        File file = new File("person.txt");
        
        ObjectInputStream objectInputStream = new ObjectInputStream(new FileInputStream(file));
        Person newPerson = (Person)objectInputStream.readObject();
        objectInputStream.close();
        
        System.out.println(newPerson); // Person [age=null, name=abc]
    }
}

class Person implements Serializable {

    private transient Integer age;
    private String name;
    private String address; // 新添加的属性
    
    ... // 省略构造函数/getter/setter/toString方法
}
报错:
```
Exception in thread "main" java.io.InvalidClassException: wotest.test.Person; 
local class incompatible: stream classdesc serialVersionUID = 1, 
local class serialVersionUID = 3229018537912438741
```
原因:

> 没有定义serialVersionUID值, 反序列化可能会出现local class incompatible异常, 是Java的安全机制.

> 当序列化对象时, 如果该对象所属类没有serialVersionUID, Java编译器会对jvm中该类的Class文件进行摘要算法生成一个

> serialVersionUID(version1), 并保存在序列化结果中.

> 当反序列化时, jvm会再次对jvm中Class文件摘要生成一个serialVersionUID(version2). 当且仅当version1=version2时, 才会将反序列化结果加载入jvm中, 否则jvm会判断为不安全, 拒绝载入并抛出local class incompatible异常.

> 这样存在的问题就是, 当对象被序列化后, 其所属类只要进行过类名称,它所实现的接口的名称,以及所有成员名称的修改, 会导致摘要算法算出的serialVersionUID变化.

从而version1 != version2导致抛出异常.

> 例如序列化对象存储在磁盘中后, jvm停止, 对其所属类进行修改. 再次启动jvm, 对该对象序列化时就会抛异常.

由此可知, 反序列化时会通过比较serialVersionUID进行判断反序列化内容是否安全, 所以添加如下声明:

> private static final long serialVersionUID = 3229018537912438741L;

将serialVersionUID值固定下来, 可以防止这种情况下的反序列化失败.

另外, 如果 User 对象升级版本，修改了结构，而且不想兼容之前的版本，那么只需要修改下 serialVersionUID 的值就可以了。

建议，每个需要序列化的对象，都要添加一个 serialVersionUID 字段。

### ObjectInputStream
```java
ObjectInputStream objectInputStream = new ObjectInputStream(new FileInputStream(file));
Person newPerson = (Person)objectInputStream.readObject();
```
反序列化，从流中读取使用的方法为readObject。

### readObject
源码如下：
```java
    public final Object readObject() throws IOException, ClassNotFoundException {
        if (enableOverride) { // 1. 判断是否重写了readObject方法
            return readObjectOverride();
        }

        // if nested read, passHandle contains handle of enclosing object
        int outerHandle = passHandle;
        try {
            Object obj = readObject0(false); // 2. 主要的反序列化方法
            handles.markDependency(outerHandle, passHandle);
            ClassNotFoundException ex = handles.lookupException(passHandle);
            if (ex != null) {
                throw ex;
            }
            if (depth == 0) {
                vlist.doCallbacks();
            }
            return obj;
        } finally {
            passHandle = outerHandle;
            if (closed && depth == 0) {
                clear();
            }
        }
    }
```

> 1. 判断子类是否重写了readObject方法，因为readObject是final修饰，所以实质上重写的是readObjectOveride方法，如果重写了，则直接执行子类重写的readObjectOverride方法。

> 2. readObject0才是执行反序列化的主要方法。

> readObject相当于一个公有的构造器, 其以字节流作为唯一参数.

### readObject0
源码如下：
```java
    private Object readObject0(boolean unshared) throws IOException {
        boolean oldMode = bin.getBlockDataMode();
        if (oldMode) {
            int remain = bin.currentBlockRemaining();
            if (remain > 0) {
                throw new OptionalDataException(remain);
            } else if (defaultDataEnd) {
                /*
                 * Fix for 4360508: stream is currently at the end of a field
                 * value block written via default serialization; since there
                 * is no terminating TC_ENDBLOCKDATA tag, simulate
                 * end-of-custom-data behavior explicitly.
                 */
                throw new OptionalDataException(true);
            }
            bin.setBlockDataMode(false);
        }

        byte tc;
        while ((tc = bin.peekByte()) == TC_RESET) {
            bin.readByte();
            handleReset();
        }

        depth++;
        totalObjectRefs++;
        try {
            switch (tc) { // 1. 选择反序列化方式
                case TC_NULL:
                    return readNull();

                case TC_REFERENCE:
                    return readHandle(unshared);

                case TC_CLASS:
                    return readClass(unshared);

                case TC_CLASSDESC:
                case TC_PROXYCLASSDESC:
                    return readClassDesc(unshared);

                case TC_STRING:
                case TC_LONGSTRING:
                    return checkResolve(readString(unshared));

                case TC_ARRAY:
                    return checkResolve(readArray(unshared));

                case TC_ENUM:
                    return checkResolve(readEnum(unshared));

                case TC_OBJECT: // 2. 对象类型的反序列化
                    return checkResolve(readOrdinaryObject(unshared));

                case TC_EXCEPTION:
                    IOException ex = readFatalException();
                    throw new WriteAbortedException("writing aborted", ex);

                case TC_BLOCKDATA:
                case TC_BLOCKDATALONG:
                    if (oldMode) {
                        bin.setBlockDataMode(true);
                        bin.peek();             // force header read
                        throw new OptionalDataException(
                            bin.currentBlockRemaining());
                    } else {
                        throw new StreamCorruptedException(
                            "unexpected block data");
                    }

                case TC_ENDBLOCKDATA:
                    if (oldMode) {
                        throw new OptionalDataException(true);
                    } else {
                        throw new StreamCorruptedException(
                            "unexpected end of block data");
                    }

                default:
                    throw new StreamCorruptedException(
                        String.format("invalid type code: %02X", tc));
            }
        } finally {
            depth--;
            bin.setBlockDataMode(oldMode);
        }
    }
```
> 1. 根据选择反序列化方法

> 2. 我们这里是反序列化对象，所以会执行readOrdinaryObject方法。

### readOrdinaryObject
源码如下：
```java
    private Object readOrdinaryObject(boolean unshared) throws IOException {
        if (bin.readByte() != TC_OBJECT) { // 类型检查
            throw new InternalError();
        }

        ObjectStreamClass desc = readClassDesc(false);
        desc.checkDeserialize();

        Class<?> cl = desc.forClass(); // 获取到了Class对象
        if (cl == String.class || cl == Class.class || cl == ObjectStreamClass.class) {
            throw new InvalidClassException("invalid class descriptor");
        }

        Object obj;
        try {
            obj = desc.isInstantiable() ? desc.newInstance() : null; // 1. 在这里进行了对象的创建
        } catch (Exception ex) {
            throw (IOException) new InvalidClassException(desc.forClass().getName(), "unable to create instance").initCause(ex);
        }

        passHandle = handles.assign(unshared ? unsharedMarker : obj);
        ClassNotFoundException resolveEx = desc.getResolveException();
        if (resolveEx != null) {
            handles.markException(passHandle, resolveEx);
        }

        if (desc.isExternalizable()) { // 2. 判断序列化方式
            readExternalData((Externalizable) obj, desc);
        } else {
            readSerialData(obj, desc);
        }

        handles.finish(passHandle);

        if (obj != null && handles.lookupException(passHandle) == null &&
            desc.hasReadResolveMethod()) { // 3. 判断是否包含readResolve方法
            Object rep = desc.invokeReadResolve(obj);
            if (unshared && rep.getClass().isArray()) {
                rep = cloneArray(rep);
            }
            if (rep != obj) {
                // Filter the replacement object
                if (rep != null) {
                    if (rep.getClass().isArray()) {
                        filterCheck(rep.getClass(), Array.getLength(rep));
                    } else {
                        filterCheck(rep.getClass(), -1);
                    }
                }
                handles.setObject(passHandle, obj = rep);
            }
        }

        return obj;
    }
```
> 1. 反序列化创建了新的对象，这就是反序列化可以破坏单例的原因；

> 2. 判断序列化方式，可以看到实现Externalizable接口的方式优先级要高于实现Serializable接口的方式，因为我们使用的是Serializable方式，所以会执行readSerialData方法。

> 3. 判断被序列化的类是否包含readResolve方法，如果包含，则执行readResolve方法，并使用该方法返回的对象替换之前创建的obj（代码obj=rep进行的替换）。

### readSerialData
源码如下：
```java
    private void readSerialData(Object obj, ObjectStreamClass desc) throws IOException {
        ObjectStreamClass.ClassDataSlot[] slots = desc.getClassDataLayout();
        for (int i = 0; i < slots.length; i++) {
            ObjectStreamClass slotDesc = slots[i].desc;

            if (slots[i].hasData) {
                if (obj == null || handles.lookupException(passHandle) != null) {
                    defaultReadFields(null, slotDesc); // skip field values
                } else if (slotDesc.hasReadObjectMethod()) { // 1. 判断是否有readObject方法
                    ThreadDeath t = null;
                    boolean reset = false;
                    SerialCallbackContext oldContext = curContext;
                    if (oldContext != null)
                        oldContext.check();
                    try {
                        curContext = new SerialCallbackContext(obj, slotDesc);

                        bin.setBlockDataMode(true);
                        slotDesc.invokeReadObject(obj, this); // 2. 如果有readObject，就执行
                    } catch (ClassNotFoundException ex) {
                        /*
                         * In most cases, the handle table has already
                         * propagated a CNFException to passHandle at this
                         * point; this mark call is included to address cases
                         * where the custom readObject method has cons'ed and
                         * thrown a new CNFException of its own.
                         */
                        handles.markException(passHandle, ex);
                    } finally {
                        do {
                            try {
                                curContext.setUsed();
                                if (oldContext!= null)
                                    oldContext.check();
                                curContext = oldContext;
                                reset = true;
                            } catch (ThreadDeath x) {
                                t = x;  // defer until reset is true
                            }
                        } while (!reset);
                        if (t != null)
                            throw t;
                    }

                    /*
                     * defaultDataEnd may have been set indirectly by custom
                     * readObject() method when calling defaultReadObject() or
                     * readFields(); clear it to restore normal read behavior.
                     */
                    defaultDataEnd = false;
                } else {
                    defaultReadFields(obj, slotDesc); // 3. 如果没有readObject，就执行ObjectInputStream默认的方法
                    }

                if (slotDesc.hasWriteObjectData()) {
                    skipCustomData();
                } else {
                    bin.setBlockDataMode(false);
                }
            } else {
                if (obj != null &&
                    slotDesc.hasReadObjectNoDataMethod() &&
                    handles.lookupException(passHandle) == null)
                {
                    slotDesc.invokeReadObjectNoData(obj);
                }
            }
        }
    }
```
> 1. 判断被序列化的类中是否包含readObject方法；

> 2. 如果包含，就执行被序列化类中的readObject方法；

> 3. 如果不包含， 执行ObjectInputStream默认的defaultReadFields方法。

### HashMap中重写的writeObject和readObject
#### HashMap序列化存在的问题
HashMap有必须重写它们的理由, 因为序列化会导致字节流在不同的jvm中传输, 而序列化基本要求就是反序列化后的对象与序列化之前的对象是一致的.

HashMap中，由于Entry的存放位置是根据Key的Hash值计算, 对于同一个Key，在不同的jvm中计算得出的Hash值可能是不同的.

Hash值不同导致HashMap对象反序列化的结果与序列化之前不一致. 有可能序列化之前Key=’name’的元素放在数组的第0个位置, 而反序列化后在数组第2个位置.

#### HashMap的解决方式
- 将可能造成数据不一致的元素使用transient修饰，然后重写writeObject/readObject方法, 在该方法中操作这些敏感元素, 避免默认序列化方法的干扰。被transient修饰的元素有: Entry[] table,size,modCount。
- 首先，HashMap序列化的时候会屏蔽掉负载因子, 只把不为空的key和value进行序列化. 传送到新的jvm反序列化时, 根据新的jvm处的规则重新对key进行hash算法, 重新填充一个数组. 这样避免了对象的不一致.
#### HashMap源码
```java
writeObject
    private void writeObject(java.io.ObjectOutputStream s)
        throws IOException {
        int buckets = capacity();
        // Write out the threshold, loadfactor, and any hidden stuff
        s.defaultWriteObject();
        s.writeInt(buckets);
        s.writeInt(size);
        internalWriteEntries(s); // 将有效的键值对进行了序列化.
    }
```

> 可以看到writeObject中序列化了buckets和size, 之后又通过internalWriteEntries方法将有效的键值对进行了序列化.

> 之后在readObject方法中反序列化时也要按照这个顺序.

- readObject
```java
    private void readObject(java.io.ObjectInputStream s) throws IOException, ClassNotFoundException {
        // Read in the threshold (ignored), loadfactor, and any hidden stuff
        s.defaultReadObject();
        reinitialize();
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new InvalidObjectException("Illegal load factor: " +
                                             loadFactor);
        s.readInt();                // Read and ignore number of buckets
        int mappings = s.readInt(); // Read number of mappings (size)
        if (mappings < 0)
            throw new InvalidObjectException("Illegal mappings count: " +
                                             mappings);
        else if (mappings > 0) { // (if zero, use defaults)
            // Size the table using given load factor only if within
            // range of 0.25...4.0
            float lf = Math.min(Math.max(0.25f, loadFactor), 4.0f);
            float fc = (float)mappings / lf + 1.0f;
            int cap = ((fc < DEFAULT_INITIAL_CAPACITY) ?
                       DEFAULT_INITIAL_CAPACITY :
                       (fc >= MAXIMUM_CAPACITY) ?
                       MAXIMUM_CAPACITY :
                       tableSizeFor((int)fc));
            float ft = (float)cap * lf;
            threshold = ((cap < MAXIMUM_CAPACITY && ft < MAXIMUM_CAPACITY) ?
                         (int)ft : Integer.MAX_VALUE);
            @SuppressWarnings({"rawtypes","unchecked"})
                Node<K,V>[] tab = (Node<K,V>[])new Node[cap];
            table = tab;

            // Read the keys and values, and put the mappings in the HashMap
            for (int i = 0; i < mappings; i++) {
                @SuppressWarnings("unchecked")
                    K key = (K) s.readObject();
                @SuppressWarnings("unchecked")
                    V value = (V) s.readObject();
                putVal(hash(key), key, value, false, false);
            }
        }
    }
```
> 按照顺序依次读出了:

> - 对象中非transient以及非static字段(defaultReadObject方法);
> - buckets, 但是被忽略了(见注释: Read and ignore number of buckets);
> - size(int mappings = s.readInt());
> - 根据size遍历读取键值对(for (int i = 0; i < mappings; i++)).
遍历读取的键值对最终依据新的规则保存到了新的map中, 如下:
```java
            for (int i = 0; i < mappings; i++) {
                @SuppressWarnings("unchecked")
                    K key = (K) s.readObject();
                @SuppressWarnings("unchecked")
                    V value = (V) s.readObject();
                putVal(hash(key), key, value, false, false);
            }
```
#### ArrayList重写writeObject和readObject
值得注意的是, ArrayList重写writeObject和readObject. 是因为在ArrayList中的数组容量基本上都会比实际的元素的数大, 为了避免序列化没有元素的数组而重写.

- writeObject
```java
    private void writeObject(java.io.ObjectOutputStream s)
        throws java.io.IOException{
        // Write out element count, and any hidden stuff
        int expectedModCount = modCount;
        s.defaultWriteObject();

        // Write out size as capacity for behavioural compatibility with clone()
        s.writeInt(size);

        // Write out all elements in the proper order.
        for (int i=0; i<size; i++) {
            s.writeObject(elementData[i]);
        }

        if (modCount != expectedModCount) {
            throw new ConcurrentModificationException();
        }
    }
```
- readObject
```java
    private void readObject(java.io.ObjectInputStream s) throws java.io.IOException, ClassNotFoundException {
        elementData = EMPTY_ELEMENTDATA;

        // Read in size, and any hidden stuff
        s.defaultReadObject();

        // Read in capacity
        s.readInt(); // ignored

        if (size > 0) {
            // be like clone(), allocate array based upon size not capacity
            ensureCapacityInternal(size);

            Object[] a = elementData;
            // Read in all elements in the proper order.
            for (int i=0; i<size; i++) {
                a[i] = s.readObject();
            }
        }
    }
```
