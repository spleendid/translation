
原文地址： [http://www.javaworld.com/javaworld/jw-10-2012/121010-jvm-performance-optimization-garbage-collection.html?page=1][15]

_The Java platform's garbage collection mechanism greatly increases developer productivity, but a poorly implemented garbage collector can over-consume application resources. In this third article in the JVM performance optimization series, Eva Andreasson offers Java beginners an overview of the Java platform's memory model and GC mechanism. She then explains why fragmentation (and not GC) is the major "gotcha!" of Java application performance, and why generational garbage collection and compaction are currently the leading (though not most innovative) approaches to managing heap fragmentation in Java applications._

_Java平台的垃圾回收机制大大提高的开发人员的生产力，但实现糟糕的垃圾回收器却会大大消耗应用程序的资源。本文作为JVM性能优化系列的第3篇，Eva Andeasson将为Java初学者介绍Java平台的内存模型和GC机制。她将解释为什么碎片化（不是GC）是Java应用程序出现性能问题的主要原因，以及为什么当前主要通过分代垃圾回收和压缩，而不是其他最具创意的方法，来解决Java应用程序中碎片化的问题。_


*Garbage collection (GC)* is the process that aims to free up occupied memory that is no longer referenced by any reachable Java object, and is an essential part of the Java virtual machine's (JVM's) dynamic memory management system. In a typical garbage collection cycle all objects that are still referenced, and thus reachable, are kept. The space occupied by previously referenced objects is freed and reclaimed to enable new object allocation.

*垃圾回收（GC）*是旨在释放不可达Java对象所占用的内存的过程，是Java virtual machine（JVM）中动态内存管理系统的核心组成部分。在一个典型的垃圾回收周期中，所有仍被引用的对象，即可达对象，会被保留。没有被引用的Java对象所占用的内存会被释放并回收，以便分配给新创建的对象。

In order to understand garbage collection and the various GC approaches and algorithms, you must first know a few things about the Java platform's memory model.

为了更好的理解垃圾回收与各种不同的GC算法，你首先需要了解一些关于Java平台内存模型的内容。


#Garbage collection and the Java platform memory model#

#垃圾回收与Java平台内存模型

When you specify the startup option -Xmx on the command line of your Java application (for instance: java -Xmx:2g MyApp) memory is assigned to a Java process. This memory is referred to as the *Java heap* (or just *heap*). This is the dedicated memory address space where all objects created by your Java program (or sometimes the JVM) will be allocated. As your Java program keeps running and allocating new objects, the Java heap (meaning that address space) will fill up.

当你在启动Java应用程序时指定了启动参数_-Xmx_（例如，java -Xmx2g MyApp），则相应大小的内存会被分配给Java进程。这块内存即所谓的*Java堆*（或简称为*堆*）。这块专用的内存地址空间用于存储Java应用程序（有时是JVM）所创建的对象。随着Java应用程序的运行，会不断的创建新对象并为之分配内存，Java堆（即地址空间）会逐渐被填满。

Eventually, the Java heap will be full, which means that an allocating thread is unable to find a large-enough consecutive section of free memory for the object it wants to allocate. At that point, the JVM determines that a garbage collection needs to happen and it notifies the garbage collector. A garbage collection can also be triggered when a Java program calls System.gc(). Using System.gc() does not guarantee a garbage collection. Before any garbage collection can start, a GC mechanism will first determine whether it is safe to start it. It is safe to start a garbage collection when all of the application's active threads are at a safe point to allow for it, e.g. simply explained it would be bad to start garbage collecting in the middle of an ongoing object allocation, or in the middle of executing a sequence of optimized CPU instructions (see my previous article on compilers), as you might lose context and thereby mess up end results.

最后，Java堆会被填满，这就是说想要申请内存的线程无法获得一块足够大的连续空闲空间来存放新创建的对象。此时，JVM判断需要启动垃圾回收器来回收内存了。当Java程序调用System.gc()方法时，也有可能会触发垃圾回收器以执行垃圾回收的工作。使用System.gc()方法并不能保证垃圾回收工作肯定会被执行。在执行垃圾回收前，垃圾回收机制首先会检查当前是否是一个“恰当的时机”，而“恰当的时机”指所有的应用程序活动线程都处于安全点（safe point），以便启动垃圾回收。简单举例，为对象分配内存时，或正在优化CPU指令（参见本系列的[前一篇文章][3]）时，就不是“恰当的时机”，因为你可能会丢失上下文信息，从而得到混乱的结果。

A garbage collector should never reclaim an actively referenced object; to do so would break the [Java virtual machine specification][1]. A garbage collector is also not required to immediately collect dead objects. Dead objects are eventually collected during subsequent garbage collection cycles. While there are many ways to implement garbage collection, these two assumptions are true for all varieties. The real challenge of garbage collection is to identify everything that is live (still referenced) and reclaim any unreferenced memory, but do so without impacting running applications any more than necessary. A garbage collector thus has two mandates:

垃圾回收不应该回收当前有活动引用指向的对象所占用的内存；因为这样做将违反[JVM规范][1]。在JVM规范中，并没有强制要求垃圾回收器立即回收已死对象（dead object）。已死对象最终会在后续的垃圾回收周期中被释放掉。目前，已经有多种垃圾回收的实现，它们都包含两个沟通的假设。对垃圾回收来说，真正的挑战在于标识出所有活动对象（即仍有引用指向的对象），回收所有不可达对象所占用的内存，并尽可能不对正在运行的应用程序产生影响。因此，垃圾回收器运行的两个目标：

1. To quickly free unreferenced memory in order to satisfy an application's allocation rate so that it doesn't run out of memory.
2. To reclaim memory while minimally impacting the performance (e.g., latency and throughput) of a running application.

1. 快速释放不可达对象所占用的内存，防止应用程序出现OOM错误。
2. 回收内存时，对应用程序的性能（指延迟和吞吐量）的影响要紧性能小。
    

#Two kinds of garbage collection#

#两类垃圾回收#

In the [first article][2] in this series I touched on the two main approaches to garbage collection, which are reference counting and tracing collectors. This time I'll drill down further into each approach then introduce some of the algorithms used to implement tracing collectors in production environments.

在本系列的[第一篇文章][2]中，我提到了2种主要的垃圾回收方式，引用计数（reference counting）和引用追踪（tracing collector。译者注，在第一篇中，给出的名字是“reference tracing”，这里仍沿用之前的名字）。这里，我将深入这两种垃圾回收方式，并介绍用于生产环境的实现了引用追踪的垃圾回收方式的相关算法。

>Read the JVM performance optimization series
>
>JVM performance optimization, Part 1: [Overview][2]
>JVM performance optimization, Part 2: [Compilers][3]

>相关阅读：JVM性能优化系列
>
>JVM性能优化，第一部分: [概述][2]
>JVM性能优化，第二部分: [编译器][3]


##Reference counting collectors##

##引用计数垃圾回收器##

