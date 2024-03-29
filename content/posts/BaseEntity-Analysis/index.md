---
title: "某基类的源码分析"
date: "2023-05-10"
description: "浅浅分析一下"
tags: ["java"]
ShowToc: false
ShowBreadCrumbs: true
---

开发中发现许多对象都依赖了css的基类对象`BaseEntity`
偶然点开发现里面的写法不太容易理解，因此分析记录一下


![](images/Pasted%20image%2020230512175229.png)
点开对象可以发现，有两个属性是比较特别的，也就是 propMap 和 visitor
本文的目的也主要是分析这两个对象的用途。


![](images/Pasted%20image%2020230505092032.png)

先直接看这两个对象在哪用了，翻阅源码可以发现，两个属性的主要使用入口都在`toString`中。
最终这个函数是调用了`toString0` 或者 `hexString`
初见这一坨，我也百思不得其解，先不管，继续看看`toString0`
![](images/Pasted%20image%2020230505103131.png)


分析代码可知，`toString0`这个方法才是对象转字符串的主要实现，通过`getProps`获取对象的属性列表，然后在下面进行属性的拼接
![](images/Pasted%20image%2020230505103619.png)

看看`getProps`，可以发现propMap的用途是存储该类的所有属性名称。
如果该对象的属性未初始化，则通过反射获取。
![](images/Pasted%20image%2020230512180112.png)
需要注意的是，这里使用了 double check lock，避免重入。
当第二次进入时，就可以复用属性，不用再反射获取了。
到了此处，propMap 的用途基本就解答了。


那么为啥要有 visotor 呢？我们再回过头来看`toString`方法
![](images/Pasted%20image%2020230512173325.png)

在`toString`方法中，visitor 的逻辑主要如下
1. 如果 visitor为空，重新初始化 visitor
2. 如果 visitor.get() 为空：
		1. 设置 visitor 为当前对象
		2. 调用 toString0
		3. 设置 visitor 为 null
3. 如果 visitor 的值不为空，返回 HexString

这里的目的不太好理解。首先对于步骤1，我们可以先忽略，因为实际不影响流程。
控制流程的主要是步骤2和步骤3。
首先可以观察到，步骤2的进入条件是 visitor 为空，步骤3的进入条件为 visitor 不为空。也就是说，步骤2和步骤3这两步操作是互斥的。
在步骤2.1中，设置了 visitor 为当前对象，然后在2.3中又重置了回去。那么也就是说同一个线程，他不能重入步骤2。

这里比较有迷惑性，因为threadLocal是线程独立的，搞这么复杂干嘛呢，同一个线程还能重入这个方法不成？
读者可以先思考一下，表情包下面给出答案。

![](images/Pasted%20image%2020230512175120.png)

同一个线程还能重入这个方法不成？确实是可能的。
答案就是当对象发生了循环引用，这时调用`toString`方法，序列化子对象时，由于循环引用，就会再次进入父对象的`toString`方法，不断套娃，最终导致栈溢出。
所以此处使用visitor进行流程控制的目的就是为了打破这个循环调用链。

![](images/Pasted%20image%2020230512174950.png)


实际上，许多框架都支持了循环引用，不过方式上可能有些许差异，比如 FastJson 使用 $ref 占位符避免序列化子对象，Spring 使用了多级缓存的方式等等，这里就不赘述了。

框架开发人员总是考虑到各种场景进行了兼容，不过我认为大部分循环引用，其实并不值得解决，因为其代表是层级结构本身的不合理。