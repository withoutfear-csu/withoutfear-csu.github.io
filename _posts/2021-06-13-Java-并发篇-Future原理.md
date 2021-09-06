---
layout: post
title: Java-并发篇-Future原理
categories: Java
description: 介绍Future的原理
keywords: Java, 并发
---

本文通过对FutureTask源码的分析，简单介绍了Future的原理。



# Future

## 介绍

​	Future正如它的名字所说的那样，建模了一种异步计算，可以在未来的某个时刻获取它的值。调用方通常可以将一些(与主要逻辑**松耦合**、**潜在耗时**的)操作交给Future来处理，以使调用方得到解放，去执行其他有价值的事。

## Future接口中的方法

​	调用方可以通过get()来获得异步任务的返回结果，如果此时异步任务尚未执行结束，则阻塞当前线程进行等待。异步任务的结果可能被多个线程请求。

```java
public interface Future<V> {
		/**尝试取消任务的执行， 
      		如果任务尚未开始，取消执行；
          如果任务已经开始，根据mayInterruptIfRunning来决定是非中断执行该任务的线程以尝试结束任务
      */
    boolean cancel(boolean mayInterruptIfRunning);

    /**
     * Returns {@code true} if this task was cancelled before it completed
     * normally.
     *
     * @return {@code true} if this task was cancelled before it completed
     */
    boolean isCancelled();
    
    // 判断任务是否结束
    boolean isDone();

    /**
     * Waits if necessary for the computation to complete, and then
     * retrieves its result.
     */
    V get() throws InterruptedException, ExecutionException;

    V get(long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}
```

## Future的实现——FutureTask

<img src="\images\posts\Java-concurrency\FutureTask继承关系.png" alt="FutureTask继承关系" style="zoom:50%;" />

​	FutureTask是Future接口的一个实现，因此可以通过对FutureTask的源码进行分析以了解Future是如何实现异步计算的。

# FutureTask部分源码解读

## FutureTask的成员变量

```java
/** 任务执行状态 **/
private volatile int state;

/** 需要被执行的异步任务 **/
private Callable<V> callable;

/** 调用get()方法之后返回的 结果 或 异常 */
private Object outcome; // non-volatile, protected by state reads/writes

/** The thread running the callable; CASed during run() */
private volatile Thread runner;

/** Treiber stack of waiting threads */
/** 等待线程的堆栈(以链表的形式) **/
private volatile WaitNode waiters;
static final class WaitNode {
    volatile Thread thread;
    volatile WaitNode next;
    WaitNode() { thread = Thread.currentThread(); }
}

private static final sun.misc.Unsafe UNSAFE;
private static final long stateOffset;
private static final long runnerOffset;
private static final long waitersOffset;
// 在初始化时获取偏移
static {
    try {
        UNSAFE = sun.misc.Unsafe.getUnsafe();
        Class<?> k = FutureTask.class;
        stateOffset = UNSAFE.objectFieldOffset
            (k.getDeclaredField("state"));
        runnerOffset = UNSAFE.objectFieldOffset
            (k.getDeclaredField("runner"));
        waitersOffset = UNSAFE.objectFieldOffset
            (k.getDeclaredField("waiters"));
    } catch (Exception e) {
        throw new Error(e);
    }
}
```



## state的几种状态

```java
private static final int NEW          = 0;
private static final int COMPLETING   = 1;
private static final int NORMAL       = 2;
private static final int EXCEPTIONAL  = 3;
private static final int CANCELLED    = 4;
private static final int INTERRUPTING = 5;
private static final int INTERRUPTED  = 6;
```



## FutureTask的构造方法

由构造方法可知

- FutureTask的实现依赖于Callable。及时入参有Runnable，也通过了适配器转换成了Callable
- 任务在新建的时候，**state**就被设置为**NEW**，为了确保线程间的可见性，将state用**volatile**关键词修饰

**构造函数**

```java
public FutureTask(Callable<V> callable) {
    if (callable == null)
        throw new NullPointerException();
    this.callable = callable;
    this.state = NEW;       // ensure visibility of callable
}

public FutureTask(Runnable runnable, V result) {
    this.callable = Executors.callable(runnable, result);
    this.state = NEW;       // ensure visibility of callable
}
```

## FutureTask.run方法