*Reference counting collectors* keep track of how many references are pointing to each Java object. Once the count for an object becomes zero, the memory can be immediately reclaimed. This immediate access to reclaimed memory is the major advantage of the reference-counting approach to garbage collection. There is very little overhead when it comes to holding on to un-referenced memory. Keeping all reference counts up to date can be quite costly, however.

*引用计数垃圾回收器*会对指向每个Java对象的引用数进行跟踪。一旦发现指向某个对象的引用数为0，则立即回收该对象所占用的内存。引用计数垃圾回收的主要优点就在于可以立即访问被回收的内存。垃圾回收器维护未被引用的内存并不需要消耗很大的资源，但是保持并不断更新引用计数却代价不菲。

The main difficulty with reference counting collectors is keeping the reference counts accurate. Another well-known challenge is the complexity associated with handling circular structures. If two objects reference each other and no live object refers to them, their memory will never be released. Both objects will forever remain with a non-zero count. Reclaiming memory associated with circular structures requires major analysis, which brings costly overhead to the algorithm, and hence to the application.

使用引用计数方式执行垃圾回收的主要困难在于保持引用计数的准确性，而另一个众所周知的问题在于解决循环引用结构所带来的麻烦。如果两个对象互相引用，并且没有其他存活东西引用它们，那么这两个对象所占用的内存将永远不会被释放，两个对象都会因引用计数不为0而永远存活下去。要解决循环引用带来的问题需要，而这会使算法复杂度增加，从而影响应用程序的运行性能。


##Tracing collectors##

##引用跟踪垃圾回收##

*Tracing collectors* are based on the assumption that all live objects can be found by iteratively tracing all references and subsequent references from an initial set of known to be live objects. The initial set of live objects (called root objects or just roots for short) are located by analyzing the registers, global fields, and stack frames at the moment when a garbage collection is triggered. After an initial live set has been identified, the tracing collector follows references from these objects and queues them up to be marked as live and subsequently have their references traced. Marking all found referenced objects live means that the known live set increases over time. This process continues until all referenced (and hence all live) objects are found and marked. Once the tracing collector has found all live objects, it will reclaim the remaining memory.

*引用跟踪垃圾回收器*基于这样一种假设，所有存活对象都可以通过迭代地跟踪从已知存活对象集中对象发出的引用及引用的引用来找到。可以通过对寄存器、全局域、以及触发垃圾回收时栈帧的分析来确定初始存活对象的集合（称为“根对象”，或简称为“根”）。在确定了初始存活对象集后，引用跟踪垃圾回收器会跟踪从这些对象中发出的引用，并将找到的对象标记为“活的（live）”。标记所有找到的对象意味着已知存货对象的集合会随时间而增长。这个过程会一直持续到所有被引用的对象（因此是“存活的”对象）都被标记。当引用跟踪垃圾回收器找到所有存活的对象后，就会开始回收未被标记的对象。

Tracing collectors differ from reference-counting collectors in that they can handle circular structures. The catch with most tracing collectors is the marking phase, which entails a wait before being able to reclaim non-referenced memory.

不同于引用计数垃圾回收器，引用跟踪垃圾回收器可以解决循环引用的问题。由于标记阶段的存在，大多数引用跟踪垃圾回收器无法立即释放“已死”对象所占用的内存。

Tracing collectors are most commonly used for memory management in dynamic languages; they are by far the most common for the Java language and have been commercially proven in production environments for many years. I'll focus on tracing collectors for the remainder of this article, starting with some of the algorithms that implement this approach to garbage collection.

引用跟踪垃圾回收器广泛用于动态语言的内存管理；到目前为止，在Java编程语言的视线中也是应用最广的，并且在多年的商业生产环境中，已经证明其实用性。在本文余下的内容中，我将从一些相关的实现算法开始，介绍引用跟踪垃圾回收器，


#Tracing collector algorithms#

#引用跟踪垃圾回收器算法#

*Copying* and *mark-and-sweep* garbage collection are not new, but they're still the two most common algorithms that implement tracing garbage collection today.

*拷贝*和*标记-清理*垃圾回收算法并非新近发明，但仍然是当今实现引用跟踪垃圾回收器最常用的两种算法。


##Copying collectors##

##拷贝垃圾回收器##

Traditional copying collectors use a *from-space* and a *to-space* -- that is, two separately defined address spaces of the heap. At the point of garbage collection, the live objects within the area defined as from-space are copied into the next available space within the area defined as to-space. When all the live objects within the from-space are moved out, the entire from-space can be reclaimed. When allocation begins again it starts from the first free location in the to-space.

传统的拷贝垃圾回收器会使用一个“from”区和一个“to”区，它们是堆中两个不同的地址空间。在执行垃圾回收时，from区中存活对象会被拷贝到to区。当from区中所有的存活对象都被拷贝到to后，垃圾回收器会回收整个from区。当再次分配内存时，会首先从to区中的空闲地址开始分配。

In older implementations of this algorithm the from-space and to-space switch places, meaning that when the to-space is full, garbage collection is triggered again and the to-space becomes the from-space, as shown in Figure 1.

在该算法的早期实现中，from区和to区会在垃圾回收周期后进行交换，即当to区被填满后，将再次启动垃圾回收，这是to区会“变成”from区。如图Figure 1所示。

![Figure 1. A traditional copying garbage collection sequence](images/jvmseries3-fig1.png?raw=true "Figure 1. A traditional copying garbage collection sequence")

Figure 1. A traditional copying garbage collection sequence

More modern implementations of the copying algorithm allow for arbitrary address spaces within the heap to be assigned as to-space and from-space. In these cases they do not necessarily have to switch location with each other; rather, each becomes another address space within the heap.

在该算法的近期实现中，可以将堆中任意地址空间指定为from区和to区，这样就不再需要交换from区和to区，堆中任意地址空间都可以成为from区或to区。

One advantage of copying collectors is that objects are allocated together tightly in the to-space, completely eliminating fragmentation. Fragmentation is a common issue that other garbage collection algorithms struggle with; something I'll discuss later in this article.

拷贝垃圾回收器的一个优点是存活对象的位置会被to区中重新分配，紧凑存放，可以完全消除碎片化。碎片化是其他垃圾回收算法所要面临的一大问题，这点会在后续讨论。


###Downsides of copying collectors###

###拷贝垃圾回收的缺陷###

Copying collectors are usually *stop-the-world* collectors, meaning that no application work can be executed for as long as the garbage collection is in cycle. In a stop-the-world implementation, the larger the area you need to copy, the higher the impact on your application performance will be. This is a disadvantage for applications that are sensitive to response time. With a copying collector you also need to consider the worst-case scenario, when everything is live in the from-space. You always have to leave enough headroom for live objects to be moved, which means the to-space must be large enough to host everything in the from-space. The copying algorithm is slightly memory inefficient due to this constraint.

