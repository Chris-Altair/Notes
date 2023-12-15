[TOC]

## Thread类方法

### 1.非静态方法

#### *setDaemon()*

*setDaemon*方法：将线程设置为守护线程

- 若线程当前状态为**正常活动状态**,抛出*IllegalThreadStateException*
- 否则将线程设置为守护线程

java中有两种类型的thread，***user threads*** 和 ***daemon threads***。

User threads是高优先级的thread，JVM将会等待所有的User Threads运行完毕之后才会结束运行。

daemon threads是低优先级的thread，它的作用是为User Thread提供服务。 因为daemon threads的低优先级，并且仅为user thread提供服务，所以当所有的user thread都结束之后，JVM会自动退出，不管是否还有daemon threads在运行中。

因为这个特性，所以我们**通常在daemon threads中处理无限循环的操作，因为这样不会影响user threads的运行。**

daemon threads并不推荐使用在I/O操作中。

但是有些不当的操作也可能导致daemon threads阻塞JVM关闭，比如在daemon thread中调用join（）方法。

```java
Thread thread = new Thread(() -> {
            String threadName = Thread.currentThread().getName();
            int i = 0;
            while (true) {
                System.out.println(threadName + ":" + (i++));
            }
        });
		//注释掉则main线程不会终止,因为thread默认是用户线程,jvm必须等全部用户线程运行完才会停止,所以一直执行任务
		//setDaemon(true)则main线程执行完代码就终止了
        thread.setDaemon(true);
        thread.start();
```

#### *interrupt（）*

*interrupt*方法：将线程的中断标识设置为*true*

- 若线程当前状态为**被阻塞状态**（如*sleep、wait、join*等状态），则线程将立即退出被阻塞状态，并会抛出*InterruptedException*
- 若线程当前状态为**正常活动状态**，则会将线程的中断标识设置为true（**仅set标识，该线程将继续正常运行，不受影响**）

```java
//interrupt源码
public void interrupt() {
        if (this != Thread.currentThread())
            //若非当前线程，则判断当前正在运行的线程是否有权限修改此线程，无权则抛出SecurityException
            checkAccess();
        synchronized (blockerLock) {
            Interruptible b = blocker;
            if (b != null) {
                interrupt0();           // Just to set the interrupt flag
                b.interrupt(this);
                return;
            }
        }
        interrupt0();//native方法
    }
private native void interrupt0();
```

常见用法：

```java
Runnable runnable = () -> {
            while (!Thread.currentThread().isInterrupted()) {
                //正常任务
            }
    		//中断处理任务
        };
Thread thread = new Thread(runnable);
thread.start();
// 一段时间后
thread.interrupt();
```

#### *join()*

作用：主线程等待该线程结束。

 Join方法实现是通过wait（Object 提供的方法）。 当main线程调用t.join时候，main线程会获得线程对象t的锁（wait 意味着拿到该对象的锁),调用该对象的wait(等待时间)，直到该对象唤醒main线程 ，比如退出后。这就意味着main 线程调用t.join时，必须能够拿到线程t对象的锁。 

如果t执行执行过程中抛出Exception，t.join()后的代码仍会执行

若main方法中断，不会影响t线程的执行

```java
main() {
    Thread t = new Thread();
    t.start();
    t.join();
    //当t线程执行完或者t抛出异常后，才会执行main后续的逻辑
    ...
}
```

### 2.静态方法

#### *interrupted()*

*Thread.interrupted()*方法做两件事:

- 返回当前中断状态
- 清除中断状态，不能恢复false为true，只能把true清除为false

```java
//源码
public static boolean interrupted() {
        return currentThread().isInterrupted(true);
    }
```

#### *sleep(ms)*

*Thread.sleep(ms)*，当前线程变为***TIMED_WAITING***状态，等待时间单位为ms（超时等待，超时后会自动唤醒）

*sleep*方法和*wait*方法的区别：**sleep不释放锁，wait会释放锁**

## ThreadLocal详解

threadLocal线程缓存，可以在线程内共享变量，对不同线程隔离

其本质是将变量set到thread对象的threadLocals内

```java
/* ThreadLocal values pertaining to this thread. This map is maintained
     * by the ThreadLocal class. */
ThreadLocal.ThreadLocalMap threadLocals = null;
```

threadLocal的源码

```java
public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);//可见拿的就是线程对象的threadLocals
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }
ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
    }

public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        return setInitialValue();
    }
```

threadLocalMap内部是Entry[]数组，注意key用WeakReference（弱引用）包了一层，为什么要这么做呢？是为了防止内存泄漏，当没有该threadLocal的强引用时，如果执行gc，则会收回该threadLocal，即便如此还是推荐在线程后remove掉

