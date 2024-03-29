---
title: Go 垃圾回收介绍
tags:
  - go
cover: 'http://images.ashery.cn/img/P1100340.JPG'
abbrlink: 8c5a54e3
date: 2023-03-12 15:07:10
categories:
description:
---


## 什么是垃圾回收机制

### 从内存分配机制讲起

程序中的数据和变量都会被分配到程序所在的虚拟内存中，内存空间包含两个重要区域：**栈区（Stack）和堆区（Heap）**。（当然还有堆外内存，不在这里讨论）

函数调用的参数、返回值以及局部变量大都会被分配到**栈**上，这部分内存会由编译器进行管理。

而不同编程语言使用不同的方法管理**堆**区的内存，C++ 等编程语言会由工程师**主动申请和释放**内存，Go 以及 Java 等编程语言会由工程师和编译器共同管理，**堆中的对象由内存分配器分配并由垃圾收集器回收。**



## Go的垃圾回收机制

Java中有很多垃圾回收的算法和垃圾回收器，比如Parallel Scavenge、CMS、G1等，而在Go中只有一种，与CMS类似，都是使用的标记-清除的算法，并在此基础上优化了性能。

### 标记-清除算法

标记清除（Mark-Sweep）算法是最常见的垃圾收集算法，标记清除收集器是跟踪式垃圾收集器，其执行过程可以分成标记（Mark）和清除（Sweep）两个阶段：

1. 标记阶段： 从根对象出发查找并标记堆中所有存活的对象
2. 清除阶段： 遍历堆中的全部对象，回收未被标记的垃圾对象并将回收的内存加入空闲链表

这里有几个问题：

1. **如何识别垃圾**：主要有两种方式，
   1. **引用计数**：为对象添加一个引用计数器，当对象增加一个引用时计数器加 1，引用失效时计数器减 1。引用计数为 0 的对象可被回收。这里存在一个问题：在两个对象出现循环引用的情况下，此时引用计数器永远不为 0，导致无法对它们进行回收。
   2. **可达性分析**：以 GC Roots 为起始点进行搜索，可达的对象都是存活的，不可达的对象可被回收。Go里面的GC Roots一般包括：
      1. **全局变量**：程序在编译期就能确定的那些存在于程序整个生命周期的变量。
      2. **执行栈**：每个 goroutine 都包含自己的执行栈，这些执行栈上包含栈上的变量及指向分配的堆内存区块的指针。
      3. **寄存器**：寄存器的值可能表示一个指针，参与计算的这些指针可能指向某些赋值器分配的堆内存区块。
2. **性能问题**：在传统的标记清除算法中，清除阶段，垃圾收集器会依次遍历堆中的对象并清除其中的垃圾，整个过程需要标记对象的存活状态，用户程序在垃圾收集的过程中也不能执行（不然会改变对象的存活状态，导致垃圾回收出现错误），我们需要用到更复杂的机制来优化算法，尽可能减小 STW （Stop The World）的问题。

### 算法优化：三色标记法