通常来说，拷贝垃圾回收器是“stop-the-world”式的，即在垃圾回收周期内，应用程序是被挂起的，无法工作。在“stop-the-world”式的实现中，所需要拷贝的区域越大，对应用程序的性能所造成的影响也越大。对于那些非常注重响应时间的应用程序来说，这是难以接受的。使用拷贝垃圾回收时，你还需要考虑一下最坏情况，即当from区中所有的对象都是存活对象的时候。因此，你不得不给存活对象预留出足够的空间，也就是说to区必须足够大，大到可以将from区中所有的对象都放进去。正是由于这个缺陷，拷贝垃圾回收算法在内存使用效率上略有不足。


##Mark-and-sweep collectors##

##标记-清理垃圾回收器##

Most commercial JVMs deployed in enterprise production environments run mark-and-sweep (or marking) collectors, which do not have the performance impact that copying collectors do. Some of the most famous marking collectors are CMS, G1, GenPar, and DeterministicGC (see [Resources][13]).

大多数部署在企业生产环境的商业JVM都使用了标记-清理（或标记）垃圾回收器，这种垃圾回收器并不会想拷贝垃圾回收器那样对应用程序的性能有那么大的影响。其中最著名的几款是CMS、G1、GenPar和DeterministicGC（参见[相关资源][13]）。

A *mark-and-sweep* collector traces references and marks each found object with a "live" bit. Usually a set bit corresponds to an address or in some cases a set of addresses on the heap. The live bit can, for instance, be stored as a bit in the object header, a bit vector, or a bit map.

*标记-清理*垃圾回收器会跟踪引用，并使用标记位将每个找到的对象标记位“live”。通常来说，每个标记位都关联着一个地址或堆上的一个地址集合。例如，标记位可能是对象头（object header）中一位，一个位向量，或是一个位图。

After everything has been marked live, the sweep phase will kick in. If a collector has a sweep phase it basically includes some mechanism for traversing the heap again (not just the live set but the entire heap length) to locate all the non-marked chunks of consecutive memory address spaces. Unmarked memory is free and reclaimable. The collector then links together these unmarked chunks into organized free lists. There can be various free lists in a garbage collector -- usually organized by chunk sizes. Some JVMs (such as JRockit Real Time) implement collectors with heuristics that dynamically size-range lists based on application profiling data and object-size statistics.

当所有的存活对象都被标记位“live”后，将会开始*清理*阶段。一般来说，垃圾回收器的清理阶段包含了通过再次遍历堆（不仅仅是标记位live的对象集合，而是整个堆）来定位内存地址空间中未被标记的区域，并将其回收。然后，垃圾回收器会将这些被回收的区域保存到空闲列表（free list）中。在垃圾回收器中可以同时存在多个空闲列表——通常会按照保存的内存块的大小进行划分。某些JVM（例如JRockit实时系统， JRockit Real Time System）在实现垃圾回收器时会给予应用程序分析数据和对象大小统计数据来动态调整空闲列表所保存的区域块的大小范围。

When the sweep phase is complete allocation will begin again. New allocation areas are allocated from the free lists and memory chunks could be matched to object sizes, object size averages per thread ID, or the application-tuned TLAB sizes. Fitting free space more closely to the size of what your application is trying to allocate optimizes memory and could help reduce fragmentation.

当清理阶段结束后，应用程序就可以再次启动了。给新创建的对象分配内存时会从空闲列表中查找，而空闲列表中内存块的大小需要匹配于新创建的对象大小、某个线程中平均对象大小，或应用程序所设置的TLAB的大小。从空闲列表中为新创建的对象找到大小合适的内存区域块有助于优化内存的使用，减少内存中的碎片。


>More about TLAB sizes
>
>TLAB and TLA (Thread Local Allocation Buffer or Thread Local Area) partitioning are discussed in [JVM performance optimization, Part 1][2].

>关于TLAB
>
>更多关于TLAB和TLA（Thread Local Allocation Buffer和Thread Local Area）的内容，请参见[JVM performance optimization, Part 1][2]。


###Downsides of mark-and-sweep collectors###

###标记-清理垃圾回收器的缺陷###

The mark phase is dependent on the amount of live data on your heap, while the sweep phase is dependent on the heap size. Since you have to wait until both the *mark* and *sweep* phases are complete to reclaim memory, this algorithm causes pause-time challenges for larger heaps and larger live data sets.

标记阶段的时长取决于堆中存活对象的总量，而清理阶段的时长则依赖于堆的大小。由于在*标记*阶段和*清理*阶段完成前，你无事可做，因此对于那些具有较大的堆和较多存活对象的应用程序来说，使用此算法需要想办法解决暂停时间（pause-time）较长这个问题。

One way that you can help heavily memory-consuming applications is to use GC-tuning options that accommodate various application scenarios and needs. Tuning can, in many cases, help at least postpone either of these phases from becoming a risk to your application or service-level agreements (SLAs). (An SLA specifies that the application will meet certain application response times -- i.e., latency.) Tuning for every load change and application modification is a repetitive task, however, as the tuning is only valid for a specific workload and allocation rate.

对于那些内存消耗较大的应用程序来说，你可以使用一些GC调优选项来满足其在某些场景下的特殊需求。很多时候，调优至少可以将标记-清理阶段给应用程序或性能要求（SLA，SLA指定了应用程序需要达到的响应时间的要求，即延迟）所带来的风险推后。当负载和应用程序发生改变后，需要重新调优，因为某次调优只对特定的工作负载和内存分配速率有效。


##Implementations of mark-and-sweep##

##标记-清理算法的实现##

There are at least two commercially available and proven approaches for implementing mark-and-sweep collection. One is the parallel approach and the other is the concurrent (or mostly concurrent) approach.

目前，标记-清理垃圾回收算法至少已有2种商业实现，并且都已在生产环境中被证明有效。其一是并行垃圾回收，另一个是并发（或多数时间并发）垃圾回收。


###Parallel collectors###

###并行垃圾回收器###

*Parallel collection* means that resources assigned to the process are used in parallel for the purpose of garbage collection. Most commercially implemented parallel collectors are monolithic stop-the-world collectors -- all application threads are stopped until the entire garbage collection cycle is complete. Stopping all threads allows all resources to be efficiently used in parallel to finish the garbage collection through the mark and sweep phases. This leads to a very high level of efficiency, usually resulting in high scores on throughput benchmarks such as [SPECjbb][14]. If throughput is essential for your application, the parallel approach is an excellent choice.

*并行垃圾回收*指的是垃圾回收是多线程并行完成的。大多数商业实现的并行垃圾回收器都是stop-the-world式的垃圾回收器，即在整个垃圾回收周期结束前，所有应用程序线程都会被挂起。挂起所有应用程序线程使垃圾回收器可以以并行的方式，更有效的完成标记和清理工作。并行使得效率大大提高，通常可以在像[SPECjbb][14]这样的吞吐量基准测试中跑出高分。如果你的应用程序好似有限考虑吞吐量的，那么并行垃圾回收是你最好的选择。

