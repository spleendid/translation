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

不同于引用计数垃圾回收器，引用跟踪垃圾回收器可以解决循环引用的问题。大多数垃圾回收器在标记阶段中，

Tracing collectors are most commonly used for memory management in dynamic languages; they are by far the most common for the Java language and have been commercially proven in production environments for many years. I'll focus on tracing collectors for the remainder of this article, starting with some of the algorithms that implement this approach to garbage collection.


#Tracing collector algorithms#

*Copying* and *mark-and-sweep* garbage collection are not new, but they're still the two most common algorithms that implement tracing garbage collection today.


##Copying collectors##

Traditional copying collectors use a *from-space* and a *to-space* -- that is, two separately defined address spaces of the heap. At the point of garbage collection, the live objects within the area defined as from-space are copied into the next available space within the area defined as to-space. When all the live objects within the from-space are moved out, the entire from-space can be reclaimed. When allocation begins again it starts from the first free location in the to-space.

In older implementations of this algorithm the from-space and to-space switch places, meaning that when the to-space is full, garbage collection is triggered again and the to-space becomes the from-space, as shown in Figure 1.


Figure 1. A traditional copying garbage collection sequence

![Figure 1. A traditional copying garbage collection sequence](images/jvmseries3-fig1.png?raw=true "Figure 1. A traditional copying garbage collection sequence")

More modern implementations of the copying algorithm allow for arbitrary address spaces within the heap to be assigned as to-space and from-space. In these cases they do not necessarily have to switch location with each other; rather, each becomes another address space within the heap.

One advantage of copying collectors is that objects are allocated together tightly in the to-space, completely eliminating fragmentation. Fragmentation is a common issue that other garbage collection algorithms struggle with; something I'll discuss later in this article.


##Downsides of copying collectors##

Copying collectors are usually *stop-the-world* collectors, meaning that no application work can be executed for as long as the garbage collection is in cycle. In a stop-the-world implementation, the larger the area you need to copy, the higher the impact on your application performance will be. This is a disadvantage for applications that are sensitive to response time. With a copying collector you also need to consider the worst-case scenario, when everything is live in the from-space. You always have to leave enough headroom for live objects to be moved, which means the to-space must be large enough to host everything in the from-space. The copying algorithm is slightly memory inefficient due to this constraint.


##Mark-and-sweep collectors##

Most commercial JVMs deployed in enterprise production environments run mark-and-sweep (or marking) collectors, which do not have the performance impact that copying collectors do. Some of the most famous marking collectors are CMS, G1, GenPar, and DeterministicGC (see [Resources][13]).

A *mark-and-sweep* collector traces references and marks each found object with a "live" bit. Usually a set bit corresponds to an address or in some cases a set of addresses on the heap. The live bit can, for instance, be stored as a bit in the object header, a bit vector, or a bit map.

After everything has been marked live, the sweep phase will kick in. If a collector has a sweep phase it basically includes some mechanism for traversing the heap again (not just the live set but the entire heap length) to locate all the non-marked chunks of consecutive memory address spaces. Unmarked memory is free and reclaimable. The collector then links together these unmarked chunks into organized free lists. There can be various free lists in a garbage collector -- usually organized by chunk sizes. Some JVMs (such as JRockit Real Time) implement collectors with heuristics that dynamically size-range lists based on application profiling data and object-size statistics.

When the sweep phase is complete allocation will begin again. New allocation areas are allocated from the free lists and memory chunks could be matched to object sizes, object size averages per thread ID, or the application-tuned TLAB sizes. Fitting free space more closely to the size of what your application is trying to allocate optimizes memory and could help reduce fragmentation.


>More about TLAB sizes
>
>TLAB and TLA (Thread Local Allocation Buffer or Thread Local Area) partitioning are discussed in [JVM performance optimization, Part 1][2].


##Downsides of mark-and-sweep collectors##

The mark phase is dependent on the amount of live data on your heap, while the sweep phase is dependent on the heap size. Since you have to wait until both the *mark* and *sweep* phases are complete to reclaim memory, this algorithm causes pause-time challenges for larger heaps and larger live data sets.

