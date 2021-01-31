# io_uring-zh-cn

## 元信息
### 原文出处
https://kernel.dk/io_uring.pdf

### 贡献者
* DuckSoft <<realducksoft@gmail.com>>
* Darsvador <<Lx3JQkmzRS@protonmail.com>>
* KevinZonda <<realkevin@tutanota.com>>

### 授权协议
本翻译文本以 CC-BY-SA 4.0 协议发布：https://creativecommons.org/licenses/by-sa/4.0/deed.zh-Hans

原文著作权归原作者所有。

---

## `io_uring` 高效 IO

本文旨在作为 Linux 的最新 IO 接口 —— `io_uring` 的介绍，并将其与现有技术进行比较。我们将探讨其存在的原因，它的内部工作原理以及其用户可见的接口。

本文不会详细介绍具体的命令和函数，因为那只是重复相关 manpage 中的信息。相反，本文将尽力阐述对 `io_uring` 的介绍以及它是如何工作的，目标是希望让读者对这一切如何联系在一起有更深的理解。

也就是说，这篇文章和 manpage 之间会有一些重叠。要是什么细节也不提，那也基本不可能讲明白 `io_uring`。


### 1.0 介绍
Linux 上有很多基于文件 IO 的方法。最古老的和最基本的是 `read(2)` 和 `write(2)` 系统调用。后来 `pread(2)` 和 `pwrite(2)` 通过允许传入偏移量的方式增强了它们。之后引入的 `preadv(2)` 和 `pwritev2(2)` 是前者的矢量版本。因为这些系统调用依然不够丰富，Linux 还拥有 `pread2(2)` 和 `pwrite2(2)` 的系统调用，它们进一步扩展了 API 以允许使用修饰符标志。

除去这些系统调用的各种差异，它们的共同特征是同步的接口。这意味着当数据准备就绪（或写入完毕）时，系统调用才结束。

对于某些次优的情况，需要一个异步的接口。POSIX 拥有 `aio_read(3)` 和 `aio_write(3)` 来满足这个需求，但是实现这个需求过程很枯燥并且性能很差。

Linux确实有一个原生的异步 IO 接口，简称为 aio。不幸的是，这个东西（aio）有以下的局限性：

1. **最大的限制条件显然是这东西只能支持 `O_DIRECT`（或者无缓冲的）的异步 IO 访存。** 由于 `O_DIRECT` 本身的限制（缓存绕过，以及大小/对齐约束条件），原生的 aio 接口对大多数场景都不适用。对普通的（有缓冲的）IO，接口的行为是同步的。

2. **即使你满足了 IO 能变成异步的所有条件，有些时候它其实也不是异步的。** 有大把的方法能让 IO 提交最终变成阻塞的 —— 如果进行 IO 需要元数据，提交最后就会阻塞等待那东西。

   对于存储设备，同一时间只有固定数目的请求槽位可用。要是这些槽位都被占满了，提交就会阻塞等待，直到有一个能用的槽位。这些不确定的因素意味着依赖提交始终是异步的应用程序仍然被迫卸载这部分。

3. **垃圾 API。** 每个 IO 提交最终都需要拷贝 64+8 字节，每次完成都要拷贝 32 字节。这就得拷贝 104 字节的内存，然而 IO 本应是零拷贝的。取决于你的 IO 大小，这一点肯定很明显。

   暴露出的完成事件（completion event）环缓冲区（ring buffer）大多数情况下添麻烦让完成更慢，并且非常非常难（几乎不可能）从应用层正确地使用。

   IO 总是需要至少两次系统调用（提交 + 等待完成），而在这些 spectre/meltdown 漏洞影响的日子里会严重拖慢速度。


多年以来，人们一直在为解决上述的第一条限制（译注：不支持缓冲的 IO）而做着各种努力（作者本人在 2010 年的时候也试过），但没人成功过。

从效率的方面来说，低于10微秒时延、超高 IOPS 的硬件设备的到来，使得这接口真正地开始显老了。对于这些类型的设备来说，缓慢和非确定性的提交延迟是很致命的问题，就像单核榨不出多少性能一样。

除此之外，因为上述的限制，可以说原生 Linux aio 的用例并不多。它已沦为小众应用的一个角落，以及所有随之而来的问题（长期未发现的 bug等 ）。

此外，“正常”的应用程序用不着 aio，这说明 Linux 仍然缺乏一个能够提供他们所希望的功能的接口。应用程序或库完全没有理由继续需要创建私有的 IO 卸载线程池来获得像样的异步 IO，尤其是当内核可以更高效地完成这些工作的时候。

### 2.0 改善现状

