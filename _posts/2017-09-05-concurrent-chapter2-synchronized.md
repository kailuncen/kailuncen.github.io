---
layout: post
title: "读书笔记之<<java并发编程的艺术-第二章>>之synchronized"
date: 2017-09-05
tags:
    - java    
    - 并发
keywords: java
description: 读书笔记之<<java并发编程的艺术-第二章>>之synchronized
---

在之前的文章中学习了volatile关键字，volatile可以保证变量在线程间的可见性，但他不能真正的保证线程安全。

```java
/**
 * @author cenkailun
 * @Date 9/5/17
 * @Time 20:23
 */
public class ConcurrentAddWithVolatile implements Runnable {

    private static ConcurrentAddWithVolatile instance = new ConcurrentAddWithVolatile();
    private static volatile int i = 0;


    public static void increase() {
        i++;
    }

    public void run() {
        for (int j = 0; j < 1000000; j++) {
            i++;
        }
    }

    public static void main(String[] args) throws InterruptedException {
        Thread t1 = new Thread(instance,"线程1");
        Thread t2 = new Thread(instance, "线程2");
        t1.start();
        t2.start();
        t1.join();
        t2.join();
        System.out.println(i);
    }
}

```
如上述代码所示，如果说两个线程是正确的并发执行的话,最后得到的结果应该是2000000，但结果往往是小于2000000。那么这是为什么呢？

经过阅读书籍，可以得知，i++的这个操作，其实是要分成3步。

```
1. 读取i的当前值到操作栈
2. 对i的当前值+1
3. 写回i+1后的值
```
经过了上述3步，才完成了i++ 的这个操作，volatile保证了写回内存后，i的最新值能够被其他线程获取，但i++的这三个动作不是一个整体，即不是原子操作，是可以被拆开的。

比如，线程1和2同时读取了i为0,并各自在自己的线程中计算得到i=1，先后写入这个i的值，导致虽然i++被执行了两次，但是实际i的值只增加了1。

如果要解决这个问题，就要保证多个线程在对i进行++ 这个操作时完全同步，即i++的这三步是一起完成的，当线程1在写入时，其他线程不能读也不能写，因为在线程1写完之前，其他线程读到的肯定是一个过期的数据。

Java提供了synchronized来实现这个功能，保证多线程执行时候的同步，某一时刻只有一个线程可以对synchronized关键字保护起来的区域进行操作，相对于volatile来说是比较重量级的。

Java的synchronized关键字具体表现有以下三种形式：
1. 作用于实例方法，锁的是当前实例对象。
2. 作用于静态方法，锁的是当前类。
3. 作用于代码块，锁的是Synchronized里配置的对象。

下面是一个示例，将synchronized作用于一个给定对象instance，每当线程要进入被包裹的代码块，会请求instance的锁。如果有其他线程已经持有了这把锁，那么新到的线程就必须等待，这样保证了每次只有一个线程会执行i++操作。


```java
/**
 * @author cenkailun
 * @Date 9/5/17
 * @Time 20:23
 */
public class ConcurrentAddWithVolatile implements Runnable {

    private static ConcurrentAddWithVolatile instance = new ConcurrentAddWithVolatile();
    private static volatile int i = 0;


    public static void increase() {
        i++;
    }

    public void run() {
        for (int j = 0; j < 1000000; j++) {
            synchronized (instance) {     //同步代码块
                i++;
            }
        }
    }

    public static void main(String[] args) throws InterruptedException {
        Thread t1 = new Thread(instance,"线程1");
        Thread t2 = new Thread(instance, "线程2");
        t1.start();
        t2.start();
        t1.join();
        t2.join();
        System.out.println(i);
    }
}

```

对于java中的代码块同步，JVM是基于进入和退出Monitor对象来实现代码块同步的，将monitorenter指令插入到同步代码块的开始位置，monitorexit插入到方法结束处和异常处，每一个对象都有一个monitor与之对应，当一个monitor被持有后，它将处于锁定状态。线程执行到monitorenter指令时，将会尝试获取对象所对应的monitor的所有权，即尝试获得对象的锁。

如下面字节码所示，代表上文代码中的同步代码块。

```
13: monitorenter
14: getstatic     #2                  // Field i:I
17: iconst_1
18: iadd
19: putstatic     #2                  // Field i:I
22: aload_2
23: monitorexit
```

对于实例方法或者静态方法上加的synchronized关键字，在方法上会有一个标志位代表，如下面字节码所示。

```
public synchronized void increase();
flags: ACC_PUBLIC, ACC_SYNCHRONIZED
```

在我看来，synchronized相对于volatile的强大之处在于保证了线程安全性以及做到了线程同步，同时也能做到volatile提供的线程间可见性以及有序性。从可见性上来说，线程通过持有锁的方式获取变量的最新值。从有序性上来说，synchronized限制每次只有一个线程可以访问同步的代码，无论内部指令顺序如何被打乱，jvm会保证最终执行的结果总是一样，其他线程只能在获得锁后读取结果数据，不会读到中间值，所以有序性问题也得到了解决。


