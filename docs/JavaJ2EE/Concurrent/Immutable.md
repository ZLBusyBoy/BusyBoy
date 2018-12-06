@[toc]
# Immutable

&nbsp;&nbsp;&nbsp;&nbsp;`Immutable`是什么意思？不变的、不发生改变的意思。在`JDK`中有很多的类被设计成不可变的，举个大家经常用到的类`java.lang.String`，`String`类被设计成不可变。`String`所表示的字符串的内容绝对不会发生变化。因此，在多线程的情况下，`String`类无需进行互斥处理，不用给方法进行`synchronized`或者`lock`等操作，进行上锁、争抢锁、解锁等流程也是有一定性能损耗的。因此，若能合理的利用`Immutable`，一定对性能的提升有很大帮助。

## 为什么String不可变？
&nbsp;&nbsp;&nbsp;&nbsp;那么为什么`String`是不可变的呢？满足那些条件才可以变成不可变的类？大家可以打开`String`类的源码：

```java
public final class String
    implements java.io.Serializable, Comparable<String>, CharSequence {
    /** The value is used for character storage. */
    private final char value[];

    /** Cache the hash code for the string */
    private int hash; // Default to 0

    /** use serialVersionUID from JDK 1.0.2 for interoperability */
    private static final long serialVersionUID = -6849794470754667710L;

    /**
     * Class String is special cased within the Serialization Stream Protocol.
     *
     * A String instance is written into an ObjectOutputStream according to
     * <a href="{@docRoot}/../platform/serialization/spec/output.html">
     * Object Serialization Specification, Section 6.2, "Stream Elements"</a>
     */
    private static final ObjectStreamField[] serialPersistentFields =
        new ObjectStreamField[0];
    /**
     * Allocates a new {@code String} so that it represents the sequence of
     * characters currently contained in the character array argument. The
     * contents of the character array are copied; subsequent modification of
     * the character array does not affect the newly created string.
     *
     * @param  value
     *         The initial value of the string
     */
    public String(char value[]) {
        this.value = Arrays.copyOf(value, value.length);
    }
    /**
     * Converts this string to a new character array.
     *
     * @return  a newly allocated character array whose length is the length
     *          of this string and whose contents are initialized to contain
     *          the character sequence represented by this string.
     */
    public char[] toCharArray() {
        // Cannot use Arrays.copyOf because of class initialization order issues
        char result[] = new char[value.length];
        System.arraycopy(value, 0, result, 0, value.length);
        return result;
    }
  }
```
通过上边的这段代码你会发现：

> 1、首先`String`类本身是被`final`修饰过的，表明该类无法进行扩展，无法创建子类。因此类中声明的方法则不会被重写。
> 2、`String`类中的成员变量都被`final`修饰，同时均为`private`，被`final`修饰则表示成员变量
不会被`setter`方法再次赋值，`private`则表示成员变量均为类私有，外部无法直接调用。
> 3、`String`类中的成员变量都没有`setter`方法，避免其他接口调用改变成员变量的值。
> 4、通过构造器初始化所有成员，同时在`String`中赋值是用的`Arrays.copyOf`等深拷贝方法。
> 5、在`getter`方法中，不要直接返回对象本身，而是克隆对象，并返回对象的拷贝。

以上5点保证`String`类的不可变性（`immutability`）。


# 示例程序
&nbsp;&nbsp;&nbsp;&nbsp;下面咱们通过一些简单的示例程序来实验`Immutable`，自己动手，丰衣足食，多动手会有好处的。下边定义三个类。

|类名|说明|
|----|----|
|`People`|表示一个人的类|
|`PeopleThread`|表示`People`实例的线程的类|
|`Main`|测试程序行为的类|
下边看下每个类的示例代码，`People`类，类以及成员变量均被`final`修饰，同时只能通过构造函数来对成员变量赋值，没有`setter`方法：

```java
public final class People {
    private final String sex;
    private final int age;
    private final String address;

    public People(String sex, int age, String address) {
        this.sex = sex;
        this.age = age;
        this.address = address;
    }

    public String getSex() {
        return sex;
    }

    public int getAge() {
        return age;
    }

    public String getAddress() {
        return address;
    }

    @Override
    public String toString() {
        return "People{" +
                "sex='" + sex + '\'' +
                ", age=" + age +
                ", address='" + address + '\'' +
                '}';
    }
}
```

`PeopleThread`类：

```java
public class PeopleThread extends Thread{

    private People people;

    public PeopleThread(People people){
        this.people = people;
    }

    @Override
    public void run() {
        while (true){
            System.out.println(Thread.currentThread().getName() + " prints " + people);
        }
    }
}

```
`Main`类：

```java
public class Main {

    public static void main(String[] args) {
        People people = new People("男",27,"北京");
        new PeopleThread(people).start();
        new PeopleThread(people).start();
        new PeopleThread(people).start();
    }
}
```

下边是执行结果：