The cost of most parallel collection -- and do consider this, especially for production environments -- is that application threads cannot do any work during a GC, just like with copying collectors. Using a parallel collector that implements stop-the-world collection will have a major impact on response-time sensitive applications, especially if you have a lot of references to trace, which will happen with many live or complex data structures on the heap. (Remember that for mark-and-sweep collectors the time to free up new memory is dependent on the time it takes to trace the live data set plus the time to traverse the heap during the sweep phase.) For a monolithic parallel approach using all resources in parallel, this entire time will be a pause, and that pause corresponds to the entire GC cycle.

对于大多数并行垃圾回收器来说，尤其是考虑到应用于生产环境中，最大的问题是，像拷贝垃圾回收算法一样，在垃圾回收周期内应用程序无法工作。使用stop-the-world式的并行垃圾回收会对优先考虑响应时间的应用程序产生较大影响，尤其是当你有大量的引用需要跟踪，而此时恰好又有大量的、具有复杂结构的对象存活于堆中的时候，情况将更加糟糕。（记住，标记-清理垃圾回收器回收内存的时间取决于跟踪存活对象中所有引用的时间与遍历整个堆的时间之和。）以并行方式执行垃圾回收所导致的应用程序暂停会一直持续到整个垃圾回收周期结束。


###Concurrent collectors###

###并发垃圾回收器###

A concurrent collector is a much better fit for applications that are sensitive to response time. Concurrent means that some (or most) garbage collection work is performed concurrently with the running application threads. As not all resources are used for GC, you will have the challenge of deciding when to start a garbage collection in order to allow enough time for the cycle to end. You need enough time to trace the live set and reclaim the memory before the application runs out of memory. If the garbage collection doesn't complete in time the application will throw an out-of-memory error. You don't want to do garbage collection all the time because that would consume application resources, thus impacting throughput. It can be extra tricky to keep that balance in very dynamic environments, so heuristics have been designed to determine when to start garbage collection and when to do various GC optimizing tasks and how much at a time, etc.

并发垃圾回收器更适用于那些对响应时间比较敏感的应用程序。并发指的是一些（或大多数）垃圾回收工作可以与应用程序线程同时运行。由于并非所有的资源都由垃圾回收器使用，因此这里所面临的问题如何决定何时开始执行垃圾回收，可以保证垃圾回收顺利完成。这里需要足够的时间来跟踪存活对象即的引用，并在应用程序出现OOM错误前回收内存。如果垃圾回收器无法及时完成，则应用程序就会抛出OOM错误。此外，一直做垃圾回收也不好，会不必要的消耗应用程序资源，从而影响应用程序吞吐量。要想在动态环境中保持这种平衡就需要一些技巧，因此设计了启发式方法来决定何时开始垃圾回收，何时执行不同的垃圾回收优化任务，以及一次执行多少垃圾回收优化任务等。

Another challenge is to determine when it is safe to perform operations that require a complete and true snapshot of the world -- for instance, you need to know when all live objects have been marked, and thus when to switch to the sweep phase. In the monolithic stop-the-world scenario employed by most parallel collectors, this *phase-switching* is less tricky because the world is already standing still. But in concurrent implementations it might not be safe to switch phases immediately. For instance, if an application has modified an area that has already been traced and marked by the collector, new or unmarked references may have been touched, which would make them live. In some implementations this situation will put your application at risk for long-time running re-marking loops, potentially making it hard for your application to get new free memory when it needs it.

并发垃圾回收器所面临的另一个挑战是如何决定何时执行一个需要完整堆快照的操作时安全的，例如，你需要知道是何时标记所有存活对象的，这样才能转而进入清理阶段。在大多数并行垃圾回收器采用的stop-the-world方式中，*阶段转换（phase-switching）*并不需要什么技巧，因为世界已静止（堆上对象暂时不会发生变化）。但是，在并发垃圾回收中，转换阶段时可能并不是安全的。例如，如果应用程序修改了一块垃圾回收器已经标记过的区域，可能会涉及到一些新的或未被标记的引用，而这些引用使其指向的对象成为存活状态。在某些并发垃圾回收的实现中，这种情况有可能会使应用程序陷入长时间运行重标记（re-mark）的循环，因此当应用程序需要分配内存时无法得到足够做的空闲内存。

The takeaway from this discussion so far is that you have numerous options among garbage collectors and GC algorithms, some better suited than others to specific application types and workloads. Not only are there different algorithms, but there are actually different implementations of the various algorithms. So it is wise to be informed about your application's allocation needs and characteristics before simply specifying a garbage collector on the command line. In the next section we'll look at some of the pitfalls of the Java platform memory model -- and by pitfalls I mean places where Java developers tend to make assumptions that lead to worse performance for dynamic production loads, not better.

到目前为止的讨论中，已经介绍了各种垃圾回收器和垃圾回收算法，他们各自适用于不同的场景，满足不同应用程序的需求。各种垃圾回收方式不仅在算法上有所区别，在具体实现上也不尽相同。所以，在命令行中指定垃圾回收器之前，最好能了解应用程序的需求及其自身特点。在下一节中，将介绍Java平台内存模型中的陷阱，在这里，陷阱指的是在动态生产环境中，Java程序员常常做出的一些中使性能更糟，而非更好的假设。


#Why tuning doesn't replace garbage collection#

#为什么调优无法取代垃圾回收#

Most Java developers know that there are choices to be made if you want to maximize Java performance. The current variety of JVMs, garbage collectors, and the overwhelming selection of tuning options can lead developers to spend a lot of deployment time on the never-ending task of performance tuning. This has led some to conclude that GC is bad and that tuning so that GC happens infrequently or for just a short time is a successful workaround. But there are risks to doing GC this way.

大多数Java程序员都知道，如果有不少方法可以最大化Java程序的性能。而当今众多的JVM实现，垃圾回收器实现，以及多到令人头晕的调优选项都可能会让开发人员将大量的时间消耗在无穷无尽的性能调优上。这种情况催生了这样一种结论，“GC是糟糕的，努力调优以降低GC的频率或时长才是王道”。但是，真这么做是有风险的。

Consider what it means to tune against specific application needs. Most tuning parameters -- such as allocation rate, object sizes, timing of response-time sensitive tasks, and how fast objects die -- are tuned specifically for the application's allocation rate, such as the test workload at hand. The end result could be either (or both) of these:

考虑一下针对指定的应用程序需求做调优意味着什么。大多数调优参数，如内存分配速率，对象大小，响应时间，以及对象死亡速度等，都是针对特定的情况而来设定的，例如测试环境下的工作负载。例如。调优结果可能有以下两种：

1. Things that worked during testing fail in production.
2. Workload or application changes require you to re-tune the application entirely.

1. 测试时正常，上线就失败。
2. 一旦应用程序本身，或工作负载发生改变，就需要全部重调。

Tuning is something that will always need to be repeated! Concurrent garbage collectors in particular can require a lot of tuning -- especially in production environments. You need the heuristics to match your specific application's needs for the expected worst-case load. The end result becomes a very rigid configuration, leading to a lot of resource waste. This approach to tuning (tuning to eliminate GC) is kind of a Don Quixote quest -- a chase after an imagined enemy with sure-to-lose cause. The fact is that the more you tune your collectors to fit a specific load, the farther you'll be from the dynamic features of a Java runtime. After all, how many applications really have a static load? And how reliably can you really predict for an expected load?

