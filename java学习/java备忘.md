Java编译后出现$1.class、$2.class等多个class文件，如果java文件里存在匿名类会编译出多个class文件。

### java注释文档

1. class

   ```java
   /**
    * @description: 文件备注
    * @author: ${USER}
    * @Date: ${DATE}
    * @see: 类/方法路径 指向内部路径
    * @link <a href="https://www.example.com">External Link</a> 指向外部链接
    */
   ```

2. method

   ```java
   /**
    * @description: 文件备注
    * @param param1
    * @param param2
    * @return 返回结果
    * @throws exceptionName
    * @see: 类/方法路径 指向内部路径
    * @link <a href="https://www.example.com">External Link</a> 指向外部链接
    */
   ```

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

### synchronized String骚操作

看了看朋友的代码，将字符串字符串对应的常量池对象作为锁，利用intern方法保证了相同字符串的唯一性，很巧妙的做法

```java
void method(String key) {
    synchronized(key.intern()) {
        //...
    }
}
```

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

### protected

`private`属性或方法子类是没法访问，所以当你希望子类可以访问并且又不希望外界可以随意访问，就可以使用`protected`，这样就只有子类和包内的类可以访问

### 内部类使用好处

1. 内部类可以使用外部类的属性及方法（包括私有属性）

2. 外部类可以调用内部类的方法（调用非静态内部类的方法需要先创建对象，静态内部类则直接类引用即可）

3. 封装性

4. 间接实现多重继承

```java
public class TestIterator {
    private int value;

    private class InnerClass {
        public int cal() {
            //调用外部类私有属性，可避免属性名重复
            // return value; 也可
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
Arrays.copyOf()
//native方法
System.arraycopy(src, 0, dest, 0, length)
```

测试方法，简单测试复制接近8400万长度的byte[]，native方法平均15ms，for循环赋值19ms，考虑到没固定环境比如堆内存之类的，只是粗糙测试，效率比for方式强。

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

### 重写hashCode

可参考Arrays.hashCode(Object[] o)

todo 之所以使用31，可能是为了错开？，有时间研究一下

```java
//31 * result = result << 5 - result
public static int hashCode(Object a[]) {
        if (a == null)
            return 0;
        int result = 1;
        for (Object element : a)
            result = 31 * result + (element == null ? 0 : element.hashCode());
        return result;
    }
```

Objects.hash方法其实就是调用的Arrays.hashCode

```java
public static int hashCode(Object o) {
        return o != null ? o.hashCode() : 0;
    }
public static int hash(Object... values) {
        return Arrays.hashCode(values);
    }
```

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
List<Object>不是List<T>的父类

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

Java的泛型基本上都是在编译器这个层次上实现的，在生成的字节码中是不包含泛型中的类型信息的。

继承泛型类使用了JVM桥方法，通过桥方法将重写的object继承方法转到具体泛型的重载的方法

为什么要使用泛型擦除呢？是为了保证兼容性（老版本的代码在新版本环境仍可执行）

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

### ArrayList vs LinkedList

LinkedList的作者都说没用过LinkedList。。。

事实上由于现代计算机结构，ArrayList效率远高于LinkedList，并且对较大的列表而言，LinkedList消耗的空间大概是ArrayList的4-6倍，所以任何情况下都应该使用ArrayList，若你想使用stack\queue\deque，那也应该使用ArrayDeque而不是LinkedList。

ps: `new ArrayList<>()`和`new ArrayList<>(0)`得到的列表的扩容方式是不同的，显式传入0会遵循1.5倍扩容原则1->2->3->4->5->6...，若不传则第一次add默认扩充为10，所以如果列表经常为空或者只放一两个元素，`new ArrayList<>(0)`会省点内存

### 迭代器

**特点：轻量级对象；支持在遍历内删除元素**

以ArrayList为参考，ArrayList在通过迭代器遍历时，不允许在原list上增删元素（但可以通过迭代器的remove方法删除，实际上创建迭代器后就不能删除，因为创建迭代器后expectedModCount已经固定了），因为增删操作会修modCount，迭代器next、remove方法操作前会校验modCount是否改变，改变则抛出ConcurrentModificationException

从源码可见，每个remove调用前，一定要先调用next方法,不然lastRet=-1直接抛出异常

