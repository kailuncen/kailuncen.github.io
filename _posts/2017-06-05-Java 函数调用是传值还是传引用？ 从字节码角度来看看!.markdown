---
layout: post
title: "Java 函数调用是传值还是传引用？ 从字节码角度来看看!"
date: 2017-06-05
tags:
    - java
    - 字节码
keywords: java
description: Java 函数调用是传值还是传引用？ 从字节码角度来看看!
---

#### 一个小问题
在开源中国看到这样一则问题
>https://www.oschina.net/question/2507499_2244027，其中的变量a前后的输出是什么?

我答错了，我认为传入function的就是main函数中的a，在function中修改了a的地址，因此回到主函数后，a的地址已经变成了function中所赋予的a2的地址，因此经过function处理后a的值已经改变了。
但结果并不是，因为我忽略了Java的基础知识点之一。
>Java中传参都是值传递，如果是基本类型，就是对值的拷贝，如果是对象，就是对引用地址的拷贝。

-----
下文将从字节码的角度，分析Java中基本类型传参和对象传参。

### 基本类型传参
以下是处理类Porcess，代码应该已经能够自解释了。function1是将传参a变成2，function2是初始化int b，赋值为5，然后将b赋值给a。

```java
public class Process {

	public void function3(int a) {
		a = 2;
	}

	public void function4(int a) {
		int b = 5;
		a = b;
	}
}

```
我们继续看测试类TestPrimitive

```java
public class TestPrimitive {

    public static void main(String[] args) {
        Process process = new Process();
        int age = 18;
        System.out.println(age);
        process.function3(age);
        System.out.println(age);
    }
}
```
结果是在经过function3的处理后，输出结果是

```
18
18
```
修改测试类代码，在经过function4处理后，仍然一致。
```
18
18
```
结论: 基本类型的传参，对传参进行修改，不影响原本参数的值。

### 对象类型传参
以下是处理类Porcess，function1，将参数car的颜色设置成blue。function2，新建了car2，将car2赋值给了参数car。

```java
public class Process {

	public void function1(Car car) {
		car.setColor("blue");
	}

	public void function2(Car car) {
		Car car2 = new Car("black");
		car = car2;
        car.setColor("orange");
	}
}
```
我们继续看测试类TestReference

```java
public class TestReference {

	public static void main(String[] args) {
		Process process = new Process();
        Car car = new Car("red");
		System.out.println(car);
		process.function1(car);
		System.out.println(car);
	}
}
```
结果是在经过function1的处理后，输出结果是

```
Car{color='red'}
Car{color='blue'}
```
修改测试类，在经过function2的处理后

```
Car{color='red'}
Car{color='red'}
```
结论: 对象类型的传参，直接调用传参set方法，可以对原本参数进行修改。如果修改传参的指向地址，调用传参的set方法，无法对原本参数的值进行修改。

**综上所述，基本类型的传参，在方法内部是值拷贝，有一个新的局部变量得到这个值，对这个局部变量的修改不影响原来的参数。对象类型的传参，传递的是堆上的地址，在方法内部是有一个新的局部变量得到引用地址的拷贝，对该局部变量的操作，影响的是同一块地址，因此原本的参数也会受影响，反之，若修改局部变量的引用地址，则不会对原本的参数产生任何可能的影响。**

-----

上文已经得到结论，我们从JVM的字节码的角度看一下过程是怎么样的。

首先大致JVM的基本结构，对基本类型，和对象存放的位置有一个大致的了解。下图是JVM的基本组件图。

![输入图片说明](https://static.oschina.net/uploads/img/201706/05210143_1NXx.jpg "在这里输入图片标题")

介绍几个基本的组件
1. 程序计数器: 存储每个线程下一步将执行的JVM指令。

2. JVM栈(JVM Stack): JVM栈是线程私有的，每个线程创建的同时都会创建JVM栈，JVM栈中存放的为当前线程中局部基本类型的变量（java中定义的八种基本类型：boolean、char、byte、short、int、long、float、double）、部分的返回结果以及Stack Frame(每个方法都会开辟一个自己的栈帧)，非基本类型的对象在JVM栈上仅存放一个指向堆上的地址

3. 堆(heap): JVM用来存储对象实例以及数组值的区域，可以认为Java中所有通过new创建的对象的内存都在此分配，Heap中的对象的内存需要等待GC进行回收。

4. 方法区（Method Area): 方法区域存放了所加载的类的信息（名称、修饰符等）、类中的静态变量、类中定义为final类型的常量、类中的Field信息、类中的方法信息，当开发人员在程序中通过Class对象中的getName、isInterface等方法来获取信息时，这些数据都来源于方法区域。

5. 本地方法栈（Native Method Stacks): JVM采用本地方法栈来支持native方法的执行，此区域用于存储每个native方法调用的状态。

6. 运行时常量池（Runtime Constant Pool): 存放的为类中的固定的常量信息、方法和Field的引用信息等，其空间从方法区域中分配。JVM在加载类时会为每个class分配一个独立的常量池，但是运行时常量池中的字符串常量池是全局共享的。

