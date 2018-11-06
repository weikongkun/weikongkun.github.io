---
title: Java堆外内存
date: 2018-09-28 19:43:57
tags: JVM
category: JVM
---

# 为什么使用堆外内存

- 来源于TaobaoJVM对OpenJDK定制开发的GCIH部分，其中GCIH就是将CMS Old Heap区的一部分划分出来，这部分内存虽然还在堆中，但是已经不被GC所管理，将长生命周期Java对象放在Java堆外，GC不能管理GCIH内Java对象。

- 将长期存活的对象移入堆外内存，从而减少CMS管理的对象数量，降低Full GC的次数和频率，达到提高系统响应速度的目的。
- 加快复制的速度：堆内在缓存到远程时，会先复制到直接内存，然后再发送，而堆外内存相当于省略了这个工作。
- 这部分内容可以进程间共享，这样一台Server就都可以跑更多的VM实例。

# 堆外内存的使用

JDK1.4新加入NIO，引入了一种基于Channel与Buffer的I/O方式，可以使用Native函数库直接分配堆外内存。

由JDK提供的