```java
private class Itr implements Iterator<E> {
        int cursor;       // index of next element to return
        int lastRet = -1; // index of last element returned; -1 if no such
        int expectedModCount = modCount;

        Itr() {}

        public boolean hasNext() {
            return cursor != size;
        }

    	// Object n = next()方法调用后，cursor其实是下一个元素的index，lastRet是n的index
        @SuppressWarnings("unchecked")
        public E next() {
            checkForComodification();//操作前都会校验list是否增删
            int i = cursor;
            if (i >= size)
                throw new NoSuchElementException();
            Object[] elementData = ArrayList.this.elementData;
            if (i >= elementData.length)
                throw new ConcurrentModificationException();
            cursor = i + 1;
            return (E) elementData[lastRet = i];
        }

        public void remove() {
            if (lastRet < 0)
                throw new IllegalStateException();
            checkForComodification();//操作前都会校验list是否增删

            try {
                ArrayList.this.remove(lastRet);
                //删除元素后根据当前元素的位置重置迭代器，保证remove操作不会干扰迭代器遍历
                //即删除元素后，从下个元素开始遍历
                cursor = lastRet;
                lastRet = -1;
                expectedModCount = modCount;//删除会重置expectedModCount
            } catch (IndexOutOfBoundsException ex) {
                throw new ConcurrentModificationException();
            }
        }

        @Override
        @SuppressWarnings("unchecked")
        public void forEachRemaining(Consumer<? super E> consumer) {
            ...
        }
		/**
		  * 快速失败机制，校验list内modCount是否变化(增删操作会修改modCount)，变化则直接抛异常
		  */
        final void checkForComodification() {
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
        }
    }

```

#### java迭代器接口

```java
public interface Iterator<E> {
    boolean hasNext();
    E next();
    
    default void remove() {
        throw new UnsupportedOperationException("remove");
    }
    default void forEachRemaining(Consumer<? super E> action) {
        Objects.requireNonNull(action);
        while (hasNext())
            action.accept(next());
    }
}

```

#### 使用示例

```java
public static void main(String[] args) {
        final List<Integer> integers = new ArrayList<>();
        integers.add(0);
        integers.add(1);
        integers.add(2);
        integers.add(3);
        integers.add(4);
        integers.add(5);
        final Iterator<Integer> iterator = integers.iterator();
        int i = 0;
        while (iterator.hasNext()) {
            final Integer next = iterator.next();
            if (next % 2 != 0) {//移除奇数元素
                System.out.println(i + ":" + integers.get(i) + " " + next); //这两个值相同
                iterator.remove();//相当于把当前next元素移除，下个元素接上，所以i不变，这样下次循环get(i)就是下个值
                continue;
            }
            i++;
        }
        System.out.println(integers.toString());
    }
------返回------
1:1 1
2:3 3
3:5 5
[0, 2, 4]

```

#### ListIterator用法

```java
public interface ListIterator<E> extends Iterator<E> {
    //正序遍历
    boolean hasNext();
    E next();
    int nextIndex();     //此方法返回调用 next() 函数时将返回的元素的索引
    
    //倒序遍历
    boolean hasPrevious(); 
    E previous();
    int previousIndex(); //此方法返回调用 previous() 函数时将返回的元素的索引
    
    /*(说白了就是修改next()获取的元素)*/
    void set(E e); //此方法将调用 next() 或 previous() 方法时返回的最后一个元素替换为指定的元素
    /*(说白了就是在next()获取的元素后插入新元素)*/
    void add(E e);
    /*(说白了就是移除next()获取的元素)*/
    void remove(); //此方法从调用 next() 或 previous() 方法元素时返回的列表中删除最后一个元素
}

```

### jdk自带lambda备忘

Predicate可用于校验方法,接受一个值返回boolean类型，支持and、or等逻辑表达式

```java
Predicate<T> <=> T -> boolean
//执行lambda表达式
predicate.test(T t)
//支持and、or等逻辑表达式
predicate1.or(predicate2).test("Hello world")

//其他的常用函数式接口
Function<T, R> <=> T->R //一元函数
UnaryOperator<T> <=> T->T //特殊的一元函数
Supplier<T> <=> ()->T //生产者
Consumer<T> <=> T->void //消费者

BiFunction<T, U, R> <=> (T, U)->R //二元函数
BiPredicate<T, U> <=> (T, U)->boolean //二元判断
BiConsumer<T, U> <=> (T, U)->void //二元消费者    

```

学到了一手，枚举还可以这样用

```java
public enum RuleValidate {
    ARGUMENT_IS_NULL(ErrorCode.ARGUMENT_ERROR, Objects::isNull),
    CUSTOMER_NOT_FOUND(ErrorCode.BUSINESS_ERROR, (String customer) -> customer.equalsIgnoreCase(""))
    //省略其他校验规则......
    ;

    private ErrorCode errorCode;
    private Predicate<String> predicate;

    RuleValidate(ErrorCode errorCode, Predicate<String> predicate) {
        this.errorCode = errorCode;
        this.predicate = predicate;
    }

    public static int verity(String customer) {
        for (RuleValidate validate : RuleValidate.values()) {
            if (validate.predicate.test(customer)) {
                return validate.errorCode.getCode();
            }
        }
        return 1;
    }
}

```