```java
public void run() {
		// 要求任务状态为 NEW 
    if (state != NEW ||
    // 保障只为异步任务分配了一个线程
        !UNSAFE.compareAndSwapObject(this, runnerOffset,// var1：要修改的字段所在的对象， var2：字段在对象内的偏移
                         null, Thread.currentThread())) // var4：字段的期望值， var5：用于更新的值(仅在等于期望值时更新)
        return;
  
    try {
      	// 获取构造函数中传入的Callable对象
        Callable<V> c = callable;
        if (c != null && state == NEW) {
            // 异步任务的执行结果
            V result;
          	// 判断异步任务是否执行成功
            boolean ran;
            try {
              	// 执行call()方法中封装的具体逻辑
                result = c.call();
                ran = true;
            } catch (Throwable ex) {
                result = null;
                ran = false;
              	// 1、将 state 设置为 EXCEPTIONAL
              	// 2、将异常赋值给 outcome
                setException(ex);
            }
            if (ran)
              	// 1、将 state 设置为 NORMAL
              	// 2、将 result 赋值给 outcome
                set(result);
        }
    } finally {
        // runner 必须在status被设置前保持非空，以防并发调用run()
        runner = null;
        // state must be re-read after nulling runner to prevent
        // leaked interrupts
        int s = state;
        // 以下只在调用了 cancel() 的情况下执行
        if (s >= INTERRUPTING)
            handlePossibleCancellationInterrupt(s);
    }
}
```

### **run方法的主要流程**

![run流程](\images\posts\Java-concurrency\run流程.jpg)

## FutureTask中的set和setException方法

​	这两个方法的逻辑相似，可以类比来看。

**setException**

```java
protected void setException(Throwable t) {
    // 若调用了cancel()方法，就可能使得当前 status 不是 NEW。所以需要检查（如果已经被cancel了，就不需要继续执行了）
    if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
        outcome = t;
        UNSAFE.putOrderedInt(this, stateOffset, EXCEPTIONAL); // final state
        // 
        finishCompletion();
    }
}
```



**set**

```java
protected void set(V v) {
    // 若调用了cancel()方法，就可能使得当前 status 不是 NEW。所以需要检查
    if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
        outcome = v;
        UNSAFE.putOrderedInt(this, stateOffset, NORMAL); // final state
        finishCompletion();
    }
}
```

setException中state的变化状态：**NEW -> COMPLETING -> EXCEPTIONAL**

set中state的变化状态：**NEW -> COMPLETING -> NORMAL**

`Tips`: 从源码可知：不管任务是否正常执行，都会调用 **finishCompletion()** 方法

## FutureTask.finishCompletion方法

​	由于其他线程在调用get()方法时，任务可能尚未执行完，这时其他线程就会进入等待状态并进入**waiters队列**中以等待任务结果的返回。这些等待中的线程需要在结果准备好后被唤醒，函数finishCompletion就负责这一功能。

> 值得一提的是，**waiters队列**是以栈的形式存在的。

```java
private void finishCompletion() {
    // assert state > COMPLETING;
    // 当线程调用了Future.get()方法的时候，会使线程等待
    for (WaitNode q; (q = waiters) != null;) {
      	// 将 waiters 设置为 null
        if (UNSAFE.compareAndSwapObject(this, waitersOffset, q, null)) {
            for (;;) {
              	// 尝试获取阻塞的线程
                Thread t = q.thread;
                // 如果有线程处于阻塞状态，就将其唤醒
                if (t != null) {
                    q.thread = null;
                    // 将被阻塞的线程唤醒
                    LockSupport.unpark(t);
                }
                // 取得下一个等待线程节点
                WaitNode next = q.next;
                // 若阻塞线程都被唤醒，则退出
                if (next == null)
                    break;
                q.next = null; // unlink to help gc
                q = next;
            }
            break;
        }
    }

    done();

    callable = null;        // to reduce footprint
}
```

## FutureTask.handlePossibleCancellationInterrupt方法

​	如果其他线程调用了cancel，需要让出CPU让cancel先执行完成

```java
private void handlePossibleCancellationInterrupt(int s) {
    //如果当前 state = INTERRUPTING， 让出CPU
    if (s == INTERRUPTING)
        while (state == INTERRUPTING)
            Thread.yield(); // wait out pending interrupt
}
```



## FutureTask.get方法

