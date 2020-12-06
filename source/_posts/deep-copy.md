---
layout: post
title: 深拷贝与浅拷贝
date: 2020-11-30 10:00:00
tags: 
- 基础
categories:
- 基础
---

转自 https://zhuanlan.zhihu.com/p/95686213  有改动

## 创建对象的5种方式

- 通过 new 关键字
  这是最常用的一种方式，通过 new 关键字调用类的有参或无参构造方法来创建对象。比如 Object obj = new Object();
- 通过 Class 类的 newInstance() 方法
  这种默认是调用类的无参构造方法创建对象。比如 Person p2 = (Person) Class.forName("com.ys.test.Person").newInstance();
- 通过 Constructor 类的 newInstance 方法
  这和第二种方法类时，都是通过反射来实现。通过 java.lang.relect.Constructor 类的 newInstance() 方法指定某个构造器来创建对象。Person p3 = (Person) Person.class.getConstructors()[0].newInstance();
  实际上第二种方法利用 Class 的 newInstance() 方法创建对象，其内部调用还是 Constructor 的 newInstance() 方法。
- 利用 Clone 方法
  Clone 是 Object 类中的一个方法，通过 对象A.clone() 方法会创建一个内容和对象 A 一模一样的对象 B，clone 克隆，顾名思义就是创建一个一模一样的对象出来。
  Person p4 = (Person) p3.clone();
- 反序列化
  序列化是把堆内存中的 Java 对象数据，通过某种方式把对象存储到磁盘文件中或者传递给其他网络节点（在网络上传输）。而反序列化则是把磁盘文件中的对象数据或者把网络节点上的对象数据，恢复成Java对象模型的过程。

## 直接赋值

直接赋值是我们最常用的方式，在我们代码中的体现是Persona = new Person();Person b = a，是一种简单明了的方式，但是它只是拷贝了对象引用地址而已，并没有在内存中生成新的对象，我们可以通过下面这个例子来证明这一点

```java
/ person 对象
public class Person {
    // 姓名
    private String name;
    // 年龄
    private int age;
    // 邮件
    private String email;
    // 描述
    private String desc;
    ...省略get/set...
 }
 // main 方法
public class PersonApp {
   public static void main(String[] args) {
       // 初始化一个对象
       Person person = new Person("张三",20,"123456@qq.com","我是张三");
       // 复制对象
       Person person1 = person;
       // 改变 person1 的属性值
       person1.setName("我不是张三了");
        System.out.println("person对象："+person);
        System.out.println("person1对象："+person1);

   }
}
```

运行上面代码，你会得到如下结果：

```
person对象：Person{name='我不是张三了', age=20, email='123456@qq.com', desc='我是张三'}
person1对象：Person{name='我不是张三了', age=20, email='123456@qq.com', desc='我是张三'}
```

我们将 person 对象复制给了 person1 对象，我们对 person1 对象的 name 属性进行了修改，并未修改 person 对象的name 属性值，但是我们最后发现 person 对象的 name 属性也发生了变化，其实不止这一个值，对于其他值也是一样的，所以这结果证明了我们上面的结论：直接赋值的方式没有生产新的对象，只是生新增了一个对象引用，直接赋值在 Java 内存中的模型大概是这样的