### stream备忘

```java
//stream中添加元素的方法，concat：合并stream的方法
//0加到头部，同理放后面表示追加到尾部
Stream<Integer> intStream = Stream.of(1, 2, 3, 4, 5);
Stream<Integer> newStream = Stream.concat(Stream.of(0), intStream);

//判断是否流内所有元素都满足表达式
boolean allMatch(Predicate<? super T> predicate);
//是否存在满足条表达式的元素
boolean anyMatch(Predicate<? super T> predicate);
//判断是否流内没有元素都满足表达式
boolean noneMatch(Predicate<? super T> predicate);

//mergeFunction合并方法，mergeFunction包含两个参数，第一个是之前的结果，第二个是valueMapper执行后的结果
//例如Collectors.toMap(OrderDTO::uniqueKey, orderDto -> 1, (result, count) -> result+count)
//当出现key冲突时，result就是map.get的结果(之前的结果)，count就是valueMapper执行后的结果即count=1
public static <T, K, U>
    Collector<T, ?, Map<K,U>> toMap(Function<? super T, ? extends K> keyMapper,
                                    Function<? super T, ? extends U> valueMapper,
                                    BinaryOperator<U> mergeFunction)

```

### 枚举单例模式

枚举单例模式，利用枚举特性保证单例和线程安全， 也是effective java 作者极力推荐的单例实现模式

```java
/**
 * 枚举单例模式
 */
public class SingletonDemo {

    private SingletonDemo() {}

    /**
     * 枚举
     */
    private enum Singleton {
        INSTANCE;

        private final SingletonDemo singletonDemo;

        Singleton() {
            this.singletonDemo = new SingletonDemo();
        }

        private SingletonDemo getInstance(){
            return INSTANCE.singletonDemo;
        }
    }

    public static SingletonDemo getInstance() {
        return Singleton.INSTANCE.getInstance();
    }

    public static void main(String[] args) {
        SingletonDemo.getInstance();
    }
}

```

### BitSet使用

```java
位图，内部维护一个long[]数组，每个long占8个字节即64位，前[0,63]位放到long[0]中，如果大于64位则同理将大于的位数加到long[1]里，以此类推

```

### Timer/TimerTask详解

jdk自带的定时任务，支持一次性、周期、延时任务

```java
//简单的使用
Timer timer = new Timer();
        TimerTask task = new TimerTask() {
            @Override
            public void run() {
                System.out.println("executing now!");
            }
        };
//1s后执行一次
timer.schedule(task, 1000);

```

这块的代码简洁有趣，推荐仔细阅读

首先列几个关键类

- Timer 调度器
- TaskQueue 任务队列，内部使用了二叉堆（推测是为了根据执行实际排序）
- TimerTask 任务继承了runnable接口
- TimerThread 执行任务的线程

#### ①Timer构造函数

先看Timer的构造函数，就是通过CAS生成线程名，然后启动实际执行任务的线程TimerThread ；还有其他的构造函数，区别是将线程设置为守护线程

```java
	//先看Timer的构造函数
    private final static AtomicInteger nextSerialNumber = new AtomicInteger(0);
    private static int serialNumber() {
        return nextSerialNumber.getAndIncrement();
    }
    public Timer() {
        this("Timer-" + serialNumber());
    }
    public Timer(String name) {
        thread.setName(name);
        thread.start();
    }

```

#### ②TimerThread run方法

由①可知：Timer创建后，就会启动执行任务的线程，那么我们先看一下该线程

todo Object.wait方法需要仔细梳理下，官方推荐这东西似乎要放到同步+循环里使用

**注意synchronized锁住一个对象，不代表对象的成员就不能修改**