```java
Thread-0 prints People{sex='男', age=27, address='北京'}
Thread-0 prints People{sex='男', age=27, address='北京'}
Thread-0 prints People{sex='男', age=27, address='北京'}
Thread-0 prints People{sex='男', age=27, address='北京'}
Thread-2 prints People{sex='男', age=27, address='北京'}
Thread-2 prints People{sex='男', age=27, address='北京'}
Thread-2 prints People{sex='男', age=27, address='北京'}
Thread-2 prints People{sex='男', age=27, address='北京'}
Thread-1 prints People{sex='男', age=27, address='北京'}
Thread-1 prints People{sex='男', age=27, address='北京'}
Thread-1 prints People{sex='男', age=27, address='北京'}
Thread-1 prints People{sex='男', age=27, address='北京'}
```
通过结果就可以发现，无论启动多少个线程，打印的结果其实都是一样的。因为`People`类本身在**实例被创建且字段初始化之后，字段的值就不会再被修改**，实例的状态在初始化之后就不会再发生改变，因此也不需要在进行加锁、解锁等操作。因为想破坏也破坏不了。但是有一点比较难的就是如果确保`Immutability`，因为在对类的创建过程中少个`final`，多个`setter`等，那么就无法保证类的`Immutability`。

# 何时使用呢？
上边简单的讲解了下`Immutable`模式，那么在那些情况下考虑使用`Immutability`不可变性呢？
## 实例创建后，状态不再发生变化时
这个可以参考上边的示例，一个人的属性被赋值之后就不会发生改变。这种情况下就可以考虑不可变性来实现。
## 实例是共享的，且被频繁访问时
`Immutable`模式的优点是“**不需要使用synchronized来保护**”，这就意味着能够在不是去安全性和生存性的前提下提高性能。当实例被多个线程共享，且有可能被频繁访问时，`Immutable`模式的优点就会凸显出来。关于不适用`synchronized`能提高多少性能？下边做个实现：

```java
public class NoSynchronized {

    private static final long CALL_COUNT = 1000000000L;

    public static void main(String[] args) {
        trial("NotSynch",CALL_COUNT,new NotSynch());
        trial("Synch",CALL_COUNT,new Synch());
    }

    private static void trial(String msg,long count,Object object){
        System.out.println(msg + ": BEGIN");
        long start_time = System.currentTimeMillis();
         for (long i = 0; i< count; i++){
             object.toString();
         }

         System.out.println(msg + ": END");
         System.out.println("Elapsed time = " + (System.currentTimeMillis() - start_time) + "msec");
    }

    static class NotSynch{
        private final String name = "NotSynch";

        @Override
        public String toString() {
            return "NotSynch{" +
                    "name='" + name + '\'' +
                    '}';
        }
    }

    static class Synch{
        private final String name = "Synch";

        @Override
        public synchronized String toString() {
            return "Synch{" +
                    "name='" + name + '\'' +
                    '}';
        }
    }
}

```

查看执行结果，差了44倍，这是在完全没有发生线程冲突的情况下测试的，所以测试的时间就是获取和释放实例锁所花费的时间。当然也跟本地的环境也会对时间差有一定的影响，因此仅供参考：

```java
NotSynch: BEGIN
NotSynch: END
Elapsed time = 693msec

Synch: BEGIN
Synch: END
Elapsed time = 31162msec
```

# 哪些情况会破坏不可变性？
-  `getter`返回的不是不可变的类
举个例子，比如变量为`StringBuffer`类型：

```java
public final class UserDetail {
    private final StringBuffer detail;
    public UserDetail(String sex,int age,String address){
        this.detail = new StringBuffer(sex + ":" + age + ":" + address);
    }

    public StringBuffer getDetail() {
        return detail;
    }

    @Override
    public String toString() {
        return "UserDetail{" +
                "detail=" + detail +
                '}';
    }


    public static void main(String[] args) {
        UserDetail userDetail = new UserDetail("男",27,"北京");
        //显示
        System.out.println(userDetail);

        StringBuffer detail = userDetail.getDetail();
        detail.append(":::").append("test");
        //再次显示
        System.out.println(userDetail);
    }
}

运行结果：
UserDetail{detail=男:27:北京}
UserDetail{detail=男:27:北京:::test}
```


在`get`结果之后重新进行`append`操作，`StringBuffer`包含修改内部状态的方法，所以`detail`字段的内容也是可以被外部修改的。

- 在一个类中使用了其他的类，其他的类是可变的
当一个不可变的类中使用了其他的可变类之后，那么受影响不可变的类也会变成可变的类。

# 扩展
在`Java`的标准类库中，有些类也用到了`Immutable`模式
- `java.lang.String`
- `java.math.BigInteger` && `java.math.BigDecimal`
- `java.util.regex.Pattern`
- `java.lang.Integer` && `java.lang.Short`等基本数据类型包装类

在一开始的时候说到了`String`是不可变的，那么是否真的不可变呢？有一种方式是可以更改其状态的，反射机制。

```java
//创建字符串"Hello World"， 并赋给引用s
    String s = "Hello World"; 
    System.out.println("s = " + s); //Hello World

    //获取String类中的value字段
    Field valueFieldOfString = String.class.getDeclaredField("value");
    //改变value属性的访问权限
    valueFieldOfString.setAccessible(true);

    //获取s对象上的value属性的值
    char[] value = (char[]) valueFieldOfString.get(s);
    //改变value所引用的数组中的第5个字符
    value[5] = '#';
    System.out.println("s = " + s);  //Hello#World

运行结果
s = Hello World
s = Hello#World
```

发现String的值已经发生了改变。也就是说，通过反射是可以修改所谓的“不可变”对象的。这些在使用的时候都是需要注意的地方。
