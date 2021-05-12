Java编译后出现$1.class、$2.class等多个class文件，如果java文件里存在匿名类会编译出多个class文件。

### java assert关键字

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

推荐使用**静态方法**创建**特定类型**的集合,语义更清晰

```java
//这两种方式创建的集合不能add和remove元素,否则会抛出UnsupportedOperationException
List<String> singletonList = Collections.singletonList("XXX");
List<String> emptyList = Collections.emptyList();
```

java方法是参数是赋值操作（基本类型赋值，对象赋给地址），jvm每个线程有自己的stack（所以各个线程调用方法内部的局部变量是不会互相干扰的），所有线程的对象存到heap，java方法的局部变量（基本类型和对象地址）存到栈内，java方法栈大小编译期就能计算好

### Optional用法

Optional最经典的null-safe get chain

```java
/*某个环节为空也不会报错*/
return Optional.ofNullable(a)
    .map(A::getB)
    .map(B::getC)
    .map(C::getD)
    .orElse(null);
```