最初的工作集中在改善 aio 接口上。并且在此之前，工作进展相当缓慢.

选择此初始方向有多种原因:

1. **如果你扩展和改进现有接口，比提供新接口要好的多。** 采用新的接口需要花费时间，并且要审核和批准新接口可能是一项漫长而艰巨的任务。

2. **一般而言，这项工作更加轻松。** 作为一个开发者，你总是希望以最少的投入完成最大的产出。扩展现有接口在现有测试基础结构方面为你带来许多优势。

现有的 aio 接口由三个主要的系统调用组成：用于设置 aio 上下文的系统调用（`io_setup(2)`）、一个提交IO （`io_submit(2)`）和一个收割和等待IO完成的系统调用（`io_getevents(2)`）。由于需要对多个这些系统调用进行行为更改，因此我们需要添加新的系统调用以传递此信息。

这样就创建了指向同一代码的多个入口点，以及在其他位置的快捷方式。这样的结果在代码复杂性和可维护性方面不是很好，并且最终只能解决其中一个问题。最重要的是，这实际上使其中之一变得更糟，因为现在API的理解和使用更加复杂

尽管很难放弃一开始的工作而另起炉灶，但是很显然，我们需要一些全新的东西。一个可以让我们实现所有要点的东西。我们需要它具有良好的性能和可扩展性，同时还要使它易于使用，并具有现有接口所缺乏的功能。


### 3.0 新接口的设计目标

尽管从头开始不是个很容易做出的决定，这确实让我们有了充分发挥艺术自由、创造新东西的空间。

按重要性由高到低的顺序，主要设计目标是：

1. **用着简单，难以误用。** 一切用户层可见的接口都应以此为主要目标。接口应该直观易用。

2. **可扩展。** 虽然我的背景主要是与存储相关的，但我希望这套接口不仅仅是面向块设备能用。也就是说，也有可能会出现网络和非块存储设备接口。造新轮子当然要（或者至少尝试）要面向未来嘛。

3. **功能丰富。** Linux aio 只满足了一部分应用程序的需要。我不想再造一套只覆盖部分功能，或者需要应用重复造轮子（比如 IO 线程池）的接口。

4. **高效。** 尽管存储 IO 大多都是基于块的，因而大小多在 512 或 4096 字节，但这些大小时候的效率还是对某些应用至关重要的。此外，一些请求甚至可能不携带数据有效载荷。新接口得在每次请求的开销上精打细算。

5. **可伸缩。** 尽管效率和低时延很重要，但在峰值端提供最佳性能也很关键。特别是在存储方面，我们一直在努力提供一个可扩展的基础设施。新接口应该允许我们将这种可扩展性一直暴露在应用程序中。


上面的有些目标可能看起来互相矛盾。高效和可扩展的接口往往很难使用，更重要的是，很难正确使用。既要功能丰富，又要效率高，也很难做到。不过，这些都是我们的目标。

### 4.0 进入 `io_uring` 新时代

不论设计目标先后如何，最初的设计是以效率为中心。效率不是可以事后考虑的事情，必须从一开始进行设计，一旦接口固定就无法修改。

我知道我既不需要提交或完成事件的任何内存副本，也不需要间接访存。在以前的基于 aio 的设计结束时，效率和可扩展性都明显受到了 aio 必须处理多个独立副本来处理两方面的 IO 的损害。

由于拷贝是不可取的，所以很明显，内核以及程序必须优雅地共享 IO 自身定义的结构并完成事件。如果你把共享的思路发展的足够远，那么把共享数据协调同样驻留在应用与内核共享的内存中是一个自然延伸。一旦你实现了这一飞跃，那么两者之间的同步必须以某种方式进行管理也就很清楚了。

一个应用程序无法在不执行系统调用的情况下与内核共享锁，并且系统调用肯定会减少与内核通信的速率。这与我们的效率目标是相悖的。

一种满足我们需求的数据结构应该是单生产者单消费者环形缓冲区。有了共享的环形缓冲区，我们可以通过巧妙运用内存顺序（memory ordering）和内存屏障（memory barrier）消除在应用程序和内核之间的共享锁.

异步接口有两个基本操作：提交请求的操作，以及与所述请求的完成相关联的事件。

对于提交 IO，应用程序是生产者，内核是消费者。而对于完成 IO 则恰好相反，内核会生产完成事件，应用程序则负责消耗它们。因此，我们需要一对环形队列（ring）来为内核和应用程序之间提供一个高效通信通道。

这对环形队列就是新接口 `io_uring` 的核心。它们被合理命名为提交队列 （submission queue，SQ）以及完成队列（completion queue，CQ），并组成了新接口的基础部分。