![](https://pic3.zhimg.com/v2-90f2fe9b7c61840080a6a533b2798fce_b.jpg)

```java
static class Entry extends WeakReference<ThreadLocal<?>> {
            /** The value associated with this ThreadLocal. */
            Object value;

            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }
```

弱引用测试

```java
	static class App {
        public App() {}
    }

    public static void main(String[] args) throws InterruptedException {
        //App app = new App()这种方式对象被强引用，无法回收
        WeakReference<App> reference = new WeakReference<>(new App());
        System.out.println("app:" + reference.get());
        System.gc(); //参数加上-XX:+PrintGCDetails，输出gc信息
        Thread.sleep(5000); //保证gc尽可能回收
        if (reference.get() == null) {
            System.out.println("app cleared");//表示已回收
        }
    }
```

需要注意，不同的threadLocal的set都是放在该线程的threadLocals里，key就是对应ThreadLocal对象的弱引用

```java
ThreadLocal<String> ts = new ThreadLocal<>();
ThreadLocal<Integer> ti = new ThreadLocal<>();
//这两个set都是放到当前thread的threadLocals里
ts.set("123");ti.set(1);
```

## Runtime相关

### 钩子函数addShutdownHook

```java
//看了下阿里巴巴创建链接池中调用了addShutdownHook，所以研究一下
public SyncAPIClient createAPIClient(ClientPolicy policy, AuthorizationTokenStore authorizationTokenStore) {
        SerializerProvider serializerProvider = this.initSerializerProvider();
        final SyncAPIClient syncAPIClient = new SyncAPIClient(policy, serializerProvider, authorizationTokenStore);
        syncAPIClient.start();
        Runtime.getRuntime().addShutdownHook(new Thread() {
            public void run() {
                if (syncAPIClient != null) {
                    syncAPIClient.shutdown();
                }
            }
        });
        return syncAPIClient;
    }
```


addShutdownHook钩子函数，在程序正常终止运行，会启用这个线程

```java
Runtime.getRuntime().addShutdownHook(Thread thread)
```

addShutdownHook方法会将线程塞入hooks里，ApplicationShutdownHooks部分源码如下

```java
private static IdentityHashMap<Thread, Thread> hooks;
    private ApplicationShutdownHooks() {}

    /* Add a new shutdown hook.  Checks the shutdown state and the hook itself,
     * but does not do any security checks.
     */
    static synchronized void add(Thread hook) {
        //校验方法省略。。。
        //可见key和value都是传的线程
        hooks.put(hook, hook);
    }
```

IdentityHashMap与正常HashMap的区别在于HashMap是通过key的HashCode判断key，而IdentityHashMap是通过key的地址来判断

```java
//一个典型的例子，可见即使hash相同，map仍然有3个元素
static class Key {
        @Override
        public int hashCode() {
            return 1;
        }
    }
    private static void testIdentityHashMap() {
        IdentityHashMap<Object, Integer> map = new IdentityHashMap<>();
        Key k1 = new Key();
        Key k2 = new Key();
        Key k3 = new Key();
        map.put(k1,1);
        map.put(k2,2);
        map.put(k3,3);

        System.out.println(map.get(k1));//1
        System.out.println(map.get(k2));//2
        System.out.println(map.get(k3));//3
    }
```

至于为什么要使用IdentityHashMap。。。

如果让我写，最自然的想法是使用list，有一个钩子就加一条，这样的问题是如果我加了两次同样的thread（地址相同），那就会执行两边，同样的线程显然不应该执行多次。

所以我这个集合显然要有某种去重的机制，那我用HashSet（本质是hashMap）之类的？这些归根结底是通过thread的hashcode比较的，这样会存在什么问题呢？

~~我个人觉得通过hashcode判断多此一举，实质上，我通过钩子函数创建线程执行逻辑，显然是我建了几个线程就要执行几个，依据应该是地址，而不应该通过hashcode来判断，更何况我可以继承线程类重新hashcode,如果写的有问题的话，还会存在丢失执行的问题，所以使用以地址为判断依据的IdentityHashMap就比较合理。~~

Thread沒有实现hashCode方法，所以使用了IdentityHashMap

那么如何执行传入的线程呢，答案在ApplicationShutdownHooks里

```java
static {
        try {
            //主程序shutdown会执行runHooks方法
            Shutdown.add(1 /* shutdown hook invocation order */,
                false /* not registered if shutdown in progress */,
                new Runnable() {
                    public void run() {
                        runHooks();
                    }
                }
            );
            hooks = new IdentityHashMap<>();
        } catch (IllegalStateException e) {
            // application shutdown hooks cannot be added if
            // shutdown is in progress.
            hooks = null;
        }
    }

/* Iterates over all application hooks creating a new thread for each
     * to run in. Hooks are run concurrently and this method waits for
     * them to finish.
     */
    static void runHooks() {
        Collection<Thread> threads;
        synchronized(ApplicationShutdownHooks.class) {
            threads = hooks.keySet();
            hooks = null;
        }
        for (Thread hook : threads) {
            //并发执行
            hook.start();
        }
        for (Thread hook : threads) {
            while (true) {
                try {
                    //当前线程等待所有钩子线程执行完或抛出异常则退出
                    hook.join();
                    break;
                } catch (InterruptedException ignored) {
                }
            }
        }
    }
```
## wait和notify

wait和notify必须在同步代码块使用，wait会释放当前持有锁的线程释放锁并进入等待状态，notify会唤醒一个等待的线程（不一定是随机的，跟jvm实现有关， jdk1.8, hotspot是按**先入先出**实现的）**但并不会释放锁**，只有执行wait或者执行完该锁的同步代码块后才会释放锁，并且**唤醒的线程也只有抢到锁后才会真正执行**
