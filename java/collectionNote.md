# 容器
容器可以分为两个大类： Collection和Map
### Map的类关系为：
![image](https://github.com/wangjunjie0817/note/blob/master/images/map.png)

#### HashMap
初始化
扩容
hash
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
