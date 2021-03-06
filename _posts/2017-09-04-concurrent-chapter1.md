---
layout: post
title: "读书笔记-java并发编程的艺术第二章-volatile"
date: 2017-09-04
tags:
    - java    
    - 并发
keywords: java
description: 读书笔记-java并发编程的艺术第二章-volatile
---

这一章节的话，主要是讲一下在并发操作中常见的volatile、synchronized以及原子操作的相关知识。

目前看的部分主要是volatile这个关键字。

## volatile
根据Java语言规范第3版中对volatile的定义:
> Java编程语言允许线程访问共享变量，为了确保共享变量能被准备和一致地更新，线程应该确保通过排他锁单独获得这个变量。

Java语言提供了volatile，保证了所有线程能看到共享变量最新的值。

在硬件层面的话，如果说一个变量使用volatile去修饰的话，JIT编译器生成的汇编指令中会含有Lock前缀的指令。Lock前缀的指令在多核处理器下会做两件事情

1. 将当前处理器缓存行的数据写回到系统内存。
2. 写回内存的操作使得在其他CPU里缓存了该内存地址的数据无效。

一般来说为了提高处理器速度，CPU不直接和内存交互，而是将系统内存的数据读到内部缓存再进行操作，但操作完不知道什么时候会写回内存。
如果说是对声明了volatile的变量进行写操作，就会产生这条Lock前缀的指令，把缓存内的数据写回系统内存。但是其他处理器上的值可能还是旧值，因此在多处理器下，会实现缓存一致性协议，每个处理器会嗅探总线上传播的数据检查自己缓存是否有过期，如果过期的话，就把自己内部的缓存行设置为无效，当需要对这个数据进行操作时，就会重新从系统内存读取。

这个是比较底层层面了，如果就Java的内存模型而言，我有以下几点理解。

Java的内存模型中，线程之间通信是通过主内存完成的，针对同一个变量，每个线程都会有它的一个副本。

副本是在线程的局部变量表里，在要对其进行操作，需要将放入操作栈中，如果使用了volatile关键字，其他线程对这个变量进行了修改，在加载副本到操作栈之前，就会发现本地的这个副本已经失效了，之后就去主内存再读取。从而保证了共享变量在多个线程之间的可见性，即一个线程对共享变量的修改，其他的线程可以及时感知到。

在JDK后来的更新中，volatile还增加了新的作用，禁止指令重排序，这个得看到具体后面的内容再做记录吧~

下回再见~