​	这里仅列出了没有设置超时时间的get方法。在任务还没执行结束时，需要调用awaitDone方法将线程阻塞。report方法是根据state的状态来控制返回结果。

**get和report**

```java
public V get() throws InterruptedException, ExecutionException {
    int s = state;
    // 若任务还在执行过程中，需要等待
    if (s <= COMPLETING)
        s = awaitDone(false, 0L);
    // 3种结果：1、正常执行结束；2、执行异常；3执行中断
    return report(s);
}

private V report(int s) throws ExecutionException {
    Object x = outcome;
    if (s == NORMAL)
        return (V)x;
    if (s >= CANCELLED)
        throw new CancellationException();
    throw new ExecutionException((Throwable)x);
}
```

## FutureTask.awaitDone方法

​	注意到该方法下有一个死循环，一个线程可能会多次进入这个循环，但每次只能走一个分支。

```java
private int awaitDone(boolean timed, long nanos)
    throws InterruptedException {
    final long deadline = timed ? System.nanoTime() + nanos : 0L;
    WaitNode q = null;
    boolean queued = false;
    for (;;) {
        // 若获取结果的当前线程被其他线程中断，此时移除该线程WaitNode链表节点，并抛出InterruptedException；
        if (Thread.interrupted()) {
            removeWaiter(q);
            throw new InterruptedException();
        }

        int s = state;
        // 分支一，如果当前任务执行结束或着被取消，则返回结果
        if (s > COMPLETING) {
            if (q != null)
                // 见Q1：
                q.thread = null;
            return s;
        }
        
        // 分支二，如果任务还在执行阶段，想要得到任务结果的线程就要让出CPU
        else if (s == COMPLETING) // cannot time out yet
            Thread.yield();
        
        
        // 以下分支仅当 state = NEW 时才会走

        
        // 分支三，为当前线程创建节点 但没入等待队列，此时是孤立的 WaitNode
        else if (q == null)
            q = new WaitNode();
        // 分支四，将孤立的WaitNode入队，这一操作和入栈一样
        else if (!queued)
            queued = UNSAFE.compareAndSwapObject(this, waitersOffset,
                                                 q.next = waiters, q);
        // 分支五， 对于设置超时时间的get才有用
        else if (timed) {
            nanos = deadline - System.nanoTime();
            if (nanos <= 0L) {
                removeWaiter(q);
                return state;
            }
            LockSupport.parkNanos(this, nanos);
        }
        // 分支六，阻塞当前线程
        else
            LockSupport.park(this);
    }
}
```

>**问: finishCompletion()已经将所有的WaitNode节点置空了，那为何还要调用q.thread = null？**
>
>> **答**: 分支三会产生孤立的**WaitNode**节点。



## FutureTask.cancel方法

​	参数mayInterruptIfRunning=true时：status变化过程：NEW->INTERRUPTING->CANCELLED

​	参数mayInterruptIfRunning=false时：status变化过程：NEW->CANCELLED	

```java
public boolean cancel(boolean mayInterruptIfRunning) {
    // 如果 state 的状态不为 new，说明已经处于set()、或者setException流程中
    // 若允许中断，就将state设为中间状态 INTERRUPTING
    // 若不允许中断，就直接将 state 设为 CANCELLED
    if (!(state == NEW &&
          UNSAFE.compareAndSwapInt(this, stateOffset, NEW,
              mayInterruptIfRunning ? INTERRUPTING : CANCELLED)))
        // 在这退出时， state = CANCELLED；
        return false;
    
    try {    // in case call to interrupt throws exception
        // 若允许打断
        if (mayInterruptIfRunning) {
            try {
                Thread t = runner;
                if (t != null)
                    // 只是设置了中断标志，而不是真的中断了线程。
                    t.interrupt();
            } finally { // final state
                UNSAFE.putOrderedInt(this, stateOffset, INTERRUPTED);
            }
        }
    } finally {
        finishCompletion();
    }
    // state = INTERRUPTED
    return true;
}
```

# 总结

​	需要将实现了Callable接口的对象传给FutureTask，才能将FutureTask作为一个异步任务提高给线程执行。FutureTask内部通过维护一个状态status，来判断任务的执行情况，所有操作都围绕着这一状态开展。当有多个线程并发获取异步任务的结果时，（若任务尚未执行结束）这些线程会被放入等待队列中等待，直到任务结束时这些线程才会被唤醒。