One way that you can help heavily memory-consuming applications is to use GC-tuning options that accommodate various application scenarios and needs. Tuning can, in many cases, help at least postpone either of these phases from becoming a risk to your application or service-level agreements (SLAs). (An SLA specifies that the application will meet certain application response times -- i.e., latency.) Tuning for every load change and application modification is a repetitive task, however, as the tuning is only valid for a specific workload and allocation rate.


##Implementations of mark-and-sweep##

There are at least two commercially available and proven approaches for implementing mark-and-sweep collection. One is the parallel approach and the other is the concurrent (or mostly concurrent) approach.


###Parallel collectors###

*Parallel collection* means that resources assigned to the process are used in parallel for the purpose of garbage collection. Most commercially implemented parallel collectors are monolithic stop-the-world collectors -- all application threads are stopped until the entire garbage collection cycle is complete. Stopping all threads allows all resources to be efficiently used in parallel to finish the garbage collection through the mark and sweep phases. This leads to a very high level of efficiency, usually resulting in high scores on throughput benchmarks such as [SPECjbb][14]. If throughput is essential for your application, the parallel approach is an excellent choice.

The cost of most parallel collection -- and do consider this, especially for production environments -- is that application threads cannot do any work during a GC, just like with copying collectors. Using a parallel collector that implements stop-the-world collection will have a major impact on response-time sensitive applications, especially if you have a lot of references to trace, which will happen with many live or complex data structures on the heap. (Remember that for mark-and-sweep collectors the time to free up new memory is dependent on the time it takes to trace the live data set plus the time to traverse the heap during the sweep phase.) For a monolithic parallel approach using all resources in parallel, this entire time will be a pause, and that pause corresponds to the entire GC cycle.


###Concurrent collectors###

A concurrent collector is a much better fit for applications that are sensitive to response time. Concurrent means that some (or most) garbage collection work is performed concurrently with the running application threads. As not all resources are used for GC, you will have the challenge of deciding when to start a garbage collection in order to allow enough time for the cycle to end. You need enough time to trace the live set and reclaim the memory before the application runs out of memory. If the garbage collection doesn't complete in time the application will throw an out-of-memory error. You don't want to do garbage collection all the time because that would consume application resources, thus impacting throughput. It can be extra tricky to keep that balance in very dynamic environments, so heuristics have been designed to determine when to start garbage collection and when to do various GC optimizing tasks and how much at a time, etc.

Another challenge is to determine when it is safe to perform operations that require a complete and true snapshot of the world -- for instance, you need to know when all live objects have been marked, and thus when to switch to the sweep phase. In the monolithic stop-the-world scenario employed by most parallel collectors, this *phase-switching* is less tricky because the world is already standing still. But in concurrent implementations it might not be safe to switch phases immediately. For instance, if an application has modified an area that has already been traced and marked by the collector, new or unmarked references may have been touched, which would make them live. In some implementations this situation will put your application at risk for long-time running re-marking loops, potentially making it hard for your application to get new free memory when it needs it.

The takeaway from this discussion so far is that you have numerous options among garbage collectors and GC algorithms, some better suited than others to specific application types and workloads. Not only are there different algorithms, but there are actually different implementations of the various algorithms. So it is wise to be informed about your application's allocation needs and characteristics before simply specifying a garbage collector on the command line. In the next section we'll look at some of the pitfalls of the Java platform memory model -- and by pitfalls I mean places where Java developers tend to make assumptions that lead to worse performance for dynamic production loads, not better.


#Why tuning doesn't replace garbage collection#

Most Java developers know that there are choices to be made if you want to maximize Java performance. The current variety of JVMs, garbage collectors, and the overwhelming selection of tuning options can lead developers to spend a lot of deployment time on the never-ending task of performance tuning. This has led some to conclude that GC is bad and that tuning so that GC happens infrequently or for just a short time is a successful workaround. But there are risks to doing GC this way.

Consider what it means to tune against specific application needs. Most tuning parameters -- such as allocation rate, object sizes, timing of response-time sensitive tasks, and how fast objects die -- are tuned specifically for the application's allocation rate, such as the test workload at hand. The end result could be either (or both) of these:

1. Things that worked during testing fail in production.
2. Workload or application changes require you to re-tune the application entirely.

