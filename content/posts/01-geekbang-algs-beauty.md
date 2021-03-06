---
title: "极客时间《数据结构与算法之美》学习笔记"
date: 2018-10-08T19:41:00+08:00
description: "本篇文章记录记录在学习《数据结构与算法之美》专栏文章时总结的知识点和问题，方便日后回顾。"
draft: false
tags: [Algs]
---

>为什么要学习算法？建立时空复杂度意识、写出高性能的代码，能够设计基础架构，提升编程技能，训练逻辑思维。

数据结构就是一组数据的存储方式，而算法则是操作数据的一组方法，比如队列、栈、堆、二分查找、动态规划等。

算法学习重点：
- 来历
- 自身的特点
- 适合解决的问题
- 实际的应用场景

## 复杂度分析

复杂度分析不依赖具体的测试数据来测试，可以粗略统计执行效率，排除测试数据规模和测试环境对结果的影响。

### 大 O 复杂度表示法

所有代码的执行时间 T(n) 与每行代码的执行次数成正比。

大 O 时间复杂度表示法：T(n) = O(f(n))，并不表示真正的执行时间，而是表示代码执行时间对数据规模增长的变化趋势，也叫做渐进时间复杂度。公式中的低阶、常量、系数并不影响增长趋势，可以忽略，只记录最大量级即可，如 T(n) = O(n^2)。

时间复杂度分析：
- 只关注执行次数最多的一段代码
- 加法法则：总复杂度等于量级最大的那段代码的复杂度，如果两段代码的量级相当，则加法法则不适用
- 乘法法则：嵌套代码的复杂度等于嵌套内外代码复杂度的乘积

![复杂度量级](https://static001.geekbang.org/resource/image/37/0a/3723793cc5c810e9d5b06bc95325bf0a.jpg)

非多项式量级的算法问题叫做 NP 问题（ O(2^n) 和 O(n!) ），是非常低效的算法。

空间复杂度：表示算法的存储空间与数据规模之间的增长关系。

### 常见时间复杂度

同一段代码，在不同的输入情况下，复杂度量级有可能是不一样的。

1. 最好、最坏情况时间复杂度  
最理想情况下或最糟糕的情况下，执行一段代码的时间复杂度。

2. 平均情况时间复杂度  
可以简单使用每种情况下执行次数综合除以情况总数来计算得出，但是需要考虑到每种情况出现的概率，即加权平均时间复杂度或者期望时间复杂度。需要用到[概率知识](https://www.shuxuele.com/data/probability.html)

3. 均摊时间复杂度  
对一个数据结构进行一组连续操作中，大部分情况下时间复杂度都很低，只有个别情况下时间复杂度比较高，而且这些操作之间存在前后连贯的时序关系，这个时候，我们就可以将这一组操作放在一块儿分析，看是否能将较高时间复杂度那次操作的耗时，平摊到其他那些时间复杂度比较低的操作上。而且，在能够应用均摊时间复杂度分析的场合，一般均摊时间复杂度就等于最好情况时间复杂度。


## 数组

数组（Array）是一种线性表数据结构。它用一组连续的内存空间，来存储一组具有相同类型的数据。

如何实现根据下标随机访问数组元素？计算机会给每个内存单元分配一个地址，然后通过地址来访问内存中的数据。需要随机访问数组中的某个元素时，通过下面的寻址公式，计算出元素的内存地址：
```js
a[i]_address = base_address + i * data_type_size
```

数组和链表的区别：链表适合插入、删除，时间复杂度 O(1); 数组支持随机访问，根据下标随机访问的时间复杂度为 O(1)，而查找的时间复杂度并不为 O(1), 即使是有序数组通过二分查找，时间复杂度也是 O(logn). 

**插入操作：** 需要挪动位置，平均时间复杂度为 O(n). 如果数组不需要保持有序，则可以将被插入位置的元素挪动到末尾，然后在该位置插入新元素，时间复杂度 O(1). 

**删除操作：** 需要挪动位置，平均时间复杂度为 O(n). 可以将多次删除操作集中在一起执行，每次删除操作并不是真正地搬移数据，只是记录数据已经被删除，当数组空间不足时再出发真正的删除操作。

## 链表

链表的经典应用场景：LRU 缓存淘汰算法。常见的缓存策略：  
- 先进先出策略 FIFO
- 最少使用策略 LFU
- 最近最少使用策略 LRU(Lease Recently Used)

数组 VS 链表：
- 数组需要一块连续的内存空间来存储，当内存中没有连续的、足够大的存储空间时创建数组会失败
- 链表通过“指针”将一组零散的内存块串联起来使用

常见链表结构：
- 单链表
- 双向链表
- 循环链表

### 单链表

![单链表](https://static001.geekbang.org/resource/image/b9/eb/b93e7ade9bb927baad1348d9a806ddeb.jpg)

**链表的插入和删除操作：** 只需要考虑相邻结点的指针改变，时间复杂度是 O(1).

**随机访问：** 需要根据指针一个结点一个结点地依次遍历，直到找到相应的节点。时间复杂度 O(n). 

### 双向链表

双向链表支持两个方向，每个结点不止有一格后继指针 next 指向后面的结点，还有一个钱去指针 prev 指向前面的结点。

支持 O(1) 的时间复杂度的情况下找到前驱结点。

**删除操作：** 
两种情况：1. 删除结点中“值等于某个给定值”的结点；2. 删除给定指针指向的结点。

第一种情况，无论是单链表还是双向链表，为了查到到给定值的结点，都需要从头结点开始一个一个依次遍历对比，直到找到然后通过指针操作将其删除。虽然删除操作时间复杂度是 O(1), 但是查找操作的时间复杂度是 O(n), 加法法则得出总的时间复杂度是 O(n).

第二种情况，已知要删除的结点，但是删除指定结点需要知道其前驱结点，如果是单链表，依然需要从头遍历；而如果是双向链表，可以直接通过前驱指针获取前驱结点。时间复杂度为 O(1).

**抽入操作：** 插入操作同理，单链表需要通过遍历找到前驱结点，而双向链表直接通过前驱指针获取。

**查找操作：** 如果是有序链表。通过记录上次查找的位置，再次查找时，如果是双向链表，可以通过值的比较决定是往前查找还是往后查找，平均只只需要查找一半的数据。

### 循环链表

循环链表的尾结点指针指向链表的头结点。优点是从链尾到链头比较方便。