### Cloneable接口的官方解释
1.Cloneable属于标记接口，接口内部没有方法和属性。实现该接口的类的实例可以被克隆，能够使用clone()方法。
2.如果在没有实现Cloneable接口的实例上调用Object的clone()方法，则会抛出CloneNotSupportException异常。
3.按照惯例，实现Cloneable接口的类应该重写Object.clone()方法。注意，此接口不包含clone()方法，

### Cloneable接口的使用
#### 不实现Cloneable接口直接调用clone()方法
首先定义一个Teacher类：
```java
public class Teacher
{
    private String name;
    
    private String subject;
    
    public Teacher(String name, String subject)
    {
        super();
        this.name = name;
        this.subject = subject;
    }

    public String getName()
    {
        return name;
    }

    public void setName(String name)
    {
        this.name = name;
    }

    public String getSubject()
    {
        return subject;
    }

    public void setSubject(String subject)
    {
        this.subject = subject;
    }
}
```
然后定义一个Clone测试类：
```
public class CloneTest
{

    public static void main(String[] args)
    {
        Teacher teacher = new Teacher("Mr Wang", "Data Structure");
        teacher.clone();
    }

}
```
编译的时候就报错：The method clone() from the type Object is not visible
这是因为Object的clone()方法定义如下：

protected native Object clone() throws CloneNotSupportedException;
clone()的访问类型为protected，限制本类、本包和子类可以访问，所以Teacher类不能访问clone()方法。

#### 实现Cloneable接口
先学习一下两个概念
浅拷贝：浅拷贝是指拷贝对象时只拷贝对象本身和对象中的基本变量，而对象中引用类型的变量并不会拷贝，拷贝后的对象和原对象指向同一个引用对象，对引用对象的修改会互相影响。Object对象的clone()属于浅拷贝。
深拷贝：深拷贝会连同对象的引用类型变量一起拷贝，这样原对象和拷贝得到对象任意修改对象的属性值而不会相互影响。实现深拷贝的话需要重写clone()方法。
##### 浅拷贝举例：
定义一个Student类实现Cloneable接口：
```java
public class Student implements Cloneable
{
    private String name;
    
    private int number;
    
    private Teacher teacher;

    public Student(String name, int number, Teacher teacher)
    {
        super();
        this.name = name;
        this.number = number;
        this.teacher = teacher;
    }

    public String getName()
    {
        return name;
    }

    public void setName(String name)
    {
        this.name = name;
    }

    public int getNumber()
    {
        return number;
    }

    public void setNumber(int number)
    {
        this.number = number;
    }

    public Teacher getTeacher()
    {
        return teacher;
    }

    public void setTeacher(Teacher teacher)
    {
        this.teacher = teacher;
    }
    
    public Object clone() throws CloneNotSupportedException
    {
        return super.clone();
    }
}   
```
测试类：
```java
public class CloneTest
{

    public static void main(String[] args) throws CloneNotSupportedException
    {
        Teacher teacher = new Teacher("Mr Wang", "Data Structure");
        Student stu1 = new Student("victor", 1, teacher);
        Student stu2 = (Student) stu1.clone();
        System.out.println("Name: " + stu2.getName());
        System.out.println("Number: " + stu2.getNumber());
        System.out.println("Teacher's name: " + stu2.getTeacher().getName());
        teacher.setName("Mr Li");
        System.out.println("After change, teacher's name: " + stu2.getTeacher().getName());
    }

}
```
执行结果如下，可见修改原对象的teacher属性的名称，拷贝得到的对象中teacher对象的名称也变更了。
```
Name: victor
Number: 1
Teacher's name: Mr Wang
After change, teacher's name: Mr Li
```

##### 深拷贝举例

如下，重写Student类的clone()方法：
```java
public Object clone() throws CloneNotSupportedException
{
    Student stu = (Student) super.clone();
    stu.teacher = (Teacher) stu.getTeacher().clone();
    return stu;
}
```
执行结果如下，可见修改原对象的teacher属性的名称，并不会影响拷贝得到的对象中teacher对象的名称。

```
Name: victor
Number: 1
Teacher's name: Mr Wang
After change, teacher's name: Mr Wang
```
