### 接口信息
- 实现接口RandomAccess，表示支持快速随机访问
- 实现Serializable标记接口， 重写了readObject和writeObject方法，其底层的elementData数组是transient修饰
- 实现Cloneable标记接口，ArrayList实现的是浅拷贝

### 属性
- 底层数据结构 transient Object\[] elementData
- size：size是数组数据的数量，ArrayList的大小和size不是一个概念

### 初始化
- 指定初始化容量：如果用户指定，= 0 则使用空数组（添加元素会扩容）， > 0 创建指容量数组, < 0 报错；
- 不指定初始化容量：默认的容量为10，数组赋初始值为 {} 空数组
  ```java
  public ArrayList() {
        this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
    }
  ```

### 添加元素
- 添加元素：首先如果容量不够触发扩容，然后将元素放到数组的末尾
- 指定位置添加元素：首先如果容量不够触发扩容，将 index + 1 位置开始的元素，进行往后挪，然后在指定位置插入数据

### 扩容
grow() 方法，比较简单
触发扩容的条件是：
```java
if ((s = size) == (elementData = this.elementData).length)
        elementData = grow();
```
扩容过程为：如果不是空数组，则 1.5 被扩容；如果是空数组，则按照默认的初始化容量10扩容

缩容：trimToSize(), 新建size大小的数组，将old数组的数据copy到new数组

ensureCapacity(int minCapacity) 方法，保证 elementData 数组容量至少有 minCapacity

### 插入多个元素
addAll(Collection<? extends E> c)
addAll(int index, Collection<? extends E> c)
也比较简单，一次性扩容到可以容纳所有数据的容量，可以避免多次扩容

### 移除单个元素
remove(int index)
remove()
remove(Object o)
如果 i 不是移除最末尾的元素，则将 i + 1 位置的数组往前挪，也比较简单

### 移除多个元素
removeRange(int fromIndex, int toIndex)
逻辑比较简单

### 查找单个元素
int indexOf(Object o) 
从前往后遍历，找到了返回index，否则返回 -1

contains方法就是基于indexOf ——>  return indexOf(o) >= 0;

lastIndexOf(Object o) 找到元素最后出现的位置

### 获得指定位置的元素
遍历，简单

### 设置指定位置的元素
遍历替换，简单

### 转换成数组
toArray()
toArray(T[] a)
底层基于 System.arraycopy 返回 拷贝的副本，浅拷贝

### 求hash值
```java
// 遍历每个元素，* 31 求哈希。
int hashCode = 1;
for (int i = from; i < to; i++) {
    Object e = es[i];
    hashCode = 31 * hashCode + (e == null ? 0 : e.hashCode());
}
```

### 判断相等
相等的条件是  类型相同、元素相同、元素个数相同

### 清空数组
clear()
数组长度不变，遍历数组，倒序置为null

### 序列化数组
也比较简单
```java
// <1> 写入非静态属性、非 transient 属性
s.defaultWriteObject();

// Write out size as capacity for behavioral compatibility with clone()
// <2> 写入 size ，主要为了与 clone 方法的兼容
s.writeInt(size);

// Write out all elements in the proper order.
// <3> 逐个写入 elementData 数组的元素
for (int i = 0; i < size; i++) {
    s.writeObject(elementData[i]);
}
```

### 反序列化数组
和序列化的代码相反

### 克隆
```java
public Object clone() {
    try {
        // 调用父类，进行克隆
        ArrayList<?> v = (ArrayList<?>) super.clone();
        // 拷贝一个新的数组
        v.elementData = Arrays.copyOf(elementData, size);
        // 设置数组修改次数为 0
        v.modCount = 0;
        return v;
    } catch (CloneNotSupportedException e) {
        // this shouldn't happen, since we are Cloneable
        throw new InternalError(e);
    }
}
```

### 创建子数组
subList(int fromIndex, int toIndex)
这个方法有坑， SubList 不是一个只读数组，而是和根数组 root 共享相同的 elementData 数组，只是说限制了 [fromIndex, toIndex) 的范围。

### 迭代器
看一下就好了