调优是需要不断往复的。使用并发垃圾回收器需要做很多调优工作，尤其是在生产环境中。为满足应用程序的需求，你需要不断挑战可能要面对的最差情况。这样做的结果就是，最终形成的配置非常刻板，而且在这个过程中也浪费了大量的资源。这种调优方式（试图通过调优来消除GC）是一种堂吉诃德式的探索——以根本不存在的理由去挑战一个假想敌。而事实是，你针对某个特定的负载而垃圾回收器做的调优越多，你距离Java运行时的动态特性就越远。毕竟，有多少应用程序的工作负载能保持不变呢？你所预估的工作负载的可靠性又有多高呢？

So, what if you didn't focus on tuning? What could you be doing differently to prevent out-of-memory errors and improve response times? The first step toward finding out is to identify the real challenge to Java application performance.

那么，如果不从调优入手又该怎么办呢？有什么其他的办法可以防止应用程序出现OOM错误，并降低响应时间呢？这里，首先要做的是明确影响Java应用程序性能的真正因素。


##Fragmentation##

##碎片化##

The real Java performance challenge is not the garbage collector itself; it is fragmentation, and how the garbage collector deals with it. Fragmentation is a state of the heap where free memory is available but not in a big enough consecutive memory space to host a new object about to be allocated. As I mentioned in [Part 1][2], a fragment is either a residual space in the Java heap's TLABs, or (more often) a space previously occupied by small objects between longer living objects.

影响Java应用程序性能的罪魁祸首并不是垃圾回收器本身，而是碎片化，以及垃圾回收器如何处理碎片。碎片是Java堆中空闲空间，但由于连续空间不够大而无法容纳将要创建的对象。正如我在本系列[第2篇][2]中提到的，碎片可能是TLAB中的剩余空间，也可能是（这种情况比较多）被释放掉的具有较长生命周期的小对象所占用的空间。

Over time, as an application runs, these fragments of unusable memory will appear all over the heap. In some cases the state will get worse with statically tuned options (like promotion rates, free lists, etc.) that can't keep up with the needs of a dynamic application. Such left-over (fragmented) space can't be used efficiently by an application. If you don't do anything about it you'll eventually end up with back-to-back garbage collections, the garbage collector's attempt to free up memory for a new object allocation request. In the worst-case scenario, even a number of consecutive GCs will not free up memory (too many fragments) and the JVM will be forced to throw an out-of-memory error. You can deal with fragmentation by restarting the application, which makes your heap a new space of consecutive memory mapped to your process, such that you'll be able to allocate objects freely again. Restarting leads to downtime, however, and besides the heap would eventually get fragmented again and you would need to do another restart.

随着应用程序的运行，这种无法使用的碎片会遍布于整个堆空间。在某些情况下，这种状态会因静态调优选项（如提升速率和空闲列表等）更糟糕，以至于无法满足应用程序的原定需求。这些剩下的空间（也就是碎片）无法被应用程序有效利用起来。如果你对此放任自流，就会导致不断垃圾回收，垃圾回收器会不断的释放内存以便创建新对象时使用。在最差情况下，甚至垃圾回收也无法腾出足够的内存空间（因为碎片太多），JVM会强制抛出OOM（out of memory）错误当然，你也可以重启应用程序来消除碎片，这样可以使Java堆焕然一新，于是就又可以为对象分配内存了。但是，重新启动会导致服务器停机，另外，一段时间之后，堆将再次充满碎片，你也不得不再次重启。

_OutOfMemoryErrors_, hung processes, and logs showing that your garbage collector is working overtime are all symptoms of a GC struggling to free up memory, and are indicators of a fragmented heap. While some developers seek to resolve fragmentation by simply re-tuning their garbage collector, I argue that we should explore more innovative solutions to the problem of fragmentation before it becomes an issue for GC. The remainder of this article will focus on the two most effective approaches to managing fragmentation: generational garbage collection and compaction.

_OOM错误（OutOfMemoryErrors）_会挂起进程，日志中显示的垃圾回收器很忙，是垃圾回收器努力释放内存的标志，也说明了堆中碎片非常多。一些开发人员通过重新调优垃圾回收器来解决碎片化的问题，但我觉着在解决碎片问题成为垃圾回收的使命之前应该用一些更有新意的方法来解决这个问题。本文后面的内容将聚焦于能有效解决碎片化问题的方法：粉黛式垃圾回收和压缩。


#Generational garbage collection#

#分代式垃圾回收#

You may have heard the theory that in production environments most objects die young. Generational garbage collection is an approach to GC that springs from this theory. In a generational garbage collection you basically divide the heap into different spaces (or generations), each covering different ages of objects, where age could mean the number of GC cycles that the object has survived (that is, how many times it has remained referenced).

这个理论你可以已经听说过，即在生产环境中，大部分对象的生命周期都很短。分代式垃圾回收就源于这个理论。在分代式垃圾回收中，堆被分为两个不同的空间（或成为“代”），每个空间存放具有不同年龄的对象，在这里，年龄是指该对象所经历的垃圾回收的次数（也就是该对象挺过了多少次垃圾回收而没有死掉）。

When the space for new allocations, also known as *young space*, *nursery*, or *young generation*, is full, objects still referenced within that space are moved into the next generation. Most commonly there are only two generations. Usually generational garbage collectors are one-directional copying collectors, which I discussed earlier. Some more-recent implementations of young generation collectors are parallel collectors. It is also possible to implement different collecting algorithms for young space and old space. If you are using a parallel and/or copying collector then your young generation collectors will be a stop-the-world collector (see earlier explanation).

当新创建的对象所处的空间，即*年轻代*，被对象填满后，该空间中仍然存活的对象会被移动到老年代。（译者注，以HotSpot为例，这里应该是挺过若干次GC而不死的，才会被搬到老年代，而一些比较大的对象会直接放到老年代。）大多数的实现都将堆会分为两代，年轻代和老年代。通常来说，分代式垃圾回收器都是单向拷贝的，即从年轻代向老年代拷贝，这点在早先曾讨论过。近几年出现的年轻代垃圾回收器已经可以实现并行垃圾回收，当然也可以实现一些其他的垃圾回收算法实现对年轻代和老年代的垃圾回收。如果你使用拷贝垃圾回收器（可能具有并行收集功能）对年轻代进行垃圾回收，那垃圾回收是stop-the-world式的（参见前面的解释）。

*Old space* or *old generation* hosts the promoted objects that stay referenced for a certain time or certain numbers of young space collections. On occasion, really large objects might be allocated directly into old space, as large objects are more costly to move.

*老年代(old space或old generation)*用于保存那些挺过了若干论年轻代垃圾回收之后的对象。某些情况下，一些大对象可能会被直接分配到老年代，因为移动代对象的开销比较大。


##The mechanics of generational GC##

##分代式垃圾回收的缺陷#

