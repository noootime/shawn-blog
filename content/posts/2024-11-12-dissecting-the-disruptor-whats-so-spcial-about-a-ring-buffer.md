---
title: "[译] Dissecting the Disruptor: What’s so special about a ring buffer?"
date: 2024-12-23T15:51:23+08:00
slug: 2024-12-23-2024-11-12-dissecting-the-disruptor-whats-so-spcial-about-a-ring-buffer
type: posts
draft: false
categories:
  - default
tags:
  - 翻译
  - Disruptor
---

| 本文翻译自[Dissecting the Disruptor: What’s so special about a ring buffer?](https://trishagee.com/2011/06/22/dissecting_the_disruptor_whats_so_special_about_a_ring_buffer/)

最近我们开源了`LMAX Disruptor`，它是让我们的数据交互变得如此之快的关键。至于我们为什么要开源**Distruptor**技术，是因为我们意识到传统的高性能编程可能存在特别离谱的错误。我们设计了一种更好、更快的线程间共享数据的方式，迫不及待与大家分享。这会让我们看起来顶呱呱！

在[这个网站](https://lmax-exchange.github.io/disruptor/files/Disruptor-1.0.pdf)你可以下载一篇技术文章，里面详细介绍了**Disruptor**是什么以及它好在哪里。

然而，一下子消化所有内容有点困难，所以我会把它分成小块并用更加通俗易懂的方式进行解释，这对于那些`NADD(Nerd Attention Deficiency Disorder, 泛指注意力容易分散的人)`的读者更加友好。

首先，我们要谈一下`Ring Buffer`。**Disruptor**不单单是一个`Ring Buffer`，但是这种数据结构确实是**Disruptor**的核心，此外**Disruptor**的聪明之处在于如何控制对`Ring Buffer`的访问。

## Ring Buffer究竟是什么？

`Ring Buffer`，即环形缓冲区，就像它的名字一样，它就是一个“环”（圆形的并且可以循环）。你可以用它作为缓冲区，将数据从一个上下文（一个线程）传递到另一个上下文（另一个线程）：

![alt RingBuffer](https://trishagee.com/wp-content/uploads/2020/10/RingBuffer.png)

简单来说，它就是一个带有指向下一个可用槽指针的数组：

![alt](https://trishagee.com/wp-content/uploads/2020/10/RingBufferInitial.png)

当你不断向这个环形数组中写入数据时，序号会持续增加：

![alt](https://trishagee.com/wp-content/uploads/2020/10/RingBufferWrapped.png)

如果需要找到下一个序号在数组中的位置，可以通过`mod`操作：

```
${sequence} mod ${array length} = ${array index}
```

如果用Java来计算上图中表示的12的下标，为：`12 % 10 = 2`

## So what?

如果你参考Wikipedia中关于[Circular Buffers](http://en.wikipedia.org/wiki/Circular_buffer)的讲解，你会发现它与我们的实现中一个关键区别是在数组的end位置我们没有定义指针指向它。我们只定义了下一个可用位置的下标。这是经过深思熟虑的，我们选择`Ring Buffer`的最主要因素是为了支持**消息的可靠性传输**。我们需要一个用来存储服务端发送的消息记录，以便当另外一个服务（客户端）发送一个[nak](http://en.wikipedia.org/wiki/Nak)说它没有收到消息时，能够重新发送它。

`Ring Buffer`看起来很适合做这个事情，它存储的序列能够展示缓冲区的结束位置是哪里，并且如果接受到`NAK`需要重放数据时，他能够在任意位置重放数据直到当前序列的结尾处：

![alt](https://trishagee.com/wp-content/uploads/2020/10/RingBufferReplay.png)

我们自己实现的`Ring Buffer`与传统使用的队列之间的区别是，我们不会消费缓冲区中的元素 —— 它们会一直存在，直到这块区域的数据被重写。这就是为什么我们不需要一个指向"end"位置的指针。同时，对于`Ring Buffer`的数据操作是否进行“回绕”处理（当到达边界后回到初始位置），不由这个数据结构本身管理，而是由数据生产者和数据消费者根据具体的场景自行处理。

## And it's so great because...?

这种数据结构除了能够提供良好的消息可靠性，它还有很多其它不错的特性。

首先，因为它是数组，速度上就会比其它数据结构更快，比如相比链表。从硬件层面来看，数组中的元素会被预加载到CPU缓存中，CPU并不会频繁的回到主存去加载下一个数据。

其次，基于数组实现的特性，保证了你需要对它的内存空间进行预分配，只要在使用过程中这个数据结构没有超出其预先分配的内存空间，那么其中存放的对象就会相对稳定的存储在于这片内存区域中，不需要频繁得进行内存的重新分配、释放等操作，从内存管理的角度来说是比较稳定的存在形式。同时，垃圾回收器在面对这种已经预先分配好、内部元素稳定存在的数据结构时，不需要频繁介入去判断哪些内存空间可以回收，哪些对象已经不再被使用等情况，减少了垃圾回收器的工作负担。

作为对比，在链表中，每当有一个新的数据项添加到链表时，它通常需要创建相应的对象来存储这个数据项以及维护和链表中其他节点之间的链接关系等。例如，向链表中插入一个新的节点，就需要分配内存来创建这个新节点对象。而当链表中的某个数据项不再需要，也就是要从链表中移除这个节点时，相应的对象就需要被清理掉，释放其所占用的内存空间，以便系统内存可以被高效利用。这与上面提到的数组形式的数据结构在内存管理上形成的鲜明的对比，链表在内存管理上因为这种动态的创建和释放对象的操作，需要更频繁地依赖垃圾收集机制来回收不再使用的内存空间。

## The missing pieces

有很多细节在本文其实没有体现，比如如何处理环形缓冲区的“回绕”，或是如何进行读写。

当你将**Disruptor**与一个类似实现的队列进行比较时，会发现一些有趣的事情。队列通常会关心所有的属性例如队列的start和end，如何添加元素和消费元素等。之所以关于`Ring Buffer`对于上面的这些因素没有在本文讨论，是因为它们并不是`Ring Buffer`的职责，这些内容需要放在这个数据结构之外进行讨论。

你可以通过[阅读论文](https://lmax-exchange.github.io/disruptor/files/Disruptor-1.0.pdf)或[查看源代码](https://github.com/LMAX-Exchange/disruptor/)来了解更多详情。或者看一下我的[其它文章](https://github.com/LMAX-Exchange/disruptor/)

## Core Concept

**NAK**: 在计算机网络通讯中，数据接收方通过回报数据接收状态（`ACK`或`NAK/NACK`）来告知发送方，这样，发送方就可以根据不同状态进行相对应的处理。其中，`ACK`表示确认，`NAK`或`NACK`表示否定确认。

