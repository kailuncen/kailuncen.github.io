---
layout: post
title: "Java并发笔记之 Race Condition and Critical Section"
date: 2017-09-08
tags:
    - java    
    - 并发
keywords: java
description: Java并发笔记之 Race Condition and Critical Section
---
### 前言
这几天学习并发编程,[race-conditions-and-critical-sections](http://tutorials.jenkov.com/java-concurrency/race-conditions-and-critical-sections.html),翻译一下，写点自己的笔记并加上点个人的理解。

网页中里中提到两个名词Race Condition 和 Critical Section，接下来对他们进行解释和例子演示。

### Race Condition

在多线程场景下，当多个线程访问同一块资源，且执行结果与线程访问的先后顺序相关，即表明这里面存在着Race Condition，中文翻译即竞争条件。

看下面👇的代码,多个线程都会调用add方法对同一个count值进行加法。

```
  public class Counter {

     protected long count = 0;

     public void add(long value){
         this.count = this.count + value;
     }
  }
```

然而,add方法中的加法需要好几个步骤才能完成。


```
1. 从内存中读取count的值到寄存器。
2. 加value。
3. 写回内存。
```

如果有两个线程都对add方法进行了操作，比如线程A加3,线程B加2,我们的预期结果是5。由于线程的访问顺序以及切换的时间是不可预期的,在特定的访问顺序下，可能出现一些出乎意料的结果,比如下文中的执行顺序。

```
A:  Reads this.count into a register (0)
B:  Reads this.count into a register (0)
B:  Adds value 2 to register
B:  Writes register value (2) back to memory. this.count now equals 2
A:  Adds value 3 to register
A:  Writes register value (3) back to memory. this.count now equals 3

```

由于加法不是原子性的，在加法执行过程中的每一步都可能存在着线程切换。
比如线程A和B都先后读到0,然后线程B占用了时间片完成了加2的操作,写回了内存，此时内存中count的值等于2。
然后线程A重新得到调度，此时线程A内部的count值还是0,线程A对主内存内count的变化是不可见的,然后线程完成加3操作，写回内存，此时count值等于3。

上述代码中的add方法内部就存在着竞争条件，会根据线程执行顺序的不确定性影响最后的执行结果。

### Critical Section

我们把会导致Race Condition的区域称为Critical Section,中文翻译临界区。临界区即每个线程中访问临界资源的那段代码。

在上文的代码中,this.count就是临界资源

```
this.count = this.count + value
```

就是临界区,为了保证执行结果的正确性,避免临界区内产生竞争条件,我们需要确保临界区内的执行是原子的,每次仅允许一个线程进去,进入后不允许其他线程进入。

我们可以采用线程同步做到以上的要求，线程同步可以使用synchronized同步代码,或者locks,或者是原子变量比如AtomicInteger等。

可以把整个临界区使用synchronized同步，但把临界区拆分成多个小的临界区能够降低对共享资源的争夺，增加整个临界区的吞吐量,下面举个例子。


```java
public class TwoSums {
    
    private int sum1 = 0;
    private int sum2 = 0;
    
    public void add(int val1, int val2){
        synchronized(this){
            this.sum1 += val1;   
            this.sum2 += val2;
        }
    }
}
```

在上述代码中,简单的做法就是锁住整个对象,只有一个线程能够执行两个不同变量的加法操作。然而，由于这两个变量是互相独立的，可以拆分到两个不同的synchronized块中。


```java
public class TwoSums {
    
    private int sum1 = 0;
    private int sum2 = 0;

    private Integer sum1Lock = new Integer(1);
    private Integer sum2Lock = new Integer(2);

    public void add(int val1, int val2){
        synchronized(this.sum1Lock){
            this.sum1 += val1;   
        }
        synchronized(this.sum2Lock){
            this.sum2 += val2;
        }
    }
}
```

改动后，两个线程可以同时在add方法中操作，一个线程在第一个synchronized块，另一个线程在第二个synchronized块，两个synchronized块同步的是不同的对象，所以两个线程可以独立执行，整体线程等待的时间会变少，吞吐量能够得到提升。