Tuning is something that will always need to be repeated! Concurrent garbage collectors in particular can require a lot of tuning -- especially in production environments. You need the heuristics to match your specific application's needs for the expected worst-case load. The end result becomes a very rigid configuration, leading to a lot of resource waste. This approach to tuning (tuning to eliminate GC) is kind of a Don Quixote quest -- a chase after an imagined enemy with sure-to-lose cause. The fact is that the more you tune your collectors to fit a specific load, the farther you'll be from the dynamic features of a Java runtime. After all, how many applications really have a static load? And how reliably can you really predict for an expected load?

So, what if you didn't focus on tuning? What could you be doing differently to prevent out-of-memory errors and improve response times? The first step toward finding out is to identify the real challenge to Java application performance.


##Fragmentation##

The real Java performance challenge is not the garbage collector itself; it is fragmentation, and how the garbage collector deals with it. Fragmentation is a state of the heap where free memory is available but not in a big enough consecutive memory space to host a new object about to be allocated. As I mentioned in [Part 1][2], a fragment is either a residual space in the Java heap's TLABs, or (more often) a space previously occupied by small objects between longer living objects.

Over time, as an application runs, these fragments of unusable memory will appear all over the heap. In some cases the state will get worse with statically tuned options (like promotion rates, free lists, etc.) that can't keep up with the needs of a dynamic application. Such left-over (fragmented) space can't be used efficiently by an application. If you don't do anything about it you'll eventually end up with back-to-back garbage collections, the garbage collector's attempt to free up memory for a new object allocation request. In the worst-case scenario, even a number of consecutive GCs will not free up memory (too many fragments) and the JVM will be forced to throw an out-of-memory error. You can deal with fragmentation by restarting the application, which makes your heap a new space of consecutive memory mapped to your process, such that you'll be able to allocate objects freely again. Restarting leads to downtime, however, and besides the heap would eventually get fragmented again and you would need to do another restart.

_OutOfMemoryErrors_, hung processes, and logs showing that your garbage collector is working overtime are all symptoms of a GC struggling to free up memory, and are indicators of a fragmented heap. While some developers seek to resolve fragmentation by simply re-tuning their garbage collector, I argue that we should explore more innovative solutions to the problem of fragmentation before it becomes an issue for GC. The remainder of this article will focus on the two most effective approaches to managing fragmentation: generational garbage collection and compaction.


#Generational garbage collection#

You may have heard the theory that in production environments most objects die young. Generational garbage collection is an approach to GC that springs from this theory. In a generational garbage collection you basically divide the heap into different spaces (or generations), each covering different ages of objects, where age could mean the number of GC cycles that the object has survived (that is, how many times it has remained referenced).

When the space for new allocations, also known as *young space*, *nursery*, or *young generation*, is full, objects still referenced within that space are moved into the next generation. Most commonly there are only two generations. Usually generational garbage collectors are one-directional copying collectors, which I discussed earlier. Some more-recent implementations of young generation collectors are parallel collectors. It is also possible to implement different collecting algorithms for young space and old space. If you are using a parallel and/or copying collector then your young generation collectors will be a stop-the-world collector (see earlier explanation).

*Old space* or *old generation* hosts the promoted objects that stay referenced for a certain time or certain numbers of young space collections. On occasion, really large objects might be allocated directly into old space, as large objects are more costly to move.


##The mechanics of generational GC##

In a generational approach, garbage collection happens less frequently in old space and more frequent in young space, where the GC cycle is hopefully shorter and less intrusive. Having a young space can, on rare occasions, lead to more frequent collections in old space, however. Typically, this would happen if you had tuned the nursery size to be too large compared to your current heap size, and if your application's objects tended to survive for a long time (or if you have tuned your promotion rate "incorrectly"). In that case the old generation space would become too small to host all of your long-living objects and the old generation GC would struggle freeing up space to make room for promoted objects. In general, though, by having a generational garbage collector you will gain application performance and latency consistency.

A nice side-effect of having a young generation is that fragmentation is somewhat addressed. Or rather, worst case scenario is postponed. The young, small objects that otherwise might cause fragments are cleaned out of the way. The old space also becomes more compact, because the long-living objects are more tightly allocated as they are moved to the older generation. Over time -- if you run long enough -- the old space will still become fragmented, however. In this case, you will end up with one or several full stop-the-world collections, potentially forcing the JVM to throw an _OutOfMemoryError_ or an error indicating failure to promote. Having a nursery postpones that worst-case scenario, however, and for some applications that is good enough. (In some case it will postpone the full-stop scenario to a point in the application's lifecycle where it doesn't matter anymore.) For most applications having a nursery functions as stop-gap, but it does decrease the frequency of stop-the-world GC and _OutOfMemoryError_ exceptions.