### 4.1 数据结构

有了通信基础后，是时候看看如何定义用于描述请求和完成事件的数据结构了。

完成（completion）方面清晰明了。它需要携带与操作结果有关的信息，以及某种方式将完成情况链接到它所产生的请求上。对于 `io_uring`，选定的布局如下：

```c
struct io_uring_cqe {
    __u64 user_data;
    __s32 res;
    __u32 flags;
};
```

现在我们应该认识了 `io_uring`，`_cqe` 后缀指的是一个完成队列事件（completion queue event），后面就略称为一个 cqe。

cqe 包含一个 `user_data` 字段。该字段是从请求提交开始就携带的，并且可以包含应用程序识别所述请求所需的任何信息。一种常见的用例是使其成为原始请求的指针。内核不管这个字段，就只把这个字段从提交到完成一直带着走。

`res` 字段是请求的结果。把它想象成系统调用的返回值。对于普通的读/写操作，这就像 `read(2)` 或 `write(2)` 的返回值一样。成功就返回传输了多少字节。失败就返回错误代码的相反数。比如发生了 IO 错误，`res` 的值就是 `-EIO`。

最后，结构体的 `flags` 字段携带与本次操作有关的元信息数据。目前这个字段还没用上。

---

定义请求类型更复杂。它不仅需要描述比完成事件更多的信息，另外这也是 `io_uring` 的一个设计目标，即可以扩展到未来的请求类型。

目前想到的长这样：

![image](https://i.loli.net/2021/01/31/5XS7sDQhY2jZCVf.png)

和完成事件类似，提交端的数据结构叫做提交队列项（Submission Queue Entry），或者简称 sqe。

里面存着一个 `opcode` 字段，描述本次请求的操作码，也就是究竟干点啥。比如，有一个操作码叫 `IORING_OP_READV`，也就是向量读（vectored read）。

`flags` 字段包含各命令类型通用的修饰选项。我们将在之后的进阶使用场景部分提及。

`ioprio` 是本次请求的优先级别。对一般的读写操作，这就和 `ioprio_set(2)` 系统调用的定义里的一样。

`fd` 字段是与本次请求相关联的文件描述符（file descriptor），`off` 字段是这个操作在 `fd` 的多少偏移量发生。

如果操作码描述了一个传输数据的操作，那么 `addr` 字段就包含在应该在哪个地址进行这个操作。例如，如果操作是一个向量读/向量写之类的操作，那么这个 `addr` 字段里就是一个指向 `iovec` 结构体数组的指针，就和 `preadv(2)` 里用的一样。

对于非向量化的 IO 传输，`addr` 就必须直接是目标地址。

这就引入了另一个字段 `len`，对非向量传输来说这是要传输的字节量，而对向量传输来说就是 `iovec` 结构体数组元素的数目。

接下来是一组 flag，指定了与操作码有关的东西。例如，对于上面提到的向量读（`IORING_OP_READV`），这些 flag 就和 `preadv2(2)` 系统调用中要求的保持一致。

`user_data` 在操作码中是通用的，并且内核不会去碰这个东西。当这个请求的完成事件被提交之后，它仅仅被拷贝到完成队列事件结构体（cqe）中。

`buf_index` 之后在进阶用法中会提到的。

最后，在这个结构的末尾，有一些填充（padding）。这样做是为了确保 sqe 在内存中以 64 字节的大小良好排列，同时也是为了将来可能需要包含更多数据以描述请求的情况。

这里有几个能想到的用例 —— 一个是键/值存储命令集，另一个是端到端数据保护，其中程序为其想要写入的数据传递一个预先计算的校验和（checksum）。

### 4.2 通讯信道

描述过这些数据结构之后，我们来看看这个环（ring）工作的细节。

尽管说我们有一个提交端（submission side）和一个完成端（completion side），听起来可能挺对称，但是这两个的索引方式却是完全不同的。

就像上一节一样，让我们从简单的开始，先来讲讲完成环（completion ring）。

完成请求项（cqe）被组织成了一个数组，支撑其的内存对应用程序和内核来说都是可见、可改的。然而，由于完成请求项（cqe）是内核产生的，只有内核在事实上修改 cqe 项目。

通讯是通过一个环缓冲区管理的。当一个新事件被内核提交到完成请求环（CQ 环）中时，它会更新与其相连的尾节点。当程序从中消耗一项时，它会更新头节点。因此，只要头尾节点不同，应用就知道还有一个或更多事件可以用来消耗。

环计数器（ring counter）本身是自由流动的 32 位整数，当环中事件的数目超过环的容量时，依靠自然进位处理。（译者：？？？？？）

> WIP...