![image.png](https://i.loli.net/2020/12/02/FKznv7wxp6jWsuQ.png)

## 浅拷贝

浅拷贝也可以实现对象克隆，从这名字你或许可以知道，这种拷贝一定存在某种缺陷，是的，它就是存在一定的缺陷，先来看看浅拷贝的定义：

如果原型对象的成员变量是 值类型，将复制一份给克隆对象，也就是说在堆中拥有独立的空间；如果原型对象的成员变量是 引用类型，则将引用对象的地址复制一份给克隆对象，也就是说原型对象和克隆对象的成员变量指向相同的内存地址。换句话说，在浅克隆中，当对象被复制时，只复制它本身和其中包含的值类型的成员变量，而引用类型的成员对象并没有复制。 可能你没太理解这段话，那么我们在来看看浅拷贝的通用模型：

![image.png](https://i.loli.net/2020/12/02/A4I9zlNayhdLxg3.png)

要实现对象浅拷贝还是比较简单的，只需要被复制类需要实现 Cloneable 接口，重写 clone 方法即可，对 person 类进行改造，使其可以支持浅拷贝。

```java
public class Person implements Cloneable {
    // 姓名
    private String name;
    // 年龄
    private int age;
    // 邮件
    private String email;
    // 描述
    private String desc;

    /*
    * 重写 clone 方法，需要将权限改成 public ，直接调用父类的 clone 方法就好了
    */
    @Override
    public Object clone() throws CloneNotSupportedException {
        return super.clone();
    }
    ...省略...
}
```

改造很简单只需要让 person 继承 Cloneable 接口，并且重写 clone 方法即可，clone 也非常简单只需要调用 object 的 clone 方法就好，唯一需要注意的地方就是 clone 方法需要用 public 来修饰，在简单的修改 main 方法

```java
public class PersonApp {
    public static void main(String[] args) throws Exception {
        // 初始化一个对象
        Person person = new Person("张三",20,"123456@qq.com","我是张三");
        // 复制对象
        Person person1 = (Person) person.clone();
        // 改变 person1 的属性值
        person1.setName("我是张三的克隆对象");
        // 修改 person age 的值
        person1.setAge(22);
        System.out.println("person对象："+person);
        System.out.println();
        System.out.println("person1对象："+person1);

    }
}
```

重新运行 main 方法，结果如下：

```
person对象：Person{name='张三', age=20, email='123456@qq.com', desc='我是张三'}

person1对象：Person{name='我是张三的克隆对象', age=22, email='123456@qq.com', desc='我是张三'}
```

看到这个结果，你是否有所质疑呢？说好的引用对象只是拷贝了地址，为啥修改了 person1 对象的 name 属性值，person 对象没有改变？这里就是一个非常重要的知识点了，原因在于：

String、Integer 等包装类都是不可变的对象，当需要修改不可变对象的值时，需要在内存中生成一个新的对象来存放新的值，然后将原来的引用指向新的地址，所以在这里我们修改了 person1 对象的 name 属性值，person1 对象的 name 字段指向了内存中新的 name 对象，但是我们并没有改变 person 对象的 name 字段的指向，所以 person 对象的 name 还是指向内存中原来的 name 地址，也就没有变化

这种引用是一种特列，因为这些引用具有不可变性，并不具备通用性，所以我们就自定义一个类，来演示浅拷贝，我们定义一个 PersonDesc 类用来存放person 对象中的 desc 字段，，然后在 person 对象中引用 PersonDesc 类，具体代码如下：

```java
// 新增 PersonDesc 
public class PersonDesc {
    // 描述
    private String desc;

}
public class Person implements Cloneable {
    // 姓名
    private String name;
    // 年龄
    private int age;
    // 邮件
    private String email;
    // 将原来的 string desc 变成了 PersonDesc 对象，这样 personDesc 就是引用类型
    private PersonDesc personDesc;

    @Override
    public Object clone() throws CloneNotSupportedException {
        return super.clone();
    }
    public void setDesc(String desc) {
        this.personDesc.setDesc(desc);
    }
    public Person(String name, int age, String email, String desc) {
        this.name = name;
        this.age = age;
        this.email = email;
        this.personDesc = new PersonDesc();
        this.personDesc.setDesc(desc);
    }
     ...省略...
}
```

修改 main 方法

```java
public class PersonApp {
    public static void main(String[] args) throws Exception {
        // 初始化一个对象
        Person person = new Person("平头",20,"123456@qq.com","技术");
        // 复制对象
        Person person1 = (Person) person.clone();
        // 改变 person1 的属性值
        person1.setName("我是平头的克隆对象");
        // 修改 person age 的值
        person1.setAge(22);
        person1.setDesc("我已经关注了技术");
        System.out.println("person对象："+person);
        System.out.println();
        System.out.println("person1对象："+person1);
    }
}
```

运行 main 方法，得到如下结果：

```
person对象：Person{name='平头', age=20, email='123456@qq.com', desc='我已经关注了技术'}

person1对象：Person{name='我是平头的克隆对象', age=22, email='123456@qq.com', desc='我已经关注了技术'}
```

我们修改 person1 的 desc 字段之后，person 的 desc 也发生了改变，这说明 person 对象和 person1 对象指向是同一个 PersonDesc 对象地址，这也符合浅拷贝引用对象只拷贝引用地址并未创建新对象的定义，到这你应该知道浅拷贝了吧。

## 深拷贝

深拷贝也是对象克隆的一种方式，相对于浅拷贝，深拷贝是一种完全拷贝，无论是值类型还是引用类型都会完完全全的拷贝一份，在内存中生成一个新的对象，简单点说就是拷贝对象和被拷贝对象没有任何关系，互不影响。深拷贝的通用模型如下：

![image.png](https://i.loli.net/2020/12/02/nIiLf3POJyYdroh.png)

深拷贝有两种方式，一种是跟浅拷贝一样实现 Cloneable 接口，另一种是实现 Serializable 接口，用序列化的方式来实现深拷贝，我们分别用这两种方式来实现深拷贝

### 实现 Cloneable 接口方式

实现 Cloneable 接口的方式跟浅拷贝相差不大，我们需要引用对象也实现 Cloneable 接口，具体代码改造如下：

```java
public class PersonDesc implements Cloneable{

    // 描述
    private String desc;
    ...省略...
    @Override
    public Object clone() throws CloneNotSupportedException {
        return super.clone();
    }
}
public class Person implements Cloneable {
    // 姓名
    private String name;
    // 年龄
    private int age;
    // 邮件
    private String email;

    private PersonDesc personDesc;

    /**
    * clone 方法不是简单的调用super的clone 就好，
    */
    @Override
    public Object clone() throws CloneNotSupportedException {
        Person person = (Person)super.clone();
        // 需要将引用对象也克隆一次
        person.personDesc = (PersonDesc) personDesc.clone();
        return person;
    }
    ...省略...
}
```

main 方法不需要任何改动，我们再次运行 main 方法，得到如下结果：

```
person对象：Person{name='平头', age=20, email='123456@qq.com', desc='技术'}

person1对象：Person{name='平头的克隆对象', age=22, email='123456@qq.com', desc='我已经关注了技术'}
```

可以看出，修改 person1 的 desc 时对 person 的 desc 已经没有影响了，说明进行了深拷贝，在内存中重新生成了一个新的对象。

### 实现 Serializable 接口方式

实现 Serializable 接口方式也可以实现深拷贝，而且这种方式还可以解决多层克隆的问题，多层克隆就是引用类型里面又有引用类型，层层嵌套下去，用 Cloneable 方式实现还是比较麻烦的，一不小心写错了就不能实现深拷贝了，使用 Serializable 序列化的方式就需要所有的对象对实现 Serializable 接口，我们对代码进行改造，改造成序列化的方式

```java
public class Person implements Serializable {

    private static final long serialVersionUID = 369285298572941L;
    // 姓名
    private String name;
    // 年龄
    private int age;
    // 邮件
    private String email;

    private PersonDesc personDesc;

    public Person clone() {
        Person person = null;
        try { 
            // 将该对象序列化成流, 因为写在流里的是对象的一个拷贝，而原对象仍然存在于JVM里面。所以利用这个特性可以实现对象的深拷贝
            ByteArrayOutputStream baos = new ByteArrayOutputStream();
            ObjectOutputStream oos = new ObjectOutputStream(baos);
            oos.writeObject(this);
            // 将流序列化成对象
            ByteArrayInputStream bais = new ByteArrayInputStream(baos.toByteArray());
            ObjectInputStream ois = new ObjectInputStream(bais);
            person = (Person) ois.readObject();
        } catch (IOException e) {
            e.printStackTrace();
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }
        return person;
    }

    public void setDesc(String desc) {
        this.personDesc.setDesc(desc);
    }
  ...省略...
}
public class PersonDesc implements Serializable {

    private static final long serialVersionUID = 872390113109L; 
    // 描述
    private String desc;

    public String getDesc() {
        return desc;
    }

    public void setDesc(String desc) {
        this.desc = desc;
    }

}
public class PersonApp {
    public static void main(String[] args) throws Exception {
        // 初始化一个对象
        Person person = new Person("平头",20,"123456@qq.com","技术");
        // 复制对象
        Person person1 = (Person) person.clone();
        // 改变 person1 的属性值
        person1.setName("我是平头的克隆对象");
        // 修改 person age 的值
        person1.setAge(22);
        person1.setDesc("我已经关注了技术");
        System.out.println("person对象："+person);
        System.out.println();
        System.out.println("person1对象："+person1);
    }
}

```

运行 main 方法，我们可以得到跟 Cloneable 方式一样的结果，序列化的方式也实现了深拷贝。