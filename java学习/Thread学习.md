[TOC]

## 非静态方法

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

*Thread.sleep(ms)*，当前线程变为***TIMED_WAITING***状态，等待时间单位ms（超时等待，超时后会自动唤醒）

*sleep*方法和*wait*方法的区别：**sleep不释放锁，wait会释放锁**