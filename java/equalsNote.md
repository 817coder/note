# HashCode()和equals()
#### equals() 和 == 有什么区别呢？
首先从 equals() 和 == 开始
- equals(): 我们知道equals()是Object.class中定义的方法，用来判断两个对象是否相等。在java中，很多类例如String、ArrayList....重写了equals()，如下是String中的equals方法
```java
    public boolean equals(Object anObject) {
        if (this == anObject) {
            return true;
        }
        if (anObject instanceof String) {
            String anotherString = (String)anObject;
            int n = value.length;
            if (n == anotherString.value.length) {
                char v1[] = value;
                char v2[] = anotherString.value;
                int i = 0;
                while (n-- != 0) {
                    if (v1[i] != v2[i])
                        return false;
                    i++;
                }
                return true;
            }
        }
        return false;
    }
```
- 那 == 呢？ == 实际上是比较两个对象的内存地址，如果两个对象的指针相同，那么 == 的结果就是true。

#### 重写 equals()
##### 如何重写equals()方法？

我们先定义一个类(Person)：
```java
class Person {
	private String name;
	private int age;
	
	// getters and setters
}
```

重写时要遵循以下三步：

```
- 1. 判断是否等于自身.
```java
if(other == this)
return true;
```

- 2. 使用 instanceof 运算符判断 other 是否为Person类型的对象.
```java
if(!(other instanceof Person))
return false;
```
            
- 3. 比较Person类中你自定义的数据域，name和age，一个都不能少.
```java
Person o = (Person)other;
return o.name.equals(name) && o.age == age;
```
上面的三步也是＜Effective Java＞中推荐的步骤

**warn!!!** 答案是不行，因为我们既然重写了 equals() 方法，就要将hashCode()方法也重写！！！

#### 为什么hashCode()方法也重写呢？

重写equals()而不重写hashCode()的风险

我们知道在 JDK1.8 中的HashMap 是采用 Node数组 + Node 链表（也存在红黑树）实现，每个Node用来存储对象。Java通过hashCode()方法来确定某个对象应该位于数组的那个位置，然后在相应的位置的链表中进行查找．在理想情况下，如果你的hashCode()方法写的足够健壮，那么每个数组下标的位置将会只有一个结点，这样就实现了查找操作的常量级的时间复杂度．即无论你的对象放在哪片内存中，我都可以通过hashCode()立刻定位到该区域，而不需要从头到尾进行遍历查找．这也是哈希表的最主要的应用．

如：
当我们调用HashMap的put(Object o)方法时，首先会根据o.hashCode()的返回值定位到相应数组下标，如果该数组下标位置没有结点，则将 o 放到这里，如果已经有结点了, 则把 o 挂到当前下标链表末端．同理，当调用contains(Object o)时，Java会通过hashCode()的返回值定位到相应的下标位置，然后再在对应的链表中的结点依次调用equals()方法来判断结点中的对象是否是你想要的对象．

下面我们通过一个例子来体会一下这个过程：

我们先创建２个新的Person对象：
```
Person c1 = new Person("wang", 25);
Person c2 = new Person("wang", 25);
```

现在假定我们已经重写了Person的equals()方法而没有重写hashCode()方法：
```
    @Override
	public boolean equals(Object other) {
		System.out.println("equals method invoked!");
		
		if(other == this)
			return true;
		if(!(other instanceof Person))
			return false;
		
		Person o = (Person)other;
		return o.name.equals(name) && o.age == age;
	}
```

然后我们构造一个HashSet，将c1对象放入到set中：
```java
Set<Person> set = new HashSet<Person>();
set.add(c1);
```

再执行：
```java
System.out.println(set.contains(c2));
```

我们期望contains(c2)方法返回true, 但实际上它返回了false.

c1和c2的name和age都是相同的，为什么我把c1放到HashSet中后，再调用contains(c2)却返回false呢？这就是hashCode()在作怪了．因为你没有重写hashCode()方法，所以HashSet在查找c2时，会在不同的bucket中查找．比如c1放到05这个数组下标位置，在查找c2时却在06这个下标位置找，这样当然找不到了．因此，我们重写hashCode()的目的在于，在A.equals(B)返回true的情况下，A, B 的hashCode()要返回相同的值．

那么让hashCode()每次都返回一个固定的数行吗
有人可能会这样重写：
```java
@Override
	public int hashCode() {
		return 10;
 
	}
```

如果这样的话，HashMap, HashSet等集合类就失去了其 "哈希的意义"．用<Effective Java>中的话来说就是，哈希表退化成了链表．如果hashCode()每次都返回相同的数，那么所有的对象都会被放到同一个bucket中，每次执行查找操作都会遍历链表，这样就完全失去了哈希的作用．所以我们最好还是提供一个健壮的hashCode()为妙．
    
#### 如何重写hashCode()方法

综上，如果你重写了equals()方法，那么一定要记得重写hashCode()方法．

如果是写一个中小型的应用，那么下面的原则就已经足够使用了，重写 hashCode() 时，要保证对象中所有的成员都能在hashCode中得到体现．

对于Person类，我们可以这么写：
```
    @Override
	public int hashCode() {
		int result = 17;
        result = result * 31 + (name == null? 0 : name.hashCode());
        result = result * 31 + age;
        return result;
	}
```

其中int result = 17你也可以改成20, 50等等都可以．

再看一下String类中的hashCode()方法是如何实现的．查文档知：

"Returns a hash code for this string. The hash code for a String object is computed as
 s[0]*31^(n-1) + s[1]*31^(n-2) + ... + s[n-1]
 
using  int  arithmetic, where  s[i]  is the  i th character of the string,  n  is the length of the string, and  ^  indicates exponentiation. (The hash value of the empty string is zero.)"

对每个字符的ASCII码计算n - 1次方然后再进行加和，可见Sun对hashCode的实现是很严谨的. 这样能最大程度避免２个不同的String会出现相同的hashCode的情况．