为了解决原始标记清除算法带来的长时间 STW，多数现代的追踪式垃圾收集器都会实现三色标记算法的变种以缩短 STW 的时间。三色标记算法将程序中的对象分成白色、黑色和灰色三类[4](https://draveness.me/golang/docs/part3-runtime/ch07-memory/golang-garbage-collector/#fn:4)：

- 白色对象： 潜在的垃圾，其内存可能会被垃圾收集器回收；
- 黑色对象： 活跃的对象，包括不存在任何引用外部指针的对象以及从根对象可达的对象；
- 灰色对象： 活跃的对象，因为存在指向白色对象的外部指针，垃圾收集器会扫描这些对象的子对象；

三色标记垃圾收集器的工作原理可以将其归纳成以下几个步骤：

1. 从灰色对象的集合中选择一个灰色对象并将其标记成黑色；
2. 将黑色对象指向的所有对象都标记成灰色，保证该对象和被该对象引用的对象都不会被回收；
3. 重复上述两个步骤直到对象图中不存在灰色对象

![](http://images.ashery.cn/img/2020-03-16-15843705141814-tri-color-mark-sweep.png)

当三色的标记清除的标记阶段结束之后，应用程序的堆中就不存在任何的灰色对象，我们只能看到黑色的存活对象以及白色的垃圾对象，垃圾收集器可以回收这些白色的垃圾

因为用户程序可能在标记执行的过程中修改对象的指针，**所以三色标记清除算法本身是不可以并发或者增量执行的，它仍然需要 STW**。

这时就需要用到**内存屏障**来让垃圾回收器和用户程序并发运行，并且保证三色标记的正确性。

### 内存屏障

内存屏障技术是一种屏障指令，它可以让 CPU 或者编译器在执行内存相关操作时遵循特定的约束。

目前多数的现代处理器都会乱序执行指令以最大化性能，但是该技术能够保证内存操作的顺序性，在内存屏障前执行的操作一定会先于内存屏障后执行的操作。

想要在并发或者增量的标记算法中保证正确性，我们需要达成以下两种三色不变性中的一种：

- 强三色不变性 — 黑色对象不会指向白色对象，只会指向灰色对象或者黑色对象；
- 弱三色不变性 — 黑色对象指向的白色对象必须包含一条从灰色对象经由多个白色对象的可达路径；

**插入写屏障**

Dijkstra 在 1978 年提出了插入写屏障：在写入前，对指针要指向的对象进行灰色着色

```go
writePointer(slot, ptr):
    shade(ptr)
    *slot = ptr
```

每当执行类似 `*slot = ptr` 的指针赋值表达式时，如果 `ptr` 指针是白色的，那么`shade`会将该对象设置成灰色，其他情况则保持不变。

![](http://images.ashery.cn/img/71841831-90d5b480-30c0-11ea-9c87-7ac2571546e3.png)

Dijkstra 的插入写屏障是一种相对保守的屏障技术，它会将**有存活可能的对象都标记成灰色**以满足强三色不变性。

插入式的 Dijkstra 写屏障虽然实现非常简单并且也能保证强三色不变性，但是它也有明显的缺点。因为栈上的对象在垃圾收集中也会被认为是根对象，所以为了保证内存的安全，Dijkstra 必须**为栈上的对象增加写屏障**或者在标**记阶段完成重新对栈上的对象进行扫描**，这两种方法各有各的缺点，前者会大幅度增加写入指针的额外开销，后者重新扫描栈对象时需要暂停程序，垃圾收集算法的设计者需要在这两者之间做出权衡。（Go选择了后者）

**删除写屏障**

Yuasa 在 1990 年的论文 Real-time garbage collection on general-purpose machines 中提出了删除写屏障：在写入前，对指针所在的对象进行灰色着色

```go
writePointer(slot, ptr)
    shade(*slot)
    *slot = ptr
```

<img src="https://golang.design/under-the-hood/assets/gc-wb-yuasa.png" style="zoom: 10%;" />

Yuasa 删除屏障的优势则在于不需要标记结束阶段的重新扫描， 结束时候能够准确的回收所有需要回收的白色对象。 缺陷是 Yuasa 删除屏障会拦截写操作，进而导致波面的退后（C由黑色变成了灰色），产生冗余的扫描，如图所示C被扫描了2次。



**混合写屏障**

Go 语言在 v1.8 组合 Dijkstra 插入写屏障和 Yuasa 删除写屏障构成了如下所示的**混合写屏障**，该写屏障会**将被覆盖的对象标记成灰色并在当前栈没有完成扫描时将新对象也标记成灰色**

```
// 混合写屏障
func writePointer(slot, ptr unsafe.Pointer) {
    shade(*slot)                // 对正在被覆盖的对象进行着色
    if current stack is grey {  // 如果当前 goroutine 栈还未被扫描为黑色
        shade(ptr)              // 则对引用进行着色
    }
    *slot = ptr
}
```

除了引入混合写屏障之外，在垃圾收集的标记阶段，我们还需要**将创建的所有新对象都标记成黑色**，防止新分配的栈内存和堆内存中的对象被错误地回收，因为栈内存在标记阶段最终都会变为黑色，所以不再需要重新扫描栈空间。

## Go垃圾回收器的具体实现

太难了，看不懂，埋个坑下次再写（o´・ェ・｀o）

## Go GC调优

Go 可供用户调整的参数只有 GOGC 这一个环境变量。他控制的是触发GC的阈值，是一个百分比。当Go新创建的对象所占用的内存大小，除以上次GC结束后保留下来的对象占用内存大小，所得到的比值大于`GOGC`设置的阈值时，就会触发一次GC。如果没有指定这个变量，则默认值是100，即当内存增长到上一个GC结束后剩余内存的两倍时触发GC。

`hard_target = live_dataset + live_dataset * (GOGC / 100)`

GC 的调优是在特定场景下产生的，并非所有程序都需要针对 GC 进行调优。只有那些对执行延迟非常敏感、当 GC 的开销成为程序性能瓶颈的程序，才需要针对 GC 进行性能调优，几乎不存在于实际开发中 99% 的情况。除此之外，Go 的 GC 也仍然有一定的可改进的空间，也有部分 GC 造成的问题，目前仍属于 Open Problem。

总的来说，我们可以在现在的开发中处理的有以下几种情况：

1. 对停顿敏感：GC 过程中产生的长时间停顿、或由于需要执行 GC 而没有执行用户代码，导致需要立即执行的用户代码执行滞后。
2. 对资源消耗敏感：对于频繁分配内存的应用而言，频繁分配内存增加 GC 的工作量，原本可以充分利用 CPU 的应用不得不频繁地执行垃圾回收，影响用户代码对 CPU 的利用率，进而影响用户代码的执行效率。比如一个Cronjob频繁的创建和销毁对象，导致OOM。



Uber 内部搞了一个叫 [GOGCTuner](https://www.uber.com/blog/how-we-saved-70k-cores-across-30-mission-critical-services)的库。这个库简化了 Go 的 GOGC 参数调整流程，根据容器的 memory limit使用 Go 的 runtime API ，动态地调整 GOGC 参数。

## 参考资料

1. https://draveness.me/golang/docs/part3-runtime/ch07-memory/golang-garbage-collector/
1. https://golang.design/under-the-hood/zh-cn/part2runtime/ch08gc/barrier/#825-