In a generational approach, garbage collection happens less frequently in old space and more frequent in young space, where the GC cycle is hopefully shorter and less intrusive. Having a young space can, on rare occasions, lead to more frequent collections in old space, however. Typically, this would happen if you had tuned the nursery size to be too large compared to your current heap size, and if your application's objects tended to survive for a long time (or if you have tuned your promotion rate "incorrectly"). In that case the old generation space would become too small to host all of your long-living objects and the old generation GC would struggle freeing up space to make room for promoted objects. In general, though, by having a generational garbage collector you will gain application performance and latency consistency.

在分代式垃圾回收中，老年代执行垃圾回收的平率较低，而年轻代中较高，垃圾回收的时间较短，侵入性也较低。但在某些情况下，年轻代的存在会是老年代的垃圾回收更加频繁。典型的例子是，相比于Java堆的大小，年轻代被设置的太大，而应用程序中对象的生命周期又很长（又或者给年轻代对象提升速率设了一个“不正确”的值）。在这种情况下，老年代因太小而放不下所有的存活对象，因此垃圾回收器就会忙于释放内存以便存放从年轻代提升上来的对象。但一般来说，使用分代式垃圾回收器可以使用应用程序的性能和系统延迟保持在一个合适的水平。

A nice side-effect of having a young generation is that fragmentation is somewhat addressed. Or rather, worst case scenario is postponed. The young, small objects that otherwise might cause fragments are cleaned out of the way. The old space also becomes more compact, because the long-living objects are more tightly allocated as they are moved to the older generation. Over time -- if you run long enough -- the old space will still become fragmented, however. In this case, you will end up with one or several full stop-the-world collections, potentially forcing the JVM to throw an _OutOfMemoryError_ or an error indicating failure to promote. Having a nursery postpones that worst-case scenario, however, and for some applications that is good enough. (In some case it will postpone the full-stop scenario to a point in the application's lifecycle where it doesn't matter anymore.) For most applications having a nursery functions as stop-gap, but it does decrease the frequency of stop-the-world GC and _OutOfMemoryError_ exceptions.

使用分代式垃圾回收器的一个额外效果是部分解决了碎片化的问题，或者说，发生最差情况的时间被推迟了。可能造成碎片的小对象被分配于年轻代，也在年轻代被释放掉。老年代中的对象分布会相对紧凑一些，因为这些对象在从年轻代中提升上来的时候会被会紧凑存放。但随着应用程序的运行，如果运行时间够长的话，老年代也会充满碎片的。这时就需要对年轻代和老年代执行一次或多次stop-the-world式的全垃圾回收，导致JVM抛出_OOM错误_或者表明提升失败的错误。但年轻代的存在使这种情况的出现被推迟了，对某些应用程序来说，这就就足够了。（在某些情况下，这种糟糕情况会被推迟到应用程序完全不关心GC的时候。）对大多数应用程序来说，对于大多数使用年轻代作为缓冲的应用程序来说，年轻代的存在可以降低出现stop-the-world式垃圾回收频率，减少抛出OOM错误的次数。


##Tuning generational GC##

##调优分代式垃圾回收##

As stated above, with the use of generations comes the responsibility and repetitive work of tuning young generation size and promotion rate for every new version of your application and for every load change. I can't stress enough the trade-off of specializing your runtime: by choosing a fixed number that optimizes for a specific load, you reduce your garbage collector's ability to respond dynamically to change -- and change is inevitable.

正如上面提到的，由于使用了分代式垃圾回收，你需要针对每个新版本的应用程序和不同的工作负载来调整年轻代大小和对象提升速度。我无法完整评估出固定运行时的代价：由于针对某个指定工作负载而设置了一系列优化参数，垃圾回收器应对动态变化的能力降低了，而变化是不可避免的。

My rule-of-thumb for tuning nursery size is that it should be as large as you can make it while ensuring the latency of the young space during stop-the-world collections. (Assuming that the young space is configured to use a parallel collector that is!) Keep in mind, too, that you need to leave enough old space in the heap for long-lived objects -- with margin! Here are some additional factors to consider when tuning a generational garbage collector:

对于调整年轻代大小来说，最重要的规则是要确保年轻代的大小不应该使因执行stop-the-world式垃圾回收而导致的暂停过长。（假设年轻代中使用的并行垃圾回收器。）还要记住的是，你要在堆中为老年代留出足够的空间来存放那些生命周期较长的对象。下面还有一些在调优分代式垃圾回收器时需要考虑的因素：

1. Most implementations of young space collection will ultimately result in a stop-the-world collection, and the larger the nursery, the longer the associated pause-time will be. So for applications that will be heavily impacted by that GC pause, you should carefully consider how large you want your young space to be.
2. You can mix GC algorithms for your heap generations. You could potentially make the young generation GC parallel, while using a concurrent algorithm for old space collection.
3. When you see frequent promotion failures it is usually a sign that your old space is fragmented. Promotion failure means that there isn't a large enough space in old space to fit a surviving object from young space. If this happens, consider tweaking the promotion rate (the age tuning option) or make sure your old-space GC algorithm is a compacting one (discussed in the next section) and tune the compaction in a way that fits your current application load. You may also increase the heap size and generation sizes, but doing so could impact the pause times of old space collections further -- remember that fragmentation is inevitable!
4. A generational collector work best for any Java application that has many short-lived small objects that will die within their first collection cycle. Generational GC works well to reduce fragmentation in this scenario, mainly by postponing it to a point in the application lifecycle where it may no longer be an issue.

1. 大多数年轻代垃圾回收都是stop-the-world式的，年轻代越大，相应的暂停时间越长。所以，对于那些受GC暂停影响较大的应用程序来说，应该仔细斟酌年轻代的大小。
2. 你可以综合考虑不同代的垃圾回收算法。可以在年轻代使用并行垃圾回收，而在老年代使用并行垃圾回收。
3. 当提升失败频繁发生时，这通常说明老年代中的碎片较多。提升失败指的是老年代中没有足够大的空间来存放年轻代中的存货对象。当出现提示失败时，你可以微调对象提升速率（即调整对象提升时年龄），或者确保老年代垃圾回收算法会将对象进行压缩（将在下一节讨论），并以一种适合当前应用程序工作负载的方式调整压缩。你也可以增大堆和各个代的大小，但这会使老年代垃圾回收的暂停时间延长——记住，碎片化是不可避免的。
4. 分代式垃圾回收最适用于那些具有大量短生命周期对象的应用程序，这些对象的生命周期短到活不过一次垃圾回收周期。在这种场景中，分代式垃圾回收可有效的减缓碎片化的趋势，主要是将碎片化随带来的影响推出到将来，而那时可能应用程序对此毫不关心。


#Compaction#

#压缩#

Although generational garbage collection postpones the worst-case scenario of fragmentation and _OutOfMemoryError_, compaction is the only real way of handling fragmentation. *Compaction* refers to a GC strategy of moving objects together to free up larger consecutive chunks of memory. Compaction thus creates large enough free memory to host new objects.