```java
class TimerThread extends Thread {
    /**
     * 新任务是否能被调度
     */
    boolean newTasksMayBeScheduled = true;

    /**
     * 任务队列
     */
    private TaskQueue queue;

    TimerThread(TaskQueue queue) {
        this.queue = queue;
    }

    public void run() {
        try {
            mainLoop();
        } finally {
            // Someone killed this Thread, behave as if Timer cancelled
            synchronized(queue) {
                newTasksMayBeScheduled = false;
                queue.clear();  // Eliminate obsolete references
            }
        }
    }

    /**
     * The main timer loop.  (See class comment.)
     */
    private void mainLoop() {
        while (true) {
            try {
                TimerTask task;
                boolean taskFired; // 标记任务是否应该执行
                synchronized(queue) {
                    // Wait for queue to become non-empty
                    // 创建Timer后还没加任务时，就会进到这里，执行线程就会陷入等待
                    // Object.wait方法:让当前线程进入等待状态。直到其他线程调用此对象的 notify() 方法或 notifyAll() 方法
                    // 并且wait方法释放queue的锁
                    while (queue.isEmpty() && newTasksMayBeScheduled)
                        queue.wait(); 
                    if (queue.isEmpty())
                        break; // Queue is empty and will forever remain; die

                    // Queue nonempty; look at first evt and do the right thing
                    long currentTime, executionTime;
                    task = queue.getMin();
                    synchronized(task.lock) {
                        if (task.state == TimerTask.CANCELLED) {
                            queue.removeMin();
                            continue;  // No action required, poll queue again
                        }
                        currentTime = System.currentTimeMillis();
                        executionTime = task.nextExecutionTime;
                        // 当前时间戳 <= 计划执行时间戳 说明该任务需要执行
                        if (taskFired = (executionTime<=currentTime)) {
                            if (task.period == 0) { // Non-repeating, remove
                                queue.removeMin(); //一次性任务，直接移除掉，队列重排序
                                task.state = TimerTask.EXECUTED;
                            } else { // Repeating task, reschedule
                                queue.rescheduleMin( //周期性任务，重置执行时间，队列重排序
                                  task.period<0 ? currentTime   - task.period
                                                : executionTime + task.period);
                            }
                        }
                    }
                    if (!taskFired) // Task hasn't yet fired; wait
                        // 最小时间的任务都不需要执行的话，直接等待，等待任务入队
                        queue.wait(executionTime - currentTime); 
                }
                if (taskFired)  // Task fired; run it, holding no locks
                    task.run(); //执行任务
            } catch(InterruptedException e) {
            }
        }
    }
}

```

#### ③Timer.sched添加任务方法

```java
private void sched(TimerTask task, long time, long period) {
        if (time < 0)
            throw new IllegalArgumentException("Illegal execution time.");

        // Constrain value of period sufficiently to prevent numeric
        // overflow while still being effectively infinitely large.
        if (Math.abs(period) > (Long.MAX_VALUE >> 1))
            period >>= 1;

        synchronized(queue) {
            if (!thread.newTasksMayBeScheduled)
                throw new IllegalStateException("Timer already cancelled.");

            synchronized(task.lock) {
                if (task.state != TimerTask.VIRGIN)
                    throw new IllegalStateException(
                        "Task already scheduled or cancelled");
                task.nextExecutionTime = time;
                task.period = period;
                task.state = TimerTask.SCHEDULED;
            }
            // 队列添加任务，并重排序
            queue.add(task);
            if (queue.getMin() == task)
                queue.notify(); // 添加的任务就是最小的任务，则唤醒执行线程
        }
    }

```

#### ④TaskQueue 堆实现

最小堆，用于排序

```java

```

#### ⑤其他有趣的地方

Timer取消方法，清除taskQueue的全部任务，然后唤醒执行任务的线程，

```java
public void cancel() {
        synchronized(queue) {
            thread.newTasksMayBeScheduled = false;
            queue.clear();
            queue.notify();  // In case queue was already empty.
        }
    }
notify会唤醒任务线程，之后任务线程的mainLoop会执行到下面的代码然后退出
    if (queue.isEmpty())
       break;

```

Timer有个成员，作用在于当timer没有被引用时，jvm回收会执行finalize方法，该threadReaper会优先于Timer的回收；

主要是Timer的finalize很容易受到子类的影响，finalize忘记调用它。

```java
private final Object threadReaper = new Object() {
        protected void finalize() throws Throwable {
            synchronized(queue) {
                thread.newTasksMayBeScheduled = false;
                queue.notify(); // In case queue is empty.
            }
        }
    };

```

对象没有引用时，jvm会回收掉对象，此时会执行finalize方法；测试发现，会优先回收对象的成员，然后再回收对象

```java
class Oo {
        private Object o = new Object() {
            @Override
            protected void finalize() throws Throwable {
                System.out.println("回收成员");
            }
        };

        @Override
        protected void finalize() throws Throwable {
            System.out.println("回收类");
        }
    }

    @Test
    public void testFin() throws InterruptedException {
        new Oo();
        System.gc();
    }
输出：
回收成员
回收类

```



参考：https://juejin.cn/post/7044490907726544933#comment