##Tuning generational GC##

As stated above, with the use of generations comes the responsibility and repetitive work of tuning young generation size and promotion rate for every new version of your application and for every load change. I can't stress enough the trade-off of specializing your runtime: by choosing a fixed number that optimizes for a specific load, you reduce your garbage collector's ability to respond dynamically to change -- and change is inevitable.

My rule-of-thumb for tuning nursery size is that it should be as large as you can make it while ensuring the latency of the young space during stop-the-world collections. (Assuming that the young space is configured to use a parallel collector that is!) Keep in mind, too, that you need to leave enough old space in the heap for long-lived objects -- with margin! Here are some additional factors to consider when tuning a generational garbage collector:

1. Most implementations of young space collection will ultimately result in a stop-the-world collection, and the larger the nursery, the longer the associated pause-time will be. So for applications that will be heavily impacted by that GC pause, you should carefully consider how large you want your young space to be.
2. You can mix GC algorithms for your heap generations. You could potentially make the young generation GC parallel, while using a concurrent algorithm for old space collection.
3. When you see frequent promotion failures it is usually a sign that your old space is fragmented. Promotion failure means that there isn't a large enough space in old space to fit a surviving object from young space. If this happens, consider tweaking the promotion rate (the age tuning option) or make sure your old-space GC algorithm is a compacting one (discussed in the next section) and tune the compaction in a way that fits your current application load. You may also increase the heap size and generation sizes, but doing so could impact the pause times of old space collections further -- remember that fragmentation is inevitable!
4. A generational collector work best for any Java application that has many short-lived small objects that will die within their first collection cycle. Generational GC works well to reduce fragmentation in this scenario, mainly by postponing it to a point in the application lifecycle where it may no longer be an issue.


#Compaction#

Although generational garbage collection postpones the worst-case scenario of fragmentation and _OutOfMemoryError_, compaction is the only real way of handling fragmentation. *Compaction* refers to a GC strategy of moving objects together to free up larger consecutive chunks of memory. Compaction thus creates large enough free memory to host new objects.

