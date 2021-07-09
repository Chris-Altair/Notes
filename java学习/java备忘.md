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

flatMap可聚合stream

```json
{
  "Success": false,
  "ErrorCode": null,
  "Remark": "订单上传失败",
  "SuccessOrders": [
  ],
  "FailOrders": [
    {
      "OrderId": null,
      "TraceId": null,
      "Remark": "【XXX】错误信息1",
      "ExtendData1": null,
      "ExtendData2": null,
      "ExtendData3": null,
      "ExtendData4": null,
      "ExtendData5": null
    },
    {
      "OrderId": null,
      "TraceId": null,
      "Remark": "【XXX】错误信息2",
      "ExtendData1": null,
      "ExtendData2": null,
      "ExtendData3": null,
      "ExtendData4": null,
      "ExtendData5": null
    },
    {
      "OrderId": null,
      "TraceId": null,
      "Remark": "【XXX】错误信息3",
      "ExtendData1": null,
      "ExtendData2": null,
      "ExtendData3": null,
      "ExtendData4": null,
      "ExtendData5": null
    },
    {
      "OrderId": null,
      "TraceId": null,
      "Remark": "【XXX】错误信息4",
      "ExtendData1": null,
      "ExtendData2": null,
      "ExtendData3": null,
      "ExtendData4": null,
      "ExtendData5": null
    }
  ]
}
```



```java
final JSONObject data = JSON.parseObject("@json")
        //拼接错误信息
        final String errorMsg = Optional.ofNullable(data.getJSONArray("FailOrders"))
                .flatMap(array -> array.stream()
                        .map(json -> (JSONObject) json)
                        .map(d -> d.getString("Remark"))
                        .reduce((msg1, msg2) -> msg1 + ";" + msg2))
                .orElse(data.getString("Remark"));
```

### 内部类使用好处

1. 内部类可以使用外部类的属性及方法（包括私有属性）

2. 封装性

3. 间接实现多重继承

```java
public class TestIterator {
    private int value;

    private class InnerClass {
        public int cal() {
            //调用外部类私有属性，可避免属性名重复
            return TestIterator.this.value * 10;
        }
    }

    public void test(boolean b){
        if (b) {
            //T1只能在if作用域内使用，还有这种操作
            class T1{
            }
            T1 t1 = new T1();
        }
    }
}
```

### 高效复制数组方法

```java
//native方法
System.arraycopy(src, 0, dest, 0, length)
```

测试方法，测试复制接近8400万长度的byte[]，native方法平均15ms，for循环赋值19ms，从结果看貌似没强多少，但考虑到没固定环境比如堆内存之类的，只是粗糙测试，至少效率比for方式强。

```java
public static void main(String[] args) throws IOException {
        final int length = 0x5000000;
        System.out.println("length = " + length);//83886080
        byte[] src = new byte[length];
        byte[] dest = new byte[length];
        final long start = System.currentTimeMillis();
//        System.arraycopy(src, 0, dest, 0, length);//15ms
        copyByfor(src, dest);//19ms
        final long end = System.currentTimeMillis();
        System.out.println((end - start) + "ms");
    }
```

参考：https://segmentfault.com/q/1010000020149713

### 反射备忘

java反射判断字段是否static及final修饰，可使用Modifier类

```java
Modifier.isFinal((Field)f.getModifiers())
Modifier.isStatic((Field)f.getModifiers())
```

### 泛型整理

**在java中，泛型不能继承和实现，类型参数化**

无界通配符：?，使用无界通配符的只能用object接

<T extends V> 表示T必须是V或V的子类

<T super V>     表示T必须是V或V的父类

即定义方法可以是泛型，但只要调用就必须是具体类型

```java
//需要注意
List<Object>不是List<Object>的父类

//泛型方法
public <V> void f(V v){}
//还可以使用通配符
public <V extends App> void f(V v){
        v.test();
    }
//泛型类型可以有多个
//泛型支持多个限定，表示泛型类型必须实现多个接口；若限定中有类，则必须在第一个，表示该泛型类型继承里该类
public <A extends Serializable & Closeable, V> void f(){}
```

泛型擦除

在编译期，编译器会先检查引用类型，再把泛型去掉，ArrayList<String>和ArrayList<Integer>在运行期的类型是相同的，并且通过反射是可以将不同类型的对象放到同一个List里的

参考：https://www.cnblogs.com/hongdada/p/13993251.html

在《Effective Java》给出过一个十分精炼的回答：**producer-extends, consumer-super（PECS）**。

从字面意思理解就是，`extends`是限制数据来源的（生产者），而`super`是限制数据流入的（消费者）。而显然我们一些经常使用到的代码中也都是符合了这一规则。

```javascript
public static <T> void copy(List<? super T> dest, List<? extends T> src) {
        int srcSize = src.size();
        if (srcSize > dest.size())
            throw new IndexOutOfBoundsException("Source does not fit in dest");

        if (srcSize < COPY_THRESHOLD ||
            (src instanceof RandomAccess && dest instanceof RandomAccess)) {
            for (int i=0; i<srcSize; i++)
                dest.set(i, src.get(i));
        } else {
            ListIterator<? super T> di=dest.listIterator();
            ListIterator<? extends T> si=src.listIterator();
            for (int i=0; i<srcSize; i++) {
                di.next();
                di.set(si.next());
            }
        }
    }
```

参考：https://cloud.tencent.com/developer/article/1649866
