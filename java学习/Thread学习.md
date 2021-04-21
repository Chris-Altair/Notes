[TOC]

## 非静态方法

### *setDaemon()*

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

### *interrupt（）*

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

## 静态方法

### *interrupted()*

*Thread.interrupted()*方法做两件事:

- 返回当前中断状态
- 清除中断状态，不能恢复false为true，只能把true清除为false

```java
//源码
public static boolean interrupted() {
        return currentThread().isInterrupted(true);
    }
```

### *sleep(ms)*

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