Moving objects and updating their references is a stop-the-world operation, which comes at a cost to Java performance. (This is true in all cases but one, which I'll discuss in the next article in this series.) The more popular (meaning referenced) an object is the longer the pause time will be. In a case where very little memory is left in the heap and the fragmentation situation is severe -- which usually becomes the case the longer an application runs -- compacting a popular area of the heap may mean seconds of pause time. Compacting the entire heap, which you would do if your system is close to running out of memory, could take tens of seconds.

The pause time for compaction is dependent on how much memory you need to move and how many references you need to update. With a larger heap size, it is statistically more likely that you will have a large number of both live objects with fragments between them and references needing to be updated. The observed average pause time per 1 to 2 GB live data is one second. So in a 4 GB heap, it is likely you will have at least 25 percent live data and will occasionally experience a near one-second pause time.


##Compaction and the application memory wall##

The _application memory wall_ refers to the maximum heap size you can set before GC pause time (i.e., compaction) interferes with your application enough to break a response-time SLA. Most Java applications today hit their application memory wall at between 4 GB and 20 GB per JVM, dependent on system and application. This is one reason that most enterprise applications are deployed in multiple smaller JVMs instead of using fewer larger (50- to 60-GB) instances. Let's think about this for a minute: Isn't it interesting how much of Java application design and deployment architecture in modern enterprises are defined by the limitations of compaction in the JVM? In this case, we've accepted multi-mini-instance deployments that are costly to manage over time, in order to work around the problem of stop-the-world interruptions needed to deal with fragmented heaps. This is particularly peculiar given how much large-memory capacity we have in modern hardware, and given the ever-increasing demand for more memory access in enterprise Java applications. Why should we settle for just a few gigabytes per instance? Concurrent compaction is an alternate approach that brings down the application memory wall, and will be the topic of my next article in this series.


>The observed average pause time per 1 to 2 GB live data is one second. So in a 4 GB heap, it is likely you will have at least 25 percent live data and will occasionally experience a near one-second pause time.


#Conclusion: Reflection points and highlights#

This article has been an overview of garbage collection, with the goal of refreshing your knowledge about the concepts and mechanics of garbage collection, as well as your awareness of the range of available options. Hopefully it also inspires you to further reading. Most of the options I've discussed are fairly traditional, in that they work implicitly with the limitations of the JVM. In my next article I'll introduce a newer concept, concurrent compaction, which is currently only implemented by Azul's Zing JVM. Concurrent compaction is one of an emerging class of GC techniques that seek to re-imagine the capacity of Java's memory model, particularly in light of today's increased memory and processor capacity.

For now, I'll leave you with an overview of the main points about garbage collection discussed in this article:

1. Different garbage collection algorithms and approaches will meet different application needs. Tracing collectors are most commonly used in commercial Java environments.
2. Parallel garbage collection uses available resources in parallel to perform GC. This tactic is usually implemented as a monolithic, stop-the-world collector, using all available system resources for a fast GC. Parallel GC thus provides higher throughput, but all application threads must wait until it's finished, which impacts latency.
3. Concurrent GC does its work while application threads are still running. The timing of concurrent GC is tricky because it needs to be finished before your application requires memory.
4. Generational garbage collection helps postpone fragmentation, but does not eliminate it. Generational GC divides the heap into two spaces, one for allocating young objects and one for objects that (being still referenced) have survived young-space GC. Use a generational collector for any Java application that has many short-lived small objects that will die within their first collection cycle.
5. Compaction is the only way to handle fragmentation completely. Most collectors have to perform compaction as a stop-the-world operation. The longer an application runs, the more reference complexity it will have and the more heterogeneous its object-size distribution will be. These factors will result in longer pauses to complete compaction. Larger heap size also impacts the compaction pause because there will likely be more live data and more references to update.
6. Tuning can help postpone OutOfMemoryErrors but the trade-off of too much tuning is rigidity. Be sure that you understand your production-load dynamics as well as your application's object types and reference profile before initiating tuning by trial-and-error. A too-rigid configuration will most likely break under dynamic production loads. Be sure to understand the consequences of a non-dynamic value before setting it.

Next month in the _JVM performance optimization series_: An in-depth look inside the Concurrent Continuously Compacting Collector (C4) GC algorithm.


#关于作者#

Eva Andearsson对JVM计数、SOA、云计算和其他企业级中间件解决方案有着10多年的从业经验。在2001年，她以JRockit JVM开发者的身份加盟了创业公司Appeal Virtual Solutions（即BEA公司的前身）。在垃圾回收领域的研究和算法方面，EVA获得了两项专利。此外她还是提出了确定性垃圾回收（Deterministic Garbage Collection），后来形成了JRockit实时系统（JRockit Real Time）。在技术上，Eva与SUn公司和Intel公司合作密切，涉及到很多将JRockit产品线、WebLogic和Coherence整合的项目。2009年，Eva加盟了Azul System公，担任产品经理。负责新的Zing Java平台的开发工作。最近，她改换门庭，以高级产品经理的身份加盟Cloudera公司，负责管理Cloudera公司Hadoop分布式系统，致力于高扩展性、分布式数据处理框架的开发。





#Resources#

Earlier articles in the JVM performance optimization series:

* ["JVM performance optimization, Part 1: Overview"][2] (August 2012)
* ["JVM performance optimization, Part 2: Compilers"][3] (September 2012)

Also on JavaWorld:

* ["Java's garbage-collected heap"][4] (Bill Venners, 1996)
* ["Trash talk: How Java recycles memory"][5] (Jeff Friesen, 2001)
* ["Generational garbage collection: Using the right HotSpot parameters"][6] (Ken Gottry, 2002)
* ["Reading GC logs: Report back from a JavaOne 2011 presentation"][7] (Dustin Marx, 2011)

Books about garbage collection:

* [Garbage Collection Algorithms Automatic Management][8] (Richard Jones, Rafael D. Lins; Wiley, August 1996)
* [The Garbage Collection Handbook][9] (Richard Jones, Eliot Moss; Chapman and Hall, August 2011).

JVM tuning and GC algorithms:

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