尽管分代式垃圾回收推出了碎片化和OOM错误出现的时机，但压缩仍然是唯一真正解决碎片化的方法。*压缩*是将对象移动到一起，以便释放掉大块连续内存空间的GC策略。因此，压缩可以生成足够大的空间来存放新创建的对象。

Moving objects and updating their references is a stop-the-world operation, which comes at a cost to Java performance. (This is true in all cases but one, which I'll discuss in the next article in this series.) The more popular (meaning referenced) an object is the longer the pause time will be. In a case where very little memory is left in the heap and the fragmentation situation is severe -- which usually becomes the case the longer an application runs -- compacting a popular area of the heap may mean seconds of pause time. Compacting the entire heap, which you would do if your system is close to running out of memory, could take tens of seconds.

移动对象并修改相关引用是一个stop-the-world式的操作，这会对应用程序的性能造成影响。（只有一种情况是个例外，将在本系列的下一篇文章中讨论。）存活对象越多，垃圾回收造成的暂停也越长。假如堆中的空间所剩无几，而且碎片化又比较严重（这通常是由于应用程序运行的时间很长了），那么对一块存活对象多的区域进行压缩可能会耗费数秒的时间。而如果因出现OOM而导致应用程序无法运行，因此而对整个堆进行压缩时，所消耗的时间可达数十秒。

The pause time for compaction is dependent on how much memory you need to move and how many references you need to update. With a larger heap size, it is statistically more likely that you will have a large number of both live objects with fragments between them and references needing to be updated. The observed average pause time per 1 to 2 GB live data is one second. So in a 4 GB heap, it is likely you will have at least 25 percent live data and will occasionally experience a near one-second pause time.

压缩导致的暂停时间的长短取决于需要移动的存活对象所占用的内存有多大以及有多少引用需要更新。当堆比较大时，从统计上讲，存活对象和需要更新的引用都会很多。从已观察到的数据看，每压缩1到2GB存活数据的需要约1秒钟。所以，对于4GB的堆来说，很可能会有至少25%的存活数据，从而导致约1秒钟的暂停。


##Compaction and the application memory wall##

##压缩与应用程序内存墙##

The _application memory wall_ refers to the maximum heap size you can set before GC pause time (i.e., compaction) interferes with your application enough to break a response-time SLA. Most Java applications today hit their application memory wall at between 4 GB and 20 GB per JVM, dependent on system and application. This is one reason that most enterprise applications are deployed in multiple smaller JVMs instead of using fewer larger (50- to 60-GB) instances. Let's think about this for a minute: Isn't it interesting how much of Java application design and deployment architecture in modern enterprises are defined by the limitations of compaction in the JVM? In this case, we've accepted multi-mini-instance deployments that are costly to manage over time, in order to work around the problem of stop-the-world interruptions needed to deal with fragmented heaps. This is particularly peculiar given how much large-memory capacity we have in modern hardware, and given the ever-increasing demand for more memory access in enterprise Java applications. Why should we settle for just a few gigabytes per instance? Concurrent compaction is an alternate approach that brings down the application memory wall, and will be the topic of my next article in this series.

_应用程序内存墙_涉及到在GC暂停时间对应用程序的影响大到无法达到满足预定需求之前所能设置的的堆的最大值。目前，大部分Java应用程序在碰到内存墙时，每个JVM实例的堆大小介于4GB到20GB之间，具体数值依赖于具体的环境和应用程序本身。这也是大多数企业及应用程序会部署多个小堆JVM而不是部署少数大堆（50到60GB）JVM的原因之一。在这里，我们需要思考一下：现代企业中有多少Java应用程序的设计与部署架构受制于JVM中的压缩？在这种情况下，我们接受多个小实例的部署方案，以增加管理维护时间为代价，绕开为处理充满碎片的堆而执行stop-the-world式垃圾回收所带来的问题。考虑到现今的硬件性能和企业级Java应用程序中对内存越来越多的访问要求，这种方案是在非常奇怪。为什么仅仅只能给每个JVM实例设置这么小的堆？并发压缩是一种可选方法，它可以降低内存墙带来的影响，这将是本系列中下一篇文章的主题。


>The observed average pause time per 1 to 2 GB live data is one second. So in a 4 GB heap, it is likely you will have at least 25 percent live data and will occasionally experience a near one-second pause time.

>从已观察到的数据看，每压缩1到2GB存活数据的需要约1秒钟。所以，对于4GB的堆来说，很可能会有至少25%的存活数据，从而导致约1秒钟的暂停。


#Conclusion: Reflection points and highlights#

#总结：回顾#

This article has been an overview of garbage collection, with the goal of refreshing your knowledge about the concepts and mechanics of garbage collection, as well as your awareness of the range of available options. Hopefully it also inspires you to further reading. Most of the options I've discussed are fairly traditional, in that they work implicitly with the limitations of the JVM. In my next article I'll introduce a newer concept, concurrent compaction, which is currently only implemented by Azul's Zing JVM. Concurrent compaction is one of an emerging class of GC techniques that seek to re-imagine the capacity of Java's memory model, particularly in light of today's increased memory and processor capacity.

本文对垃圾回收做了总体介绍，目的是为了使你能了解垃圾回收的相关概念和基本知识。希望本文能激发你继续深入阅读相关文章的兴趣。这里所介绍的大部分内容，它们。在下一篇文章中，我将介绍一些较新颖的概念，并发压缩，目前只有Azul公司的Zing JVM实现了这一技术。并发压缩是对GC技术的综合运用，这些技术试图重新构建Java内存模型，考虑当今内存容量与处理能力的不断提升，这一点尤为重要。

For now, I'll leave you with an overview of the main points about garbage collection discussed in this article:

现在，回顾一下本文中所介绍的关于垃圾回收的一些内容：

1. Different garbage collection algorithms and approaches will meet different application needs. Tracing collectors are most commonly used in commercial Java environments.
2. Parallel garbage collection uses available resources in parallel to perform GC. This tactic is usually implemented as a monolithic, stop-the-world collector, using all available system resources for a fast GC. Parallel GC thus provides higher throughput, but all application threads must wait until it's finished, which impacts latency.
3. Concurrent GC does its work while application threads are still running. The timing of concurrent GC is tricky because it needs to be finished before your application requires memory.
4. Generational garbage collection helps postpone fragmentation, but does not eliminate it. Generational GC divides the heap into two spaces, one for allocating young objects and one for objects that (being still referenced) have survived young-space GC. Use a generational collector for any Java application that has many short-lived small objects that will die within their first collection cycle.
5. Compaction is the only way to handle fragmentation completely. Most collectors have to perform compaction as a stop-the-world operation. The longer an application runs, the more reference complexity it will have and the more heterogeneous its object-size distribution will be. These factors will result in longer pauses to complete compaction. Larger heap size also impacts the compaction pause because there will likely be more live data and more references to update.
6. Tuning can help postpone OutOfMemoryErrors but the trade-off of too much tuning is rigidity. Be sure that you understand your production-load dynamics as well as your application's object types and reference profile before initiating tuning by trial-and-error. A too-rigid configuration will most likely break under dynamic production loads. Be sure to understand the consequences of a non-dynamic value before setting it.


1. 不同的垃圾回收算法的方式是为满足不同的应用程序需求而设计。目前在商业环境中，应用最为广泛的是引用跟踪垃圾回收器。
2. 并行垃圾回收器会并行使用可用资源执行垃圾回收任务。这种策略的常用实现是stop-the-world式垃圾回收器，使用所有可用系统资源快速完成垃圾回收任务。因此，并行垃圾回收可以提供较高的吞吐量，但在垃圾回收的过程中，所有应用程序线程都会被挂起，对延迟有较大影响。
3. 并发垃圾回收器可以与应用程序并发工作。使用并发垃圾回收器时要注意的是，确保在应用程序发生OOM错误之前完成垃圾回收。
4. 分代式垃圾回收可以推迟碎片化的出现，但并不能消除碎片化。它将堆分为两块空间，一块用于存放“年轻对象”，另一块用于存放从年轻代中存活下来的存活对象。对于那些使用了很多具有较短生命周期活不过几次垃圾回收周期的Java应用程序来说，使用分代式垃圾回收是非常合适的。
5. 压缩是可以完全解决碎片化的唯一方法。大多数垃圾回收器在压缩的时候是都stop-the-world式的。应用程序运行的时间越长，对象间的引就用越复杂，对象大小的异质性也越高。相应的，完成压缩所需要的时间也越长。如果堆的大小较大的话也会对压缩所占产生的暂停有影响，因为较大的堆就会有更多的活动数据和更多的引用需要处理。
6. 调优可以推迟OOM错误的出现，但过度调优是无意义的。在通过试错方式初始调优前，一定要明确生产环境负载的动态性，以及应用程序中的对象类型和对象间的引用情况。在动态负载下，过于刻板的配置很容会失效。在设置非动态调优选项前一定要清楚这样做后果。

Next month in the _JVM performance optimization series_: An in-depth look inside the Concurrent Continuously Compacting Collector (C4) GC algorithm.

下个月的_JVM性能调优系列_：深入了解C4垃圾回收器（Continuously Concurrent Compacting Collector）相关算法。


#关于作者#

Eva Andearsson对JVM计数、SOA、云计算和其他企业级中间件解决方案有着10多年的从业经验。在2001年，她以JRockit JVM开发者的身份加盟了创业公司Appeal Virtual Solutions（即BEA公司的前身）。在垃圾回收领域的研究和算法方面，EVA获得了两项专利。此外她还是提出了确定性垃圾回收（Deterministic Garbage Collection），后来形成了JRockit实时系统（JRockit Real Time）。在技术上，Eva与Sun公司和Intel公司合作密切，涉及到很多将JRockit产品线、WebLogic和Coherence整合的项目。2009年，Eva加盟了Azul System公司，担任产品经理。负责新的Zing Java平台的开发工作。最近，她改换门庭，以高级产品经理的身份加盟Cloudera公司，负责管理Cloudera公司Hadoop分布式系统，致力于高扩展性、分布式数据处理框架的开发。





#Resources#

#相关资源#

Earlier articles in the JVM performance optimization series:

JVM性能优化系列早期文章：

* ["JVM performance optimization, Part 1: Overview"][2] (August 2012)
* ["JVM performance optimization, Part 2: Compilers"][3] (September 2012)

Also on JavaWorld:

JavaWorld中的相关文章：

* ["Java's garbage-collected heap"][4] (Bill Venners, 1996)
* ["Trash talk: How Java recycles memory"][5] (Jeff Friesen, 2001)
* ["Generational garbage collection: Using the right HotSpot parameters"][6] (Ken Gottry, 2002)
* ["Reading GC logs: Report back from a JavaOne 2011 presentation"][7] (Dustin Marx, 2011)

Books about garbage collection:

垃圾回收的相关书籍：

* [Garbage Collection Algorithms Automatic Management][8] (Richard Jones, Rafael D. Lins; Wiley, August 1996)
* [The Garbage Collection Handbook][9] (Richard Jones, Eliot Moss; Chapman and Hall, August 2011).

JVM tuning and GC algorithms:

JVM调优与GC算法相关文章：

* ["Understanding the IBM Software Developers Kit (SDK) for Java: Memory Management"][10] (IBM Information Center).
* ["Java SE 6 HotSpot Virtual Machine Garbage Collection Tuning"][11] (Oracle Technology Network).
* ["The Azul Garbage Collector"][12] (Charles Humble, InfoQ, February 2012).





[1]:  http://docs.oracle.com/javase/specs/jvms/se7/html/index.html  "Java virtual machine specification"
[2]:  http://www.javaworld.com/javaworld/jw-08-2012/120821-jvm-performance-optimization-overview.html  "JVM performance optimization, Part 1: Overview"
[3]:  http://www.javaworld.com/javaworld/jw-09-2012/120905-jvm-performance-optimization-compilers.html  "JVM performance optimization, Part 2: Compilers"
[4]:  http://www.javaworld.com/javaworld/jw-08-1996/jw-08-gc.html  "Java's garbage-collected heap"
[5]:  http://www.javaworld.com/javaworld/jw-12-2001/jw-1207-java101.html?  "Trash talk: How Java recycles memory"
[6]:  http://www.javaworld.com/javaworld/jw-01-2002/jw-0111-hotspotgc.html  "Generational garbage collection: Using the right HotSpot parameters"
[7]:  http://www.javaworld.com/community/node/8155  "Reading GC logs: Report back from a JavaOne 2011 presentation"
[8]:  http://www.amazon.com/Garbage-Collection-Algorithms-Automatic-Management/dp/0471941484  "Garbage Collection Algorithms Automatic Management"
[9]:  http://www.amazon.com/The-Garbage-Collection-Handbook-Management/dp/1420082795  "The Garbage Collection Handbook"
[10]: http://publib.boulder.ibm.com/infocenter/java7sdk/v7r0/index.jsp?topic=%2Fcom.ibm.java.aix.70.doc%2Fdiag%2Funderstanding%2Fmemory_management.html  "Understanding the IBM Software Developers Kit (SDK) for Java: Memory Management"
[11]: http://www.oracle.com/technetwork/java/javase/gc-tuning-6-140523.html  "Java SE 6 HotSpot Virtual Machine Garbage Collection Tuning"
[12]: http://www.infoq.com/articles/azul_gc_in_detail  "The Azul Garbage Collector"
[13]: https://github.com/caoxudong/translation/blob/master/java/jvm/JVM_performance_optimization_Part_3_Garbage_collection.md#resources  "Resources"
[14]: http://www.spec.org/jbb2005/  "SPECjbb"
[15]: http://www.javaworld.com/javaworld/jw-10-2012/121010-jvm-performance-optimization-garbage-collection.html?page=1  "http://www.javaworld.com/javaworld/jw-10-2012/121010-jvm-performance-optimization-garbage-collection.html?page=1"