下图是从另一个角度解析JVM的结构，JVM是基于栈来操作的，每一个线程有自己的操作栈，遇到方法调用时会开辟栈帧，它含有自己的返回值，局部变量表，操作栈，以及对常量池的符号引用。
如果是基本类型，则存放在栈里的是值，如果是对象，存放在栈上是对象在堆上存放的地址。

![输入图片说明](https://static.oschina.net/uploads/img/201706/05210952_E5VW.png "在这里输入图片标题")

了解了JVM的基本结构，我们来看一下上述的两种代码，一种是基本类型传参，一种是对象传参，在字节码表现上的不同。
使用javap对字节码进行反编译

```
javap -verbose Main
```
### 基本类型传参字节码
以下是TestPrimitive类在执行function3时的字节码。

```java
public static void main(java.lang.String[]);
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=3, args_size=1
         0: new           #2                  // class Process
         3: dup           
         4: invokespecial #3                  // Method Process."<init>":()V
         7: astore_1      
         8: bipush        18
        10: istore_2      
        11: getstatic     #4                  // Field java/lang/System.out:Ljava/io/PrintStream;
        14: iload_2       
        15: invokevirtual #5                  // Method java/io/PrintStream.println:(I)V
        18: aload_1       
        19: iload_2       
        20: invokevirtual #6                  // Method Process.function3:(I)V
        23: getstatic     #4                  // Field java/lang/System.out:Ljava/io/PrintStream;
        26: iload_2       
        27: invokevirtual #5                  // Method java/io/PrintStream.println:(I)V
        30: return        
      LineNumberTable:
        ...........
      LocalVariableTable:
        Start  Length     Slot  Name          Signature
          0      31         0     args        [Ljava/lang/String;
          8      23         1    process      LProcess;
          11      20        2      age            I
```
主函数执行时，JVM操作栈会推入主函数栈帧，其中包含了主函数的局部变量表，字节码，返回值等信息。LocalVariableTable就是局部变量表,以0为索引起点，第0个是局部变量String数组 args,第1个是局部变量process，保存新创建的Process对象的引用地址。第2个是局部变量age。在字节码第8行，通过bipush 18，将常量18直接压入操作栈，然后第20行，是调用了process的function3方法，传入了age作为参数。
然后JVM操作栈将function3栈帧推入JVM栈，使得function3栈帧成为当前栈帧，开始执行。

```
public void function3(int);
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=2, args_size=2
         0: iconst_2      
         1: istore_1      
         2: return        
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
          0       3     0     this   LProcess;
          0       3     1       a      I
```
字节码显示，通过iconst_2，istore_1，将基本类型2推入栈，并保存在局部变量a中，这里就展示了我们在方法内部的修改都是对function3的局部变量a的值修改，不影响主函数中的a。从主函数的字节码中可以看到，它的值保存的还是第10行，通过istore_2保存到局部变量第2个索引处的18.

如果用图示来表示上述字节码执行过程中，JVM栈，man函数栈帧，function3栈帧内部变化的话，如下图所示。

1.主函数的栈帧会被推入JVM栈，成为当前操作栈。
![输入图片说明](https://static.oschina.net/uploads/img/201706/07203015_kGCX.jpg "在这里输入图片标题")

2.然后进去main函数栈帧，初始化完毕后如下图所示。
![输入图片说明](https://static.oschina.net/uploads/img/201706/07205207_oWfE.jpg "在这里输入图片标题")

3.主要看bipush 18，将基本变量18推入操作栈，基本变量类型是存储在栈帧内部的。

![输入图片说明](https://static.oschina.net/uploads/img/201706/07210405_XMBH.jpg "在这里输入图片标题")

4.然后执行istore_2, 将栈顶出栈，并且保存在局部变量索引2处。
![](https://static.oschina.net/uploads/img/201706/07210458_Xv29.jpg "在这里输入图片标题")

5.然继续执行至18: aload_1,，将创建的process的地址保存在局部变量索引1处,19:iload_2，将局部变量2处保存的基本类型压入栈。

![输入图片说明](https://static.oschina.net/uploads/img/201706/07210535_WtXK.jpg "在这里输入图片标题")

6.然后执行至20：invokevirtula #6,也就是调用function3，进入function3的栈帧。执行0: iconst_2，将常量2推入栈，此时function3的栈帧有一个局部变量1处保存着传入的参数18。

![输入图片说明](https://static.oschina.net/uploads/img/201706/07222015_K00B.jpg "在这里输入图片标题")

7.继续执行1:istore_1，将栈顶推出，保存在局部变量1处，覆盖了传入的参数18，然后return，将function3函数栈帧弹出JVM栈，继续执行main函数栈帧。
![](https://static.oschina.net/uploads/img/201706/07212133_gGyh.jpg "在这里输入图片标题")

**之后会继续执行main函数栈帧，在function3函数栈帧中发生的一切都和Main Stack中的局部变量age的值没有任何关系。**

### 对象类型传参字节码
以下是TestReference类在执行function2时的字节码。

```
Code:
      stack=3, locals=3, args_size=1
         0: new           #2                  // class Process
         3: dup           
         4: invokespecial #3                  // Method Process."<init>":()V
         7: astore_1      
         8: new           #4                  // class Car
        11: dup           
        12: ldc           #5                  // String red
        14: invokespecial #6                  // Method Car."<init>":(Ljava/lang/String;)V
        17: astore_2      
        18: getstatic     #7                  // Field java/lang/System.out:Ljava/io/PrintStream;
        21: aload_2       
        22: invokevirtual #8                  // Method java/io/PrintStream.println:(Ljava/lang/Object;)V
        25: aload_1       
        26: aload_2       
        27: invokevirtual #9                  // Method Process.function2:(LCar;)V
        30: getstatic     #7                  // Field java/lang/System.out:Ljava/io/PrintStream;
        33: aload_2       
        34: invokevirtual #8                  // Method java/io/PrintStream.println:(Ljava/lang/Object;)V
        37: return        
      LocalVariableTable:
        Start  Length  Slot       Name         Signature
          0      38         0     args         [Ljava/lang/String;
          8      30         1     process      LProcess;
          18      20        2     car          LCar;

```
我们可以通过字节码14-17行，看到局部变量索引2处存放的是Car的实例在堆上的地址，这和基本类型不同，基本类型的值都是直接存放在栈里面的。然后通过字节码第27行将car的引用地址传入function2。接下来我们看看function2的字节码。

```
public void function2(Car);
    flags: ACC_PUBLIC
    Code:
      stack=3, locals=3, args_size=2
         0: new           #4                  // class Car
         3: dup           
         4: ldc           #5                  // String black
         6: invokespecial #6                  // Method Car."<init>":(Ljava/lang/String;)V
         9: astore_2      
        10: aload_2       
        11: astore_1     
        12: aload_1       
        13: ldc           #7                  // String orange
        15: invokevirtual #3                  // Method Car.setColor:(Ljava/lang/String;)V
        18: return     
      LocalVariableTable:
        Start  Length  Slot  Name       Signature
          0      13     0          this   LProcess;
          0      13     1           car   LCar;
         10       3     2          car2   LCar;

```
题外话，因为这个是调用具体实例的函数，所以索引0处保存的是实例的引用。索引1保存的是传参car的引用地址，car2保存的是函数内创建的Car实例的地址。字节码0-9，完成了car2的引用地址保存，第10行将Car2的引用地址推入栈，第11行通过astore_1,将栈顶值保存到第一个局部变量，也就是修改了覆盖了局部变量car的引用地址。因此第15行，修改的是car当前引用的地址的实例的参数值。当退出栈帧，回到主函数，主函数的局部变量a保存的引用地址没有改变。

如果用图示来表示上述字节码执行过程中，JVM栈，man函数栈帧，function3栈帧内部变化的话，如下图所示。

1.main函数栈帧和上文测试基本类型传参时的字节码大致类似，不同的是局部变量处。局部变量2处保存的是main函数中新建的Car实例的堆上地址。对象的实际存放都是在堆中，栈帧的局部变量中保存的是他们在堆上的地址。

![输入图片说明](https://static.oschina.net/uploads/img/201706/07222348_OQFZ.jpg "在这里输入图片标题")

2.一直执行到调用function2，进入function2栈帧。在执行至9:astore_2时，栈中新创建的Car实例的引用地址出栈，保存在局部变量2处。局部变量1保存的是传参进来的Car实例的引用地址。

![输入图片说明](https://static.oschina.net/uploads/img/201706/07221727_vbtS.jpg "在这里输入图片标题")

3.然后执行至10: aload_2,11:store_1，在这里，1236df被推入栈，然后保存在了局部变量1，覆盖了局部变量car本来的引用地址。

![输入图片说明](https://static.oschina.net/uploads/img/201706/07221915_q75B.jpg "在这里输入图片标题")

**
因此，当function2对局部变量2进行相关操作时，影响的都是1236df这块地址，和main函数局部变量car中保存的1235df不是一块地址，所以前后打印结果一致。**

测试类TestReference调用function1时，function1没有改变局部变量car的引用地址，保存的仍然是传入的引用地址，所以function1中car进行的操作影响了这块地址保存的内容，导致了前后打印结果不一致。

```
Code:
      stack=2, locals=2, args_size=2
         0: aload_1       
         1: ldc           #2                  // String blue
         3: invokevirtual #3                  // Method Car.setColor:(Ljava/lang/String;)V
         6: return        
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
         0       7       0    this    LProcess;
         0       7       1     car    LCar;

```

**本文对Java基本类型传参和对象传参，从字节码角度进行了分析，现在不会再搞错了吧~**

----
![输入图片说明](https://static.oschina.net/uploads/img/201706/09172710_vHZ0.png "在这里输入图片标题")