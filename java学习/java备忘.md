Java编译后出现$1.class、$2.class等多个class文件，如果java文件里存在匿名类会编译出多个class文件。

java assert关键字

1、assert condition;
    这里condition是一个必须为真(true)的表达式。如果表达式的结果为true，那么断言为真，并且无任何行动
如果表达式为false，则断言失败，则会抛出一个AssertionError对象。这个AssertionError继承于Error对象，
而Error继承于Throwable，Error是和Exception并列的一个错误对象，通常用于表达系统级运行错误。
2、asser condition:expr;
    这里condition是和上面一样的，这个冒号后跟的是一个表达式，通常用于断言失败后的提示信息，说白了，它是一个传到AssertionError构造函数的值，如果断言失败，该值被转化为它对应的字符串，并显示出来。

hashmap 初始容量设置=4*(你要放元素的数量)/3+1

hashset(源码)，内部维护一个hashmap

```java
	private transient HashMap<E,Object> map;

    // Dummy value to associate with an Object in the backing Map
    private static final Object PRESENT = new Object();

    /**
     * Constructs a new, empty set; the backing <tt>HashMap</tt> instance has
     * default initial capacity (16) and load factor (0.75).
     */
    public HashSet() {
        map = new HashMap<>();
    }

	public boolean contains(Object o) {
        return map.containsKey(o);
    }
    public boolean add(E e) {
        return map.put(e, PRESENT)==null;
    }
```

3、hashcode equals

hashcode和equals是Object的方法

Object源码：

```java
//可见object内的equals默认比较的是内存地址，这就是为什么比较自定义对象，自定义对象要重写equals
public boolean equals(Object obj) {
        return (this == obj);
    }
public native int hashCode();
```

== 和 equals区别：非基本类型==比较的是内存地址，equals比较的是值是否相同

hashset底层是hashmap

```java
private transient HashMap<E,Object> map;
// Dummy value to associate with an Object in the backing Map
private static final Object PRESENT = new Object();
```





4、基本类型和包装类型

Integer a=3 ->自动装箱及装箱调的是valueOf()

自动拆箱及拆箱，调用的是intValue()

注意:Java的**自动拆装箱是发生在编译期**，即javac编译的那一刻，而不是发生在运行期

```java
public static Integer valueOf(int i) {
        if (i >= IntegerCache.low && i <= IntegerCache.high)
            return IntegerCache.cache[i + (-IntegerCache.low)];
        return new Integer(i);
    }
public int intValue() {
        return value;
    }
```

值相等的前提下

若比较双方引用类型都是Integer，则==比较的都是内存地址，==时会自动装箱，

​	非new创建 且 范围在[-128,127]内，取的是IntegerCache的缓存，地址相同，return true

​	非new创建 且 范围**不在**[-128,127]内，会new Integer(i)，地址肯定不同，return false

​	有new创建，无论值为多少，地址一定不同，return false

若双方引用类型 存在 基本类型 ，则 包装类型 会自动拆箱，return true

```java
	    Integer a =129;
        Integer b =129;
        System.out.println(a==b);//false
        //IntegerCache范围[-128,127]，在这范围内，值引用就返回true，范围外
        Integer a1 =127;
        Integer b1 =127;
        System.out.println(a1==b1);//true
        b1.hashCode();//应该是自动装箱

        Integer a2 = 1;
        Integer b2 = new Integer(1);
        System.out.println(a2==b2);//只要a2的值等于b2，一定返回false

        Integer a3 = 999;
        int b3 = 999;
        System.out.println(a3==b3);//b3是常量基本类型，返回true
```

Q:为什么使用IntegerCache？为什么设置[-128~127]

A:不用频繁创建对象，提升效率