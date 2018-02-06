---
layout: post
title: "JVM内存简介"
date: 2015-12-31 16:01:00 +0800
comments: true
tags: Java
keywords: Java, JVM, 内存, Java虚拟机
---
在开发中，经常会碰到内存溢出等异常，这篇文章来了解一下Java的内存机制，以及简单的配置参数。

## Java虚拟机内存组成
Java虚拟机的内存包含以下两个部分：

* 堆内存（Heap），这个里面存放了所有的类实例和数组，Java代码中所有的new操作产生的对象都存在这里
* 非堆内存（Non-Heap），这个地方被Java自己使用，方法区，JVM内部处理优化所需的内存，每个类结构以及方法和构造方法
* Java本地内存，这个地方存放了Java运行时自身所必须的一些东西。

<!--more-->

![](http://ww1.sinaimg.cn/large/4a77b961jw1ezj0g4abmtj20sb0br765.jpg)

## 堆内存
![](http://ww1.sinaimg.cn/large/4a77b961jw1ezj17jennrj20hs052gm8.jpg)

堆内存由新生代（Young Generation）和老生代（Old Generation）两个部分。其中新生代又包含Eden和两个存活带（Survivor Space）。

* Eden区域存放新生的对象
* Survivor Space有两个，From和To，存放每次垃圾回收之后存活的对象

老生代主要存放应用程序中生命周期长的存活对象

## 年轻代GC和内存分配
HotSpot JVM把年轻代分为了三部分：1个Eden区和2个Survivor区（分别叫from和to）。默认比例为8：1，接下来我们会聊到。一般情况下，新创建的对象都会被分配到Eden区(一些大对象特殊处理),这些对象经过第一次GC后，如果仍然存活，将会被移到Survivor区。对象在Survivor区中每熬过一次Minor GC，年龄就会增加1岁，当它的年龄增加到一定程度时，就会被移动到年老代中。

在GC开始的时候，对象只会存在于Eden区和名为“From”的Survivor区，Survivor区“To”是空的。紧接着进行GC，Eden区中所有存活的对象都会被复制到“To”，而在“From”区中，仍存活的对象会根据他们的年龄值来决定去向。年龄达到一定值(年龄阈值，可以通过-XX:MaxTenuringThreshold来设置)的对象会被移动到年老代中，没有达到阈值的对象会被复制到“To”区域。经过这次GC后，Eden区和From区已经被清空。这个时候，“From”和“To”会交换他们的角色，也就是新的“To”就是上次GC前的“From”，新的“From”就是上次GC前的“To”。不管怎样，都会保证名为To的Survivor区域是空的。Minor GC会一直重复这样的过程，直到“To”区被填满，“To”区被填满之后，会将所有对象移动到年老代中。

![](http://ww1.sinaimg.cn/large/4a77b961jw1ezixsbwnvhj20n109lgmf.jpg)

上图演示GC过程，黄色表示死对象，绿色表示剩余空间，红色表示幸存对象

我是一个普通的java对象，我出生在Eden区，在Eden区我还看到和我长的很像的小兄弟，我们在Eden区中玩了挺长时间。有一天Eden区中的人实在是太多了，我就被迫去了Survivor区的“From”区，自从去了Survivor区，我就开始漂了，有时候在Survivor的“From”区，有时候在Survivor的“To”区，居无定所。直到我18岁的时候，爸爸说我成人了，该去社会上闯闯了。于是我就去了年老代那边，年老代里，人很多，并且年龄都挺大的，我在这里也认识了很多人。在年老代里，我生活了20年(每次GC加一岁)，然后被回收。

参数：

* -Xms	堆的最小值
* -Xmx	堆的最大值
* -XX:MaxNewSize	年轻代的最大内存大小
* -XX:NewSize	年轻代初始化内存大小
* -XX:NewRatio	设置新生代和老生代的相对大小，例如 -XX:NewRatio=3 指定老年代/新生代为3/1. 老年代占堆大小的3/4，新生代占1/4
* -XX:SurvivorRatio	设置Eden区和Survivor的比例大小，-XX:SurvivorRatio=10 表示Eden是幸存区To大小的10倍(也是幸存区From的10倍).所以,伊甸园区(Eden)占新生代大小的10/12,幸存区From和幸存区To每个占新生代的1/12.注意,两个幸存区永远是一样大的
* -XX:+PrintTenuringDistribution	指定JVM 在每次新生代GC时，输出幸存区中对象的年龄分布
* -XX:InitialTenuringThreshold	老生代阈值初始大小
* -XX:MaxTenuringThreshold	老生代阈值最大值
* -XX:TargetSurvivorRatio	幸存区目标使用率，如果超出使用率，会重新设置老生阈值，否则以最大值为准。

## 非堆内存
非堆内存也被成为方法区，永久代。方法区是JVM中所有线程共享的内存区，存储已被虚拟机加载的类信息、常量、静态变量、即时编译后的代码等数据。

参数：

* -XX:PermSize	堆内存初始值
* -XX:MaxPermSize	堆内存最大值

##参考
<http://www.cnblogs.com/redcreen/archive/2011/05/04/2036387.html>

<http://www.cnblogs.com/zhguang/p/java-jvm-gc.html>

<http://ifeve.com/jvm-yong-generation/>

<http://m.oschina.net/blog/55010>

<http://mzorro.me/2015/08/26/jvm-memory-and-gc/>

<http://www.pointsoftware.ch/en/cookbook-jvm-memory-arguments/>

<https://www.yourkit.com/docs/kb/sizes.jsp>