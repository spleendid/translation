JVM performance optimization, Part 4: C4 垃圾回收
========================================================


原文地址: [http://www.javaworld.com/javaworld/jw-11-2012/121107-jvm-performance-optimization-low-latency-garbage-collection.html][1]


*Learn how C4's concurrently compacting garbage collection algorithm helps boost Java scalability for low-latency enterprise Java applications, in this installment of Eva Andreasson's JVM performance optimization series.*

*作为Eva Andreasson的JVM性能优化系列的第4篇，本文将对C4垃圾回收器进行介绍。使用C4垃圾回收器可以有效提升对低延迟有要求的企业级Java应用程序的伸缩性。*


By now in this series it is obvious that I consider stop-the-world garbage collection to be a serious roadblock to Java application scalability, which is fundamental to modern Java enterprise development. Fortunately, some newer flavors of JVMs are finding ways to either fine-tune stop-the-world garbage collection or -- better yet -- do away with its lengthy pauses altogether. I am excited about new approaches to Java scalability that fully leverage the potential of multicore systems, where memory is cheap and plentiful.

到目前为止，本系列的文章将stop-the-world式的垃圾回收视为影响Java应用程序伸缩性的一大障碍，而伸缩性又是现代企业级Java应用程序开发的基础要求，因此这一问题亟待改善。幸运的是，针对此问题，JVM中已经出现了一些新特性，所使用的方式或是对stop-the-world式的垃圾回收做微调，或是消除冗长的暂停（这样更好些）。在一些多核系统中，内存不再是稀缺资源，因此，JVM的一些新特性就充分利用多核系统的潜在优势来增强Java应用程序的伸缩性。

In this article I focus primarily on the C4 algorithm, which is an upgrade of the Azul Systems Pauseless GC algorithm, currently implemented only for the Zing JVM. I also briefly discuss Oracle's G1 and IBM's Balanced Garbage Collection Policy algorithms. I hope that learning about these different approaches to garbage collection will expand your sense of what is possible with Java's memory-management model and with Java application scalability. Perhaps it will even inspire you to come up with some new innovative ideas for handling compaction? You will at least know more about your options when choosing a JVM, along with some basic guidelines for different application scenarios. (Note that this article focuses on low-latency and latency-sensitive Java applications.)

在本文中，我将着重介绍C4算法，该算法是Azul System公司中无暂停垃圾回收算法的新成果，目前只在Zing JVM上得到实现。此外，本文还将对Oracle公司的G1垃圾回收算法和IBM公司的Balanced Garbage Collection Policy算法做简单介绍。希望通过对这些垃圾回收算法的学习可以扩展你对Java内存管理模型和Java应用程序伸缩性的理解，并激发你对这方面内容的兴趣以便更深入的学习相关知识。至少，你可以学习到在选择JVM时有哪些需要关注的方面，以及在不同应用程序场景下要注意的事项。


#Concurrency in the C4 algorithm#

#C4算法中的并发性#

Azul Systems' Concurrent Continuously Compacting Collector (C4) algorithm takes an interesting and unique approach to low latency generational garbage collection. C4 is different from most generational garbage collectors because it is built on the assumptions that garbage is good -- meaning that applications generating garbage are doing good work -- and that compaction is inevitable. C4 is designed to satisfy varying and dynamic memory requirements, making it especially well-suited for long-running server-side applications.

Azul System公司的C4（Concurrent Continuously Compacting Collector，译者注，Azul官网给出的名字是Continuously Concurrent Compacting Collector）算法使用独一无二而又非常有趣的方法来实现低延迟的分代式垃圾回收。相比于大多数分代式垃圾回收器，C4的不同之处在于它认为垃圾回收并不是什么坏事（即应用程序产生垃圾很正常），而压缩是不可避免的。在设计之初，C4就是要牺牲各种动态内存管理的需求，以满足需要长时间运行的服务器端应用程序的需求。


>*jvm性能优化系列文章*
>
>jvm performance optimization, part 1: [概述][2]
>jvm performance optimization, part 2: [编译器][3]
>JVM performance optimization, Part 3: [垃圾回收][4]


The C4 algorithm decouples the process of freeing memory from application behavior and allocation rates. It is concurrent in a way that allows applications to run continuously without having to wait for the garbage collector to do its thing. This concurrency is key to providing consistently low pause times, regardless of how much live data is on the heap or how many references need to be traversed and updated during garbage collection. As I discussed in "[JVM performance optimization, Part 3][4]," most garbage collectors do stop-the-world compaction, which means that they experience increased pause times as the amount of live data and complexity on the heap increases. A garbage collector running the C4 algorithm does compaction concurrently with running application threads, thereby eliminating one of the biggest hurdles to JVM scalability.

C4算法将释放内存的过程从应用程序行为和内存分配速率中分离出来，并加以区分。这样就实现了并发运行，即应用程序可以持续运行，而不必等待垃圾回收的完成。其中的并发性是关键所在，正是由于并发性的存在才可以使暂停时间不受垃圾回收周期内堆上活动数据数量和需要跟踪与更新的引用数量的影响，将暂停时间保持在较低的水平。正如我在本系列[第3篇][4]中介绍的一样，大多数垃圾回收器在工作周期内都包含了stop-the-world式的压缩过程，这就是说应用程序的暂停时间会随活动数据总量和堆中对象间引用的复杂度的上升而增加。使用C4算法的垃圾回收器可以并发的执行压缩操作，即压缩与应用程序线程同时工作，从而解决了影响JVM伸缩性的最大难题。

Being concurrent, the C4 algorithm actually changes the premise of modern Java enterprise architecture and deployment models. Just consider what hundreds of GB per JVM instance could do for your Java applications:

实际上，为了实现并发性，C4算法改变了现代Java企业级架构和部署模型的基本假设。想象一下拥有数百GB内存的JVM会是什么样的：

* What does a Java deployment look like if it's able to scale within a single JVM instance rather than across many?
* What type of objects might be stored in memory that weren't before due to GC constraints?
* How might distributed clusters -- be they caches, region servers, or some other type of server nodes -- change as a result of larger JVM instances? What happens to traditional node counts, node deaths, and cache misses when the JVM size can increase without negatively impacting application responsiveness?

* 部署Java应用程序时，对伸缩性的考量无需要多个JVM配合，在单一JVM实例中即可完成。这时的部署是什么样呢？
* 有哪些以往因GC限制而无法在内存存储的对象？
* 那些分布式集群（如缓存服务器、区域服务器，或其他类型的服务器节点）会有什么变化？当可以增加JVM内存而不会对应用程序响应时间造成负面影响时，传统的节点数量、节点死亡和缓存丢失的计算会有什么变化呢？


#The three phases of the C4 algorithm#

#C4算法的3的阶段#

The main design premises of the C4 algorithm are "garbage is good" and "compaction is inevitable." C4's design goal is to be both concurrent and collaborative, thus eliminating the need for stop-the-world collection. The C4 garbage collection algorithm consists of three phases:

C4算法的一个基本假设是“垃圾回收不是坏事”和“压缩不可避免”。C4算法的设计目标是实现垃圾回收的并发与协作，剔除stop-the-world式的垃圾回收。C4垃圾回收算法包含一下3个阶段：

1. *Marking* -- finding what's live
2. *Relocation* -- moving things together to free up larger consecutive space (also known as compaction)
3. *Remapping* -- updating references to moved objects

1. *标记（Marking）* -- 找到活动对象
2. *重定位（Relocation）* -- 将存活对象移动到一起，以便可以释放较大的连续空间，这个阶段也可称为“压缩（compaction）”
3. *重映射（Remapping）* -- 更新被移动的对象的引用。

We'll look at each phase in detail.

下面的内容将对每个阶段做详细介绍。


##Marking in C4##

##C4算法中的标记阶段##

The *marking phase* of the C4 algorithm uses a concurrent marking and reference-tracing approach, which I discussed in detail in [Part 3][4] of this series.

在C4算法中，*标记阶段（marking phase）* 使用了并发标记（concurrent marking）和引用跟踪(reference-tracing)的方法来标记活动对象，这方面内容已经在本系列的[第3篇][4]中介绍过。

The marking phase is started by GC threads traversing the references from known live objects in thread stacks and registers. These threads continue to trace references until all reachable objects have been found on the heap. In this phase, the C4 algorithm is quite similar to other concurrent markers.

在标记阶段中，GC线程会从线程栈和寄存器中的活动对象开始，遍历所有的引用，标记找到的对象，这些GC线程会遍历堆上所有的可达（reachable）对象。在这个阶段，C4算法与其他并发标记器的工作方式非常相似。

C4's marker starts to differentiate during the concurrent mark phase. If an application thread hits an unmarked object during this phase, it facilitates that object to be queued up for further reference tracing. It also ensures that the object is marked, so that it never has to be traversed again by the garbage collector or another application thread. This saves marking time and eliminates the risk of recursive remarks. (Note that a long recursive mark would potentially force the application to run out of memory before memory could be reclaimed -- a pervasive problem in most garbage collection scenarios.)

C4算法的标记器与其他并发标记器的区别也是始于并发标记阶段的。在并发标记阶段中，如果应用程序线程修改未标记的对象，那么该对象会被放到一个队列中，以备遍历。这就保证了该对象最终会被标记，也因为如此，C4垃圾回收器或另一个应用程序线程不会重复遍历该对象。这样就节省了标记时间，消除了递归重标记（recursive remark）的风险。（注意，长时间的递归重标记有可能会使应用程序因无法获得足够的内存而抛出OOM错误，这也是大部分垃圾回收场景中的普遍问题。）

![Figure 1. Application threads traverse the heap just once during marking](images/jvmseries4-fig1.png?raw=true "Figure 1. Application threads traverse the heap just once during marking")

Figure 1. Application threads traverse the heap just once during marking

If the C4 algorithm relied on dirty-card tables or other methods of logging reads and writes into already-traversed heap areas, an application's GC threads might have to revisit certain areas for re-marking. In extreme cases a thread could get stuck in an infinite re-marking scenario -- at least infinite enough to cause the application to run out of memory before new memory could be freed. But C4 relies on a self-healing *load value barrier* (LVB), which enables application threads to immediately see if a reference is already marked. If the reference is not marked, the application thread will add it to the GC queue. Once the reference is in the queue it can't be re-marked. The application thread is free to continue on with its work.

如果C4算法的实现是基于脏卡表（dirty-card tables）或其他对已经遍历过的堆区域的读写操作进行记录的方法，那垃圾回收线程就需要重新访问这些区域做重标记。在极端条件下，垃圾回收线程会陷入到永无止境的重标记中 —— 至少这个过程可能会长到使应用程序因无法分配到新的内存而抛出OOM错误。但C4算法是基于*LVB（load value barrier）*实现的，LVB具有自愈能力，可以使应用程序线程迅速查明某个引用是否已经被标记过了。如果这个引用没有被标记过，那么应用程序会将其添加到GC队列中。一旦该引用被放入到队列中，它就不会再被重标记了。应用程序线程可以继续做它自己的事。

>*Dirty objects and card tables*
>
>Objects that need to be revisited by the garbage collector for some reason (such as if the object has been accessed and changed during a concurrent GC cycle) are often called dirty objects. References to dirty objects, or dirty areas of the heap, are usually managed in a separate data structure, known as a card table.

>*脏对象（dirty object）和卡表（card table）*
>
>由于某些原因（例如在一个并发垃圾回收周期中，对象被修改了），垃圾回收器需要重新访问某些对象，那么这些对象*脏对象（dirty object）*。这这些脏对象，或堆中脏区域的引用，通过会记录在一个专门的数据结构中，这就是卡表。

With C4 there is never a need for a re-marking phase: all reachable objects are marked in the initial round of traversing the heap. Because the runtime never has to re-mark, endless re-marking loops are eliminated, thus reducing the risk of the application running out of memory before unreferenced memory can be reclaimed.

在C4算法中，并没有重标记（re-marking）这个阶段，在第一次便利整个堆时就会将所有可达对象做标记。因为运行时不需要做重标记，也就不会陷入无限循环的重标记陷阱中，由此而降低了应用程序因无法分配到内存而抛出OOM错误的风险。


##Relocation in C4 -- where threads and GC collaborate##

##C4算法中的重定位 ——　应用程序线程与GC的协作##

The *relocation phase* of C4 is both collaborative and concurrent. This is because both GC and application threads are active concurrently, and because whichever thread first reaches an object to be moved can (collaboratively) facilitate that move. Application threads can thus smoothly continue their tasks, without having to wait for an entire garbage collection cycle to complete.

C4算法中，*重定位阶段（reloacation phase）*是由GC线程和应用程序线程以协作的方式，并发完成的。这是因为GC线程和应用程序线程会同时工作，而且无论哪个线程先访问将被移动的对象，都会以协作的方式帮助完成该对象的移动任务。因此，应用程序线程可以继续执行自己的任务，而不必等待整个垃圾回收周期的完成。

As Figure 2 shows, the live objects in a fragmented memory page are to be relocated. In the case of an application thread reaching a yet-to-be-moved object, the initial part of the move is facilitated by the application thread, so that it can quickly continue with its tasks. Virtual addresses (and thus references) are kept intact, so that memory can be immediately reclaimed.

正如Figure 2所示，碎片内存页中的活动对象会被重定位。在这个例子中，应用程序线程先访问了要被移动的对象，那么应用程序线程也会帮助完成移动该对象的工作的初始部分，这样，它就可以很快的继续做自己的任务。虚拟地址（指相关引用）可以完好无损的保留，内存也可以快速回收。

![Figure 2. A page selected for relocation and the empty new page that it will be moved to](images/jvmseries4-fig2.png?raw=true "Figure 2. A page selected for relocation and the empty new page that it will be moved to")

Figure 2. A page selected for relocation and the empty new page that it will be moved to

If a GC thread is assigned an object to be moved then there are no complications; the GC thread simply does the move. If an application thread tries to reach an object during the remapping phase (which is next), then it must check whether the object is to be moved. If so the application thread will escalate the relocation of the object so that it can continue with its work. (Larger objects are moved via shattered object moves. If you are interested in learning more about how shattered object moves work, I recommend reading the white paper "C4: The Continuously Concurrent Compacting Collector," listed in Resources.)

如果是GC线程先访问到了将被移动的对象，那就简单多了，GC线程会执行移动操作的。如果在重映射阶段（re-mapping phase，后续会提到）也访问这个对象，那么它必须检查该对象是否是要被移动的。如果是，那么应用程序线程会重新定位这个对象的位置，以便可以继续完成自己任务。（对大对象的移动是通过将该对象打碎再移动完成的。如果你对这部分内容感兴趣的话，推荐你阅读一下相关资源中的这篇白皮书“C4: The Continuously Concurrent Compacting Collector”）

Once all the live objects are completely moved out of a memory page, anything left behind is garbage. The page that the objects have moved out of is immediately available to be reclaimed, as you can see in the bottom display of Figure 2.

当所有的活动对象都从某个内存也中移出后，剩下的就都是垃圾数据了，这个内存页也就可以被整体回收了。正如Figure 2中所示。

>*What about sweep?*
>
>The C4 algorithm includes a mechanism that obviates the need for a sweep phase, thus eliminating an operation that is common to most GC algorithms. The virtual address space of the from page does have to be preserved until references to moved objects have been updated to point to their new location. So C4 implements a mechanism to guarantee that no virtual address space is unlocked until all references to that page are in a sane state. Then the algorithm is free to reclaim the physical memory page immediately.

>*关于清理*
>
>在C4算法中并不需要清理阶段（sweep phase），因此也就不需要这个在大多数垃圾回收算法中比较常用的操作。在指向被移动的对象的引用都更新为指向新的位置之前，from页中的虚拟地址空间必须被完整保留。所以C4算法的实现保证了，在所有指向这个页的引用处于稳定状态前，所有的虚拟地址空间都会被锁定。然后，算法会立即回收物理内存页。

Clearly there is great benefit to eliminating the need to stop the world in order to move objects together. As all live objects are concurrently moved during the relocation phase, they are also efficiently moved into adjacent addresses, where they end up fully compacted in the target page. Through concurrent relocation the heap is continuously compacted, without the need to ever stop all application threads at once. This approach to compaction removes the traditional limits to Java application access to memory (see [Part 1][2] for more about the Java application memory model).

很明显，无需执行stop-the-world式的移动对象是有很大好处的。由于在重定位阶段，所有活动对象都是并发移动的，因此它们可以被更有效率的放入到相邻的地址中，并且可以充分的压缩。通过并发执行重定位操作，堆被压缩为连续空间，也无需刮起所有的应用程序线程。这种方式消除了Java应用程序访问内存的传统限制（更多关于Java应用程序内存模型的内容参见[Part 1][2]）。

Having said all that, what about updating references? How is that not a stop-the-world operation?

经过上述的过程后，如何更新引用呢？如何实现一个非stop-the-world式的操作呢？


#Remapping in C4#

#C4算法中的重映射#

Some references to moved objects are automatically updated as part of an object relocation. The references to a relocated page are not touched during the relocation phase, however, so they still need to be updated. C4's *remapping phase* handles updating references that are still pointing to a page where live objects have been moved out. The remapping phase is also concurrent and collaborative.

在重定位阶段，某些指向被移动的对象的引用会自动更新。但是，在重定位阶段，指向重定位页中对象的引用并不会更新，所以它们需要在后续完成更新操作。C4算法中的*重映射阶段（re-mapping phase）*负责完成对那些活动对象已经移出，但仍指向那些的引用进行更新。当然，重映射也是一个协作式的并发操作。

In Figure 3, live objects have just been moved to a new memory page during the relocation phase. After relocation, the GC threads immediately start updating references to preserved virtual memory addresses, pointing them to the moved objects' new locations. The garbage collector continues this activity until all references are updated and the virtual memory space can be reclaimed.

在Figure 3中，在重定位阶段，活动对象已经被移动到了一个新的内存页中。在重定位之后，GC线程立即开始更新那些仍然指向之前的虚拟地址空间的引用，将它们指向那些被移动的对象的新地址。垃圾回收器会一直执行此项任务，知道所有的引用都被更新，这样虚拟内存空间就可以被整体回收了。

![Figure 3. Whatever thread finds an invalid address enables an update to the correct new address](images/jvmseries4-fig3.png?raw=true "Figure 3. Whatever thread finds an invalid address enables an update to the correct new address")

Figure 3. Whatever thread finds an invalid address enables an update to the correct new address

But what if an application thread tries to access a moved object before the GC has updated the reference? It's here that C4's ability to let the application thread collaboratively facilitate updates comes in handy. If an application thread hits this non-sane reference during a remapping phase, it will check to find out whether the reference needs to be updated. If it does, the application thread will retrieve the forwarding address and trigger an immediate reference update. Once that is done, it will smoothly resume its work.

但如果在GC完成对所有引用的更新之前，应用程序线程想要访问这些引用的话，会出现什么情况呢？在C4算法中，应用程序线程可以很方便的帮助完成对引用进行更新的工作。如果在重映射阶段，应用程序线程访问了处于非稳定状态的引用，它会找到该引用的正确指向。如果应用程序线程找到了正确的引用，它会更新该引用的指向。当完成更新后，应用程序线程会继续自己的工作。

The collaborative approach to remapping ensures that a reference only needs to be touched once to be updated. All subsequent reference calls will hit the new address. Additionally, a reference's forwarding addresses will not be stored in the previous object location, as is common with other GC implementations; instead it is stored in an off-heap structure. Rather than having to keep an entire page intact until all references are updated, memory can be instantly reclaimed.

协作式的重映射保证了引用只会被更新一次。该引用下的子引用都可以指向正确的新地址。此外，在大多数其他GC实现中，引用指向的地址不会被存储在该对象被移动之前的位置；相反，这些地址被存储在一个堆外结构（off-heap structure）中。无需在对所有引用的更新完成之前，再花费精力保持整个内存页完好无损，这个内存页可以被整体回收。


#So is C4 really pauseless?#

#C4算法真的是无暂停的么？#

A thread following a reference during the C4 remapping phase will be interrupted just once, and the interrupt will only last as long as a lookup and an update, after which the thread will be up and running. This approach to remapping is a huge improvement over other concurrent algorithms that have to run every single thread to a safe point, stop all threads at the same time, perform all reference updates, and only then release all threads.

在C4算法的重映射阶段，正在跟踪引用的线程仅会被中断一次，而这次中断仅仅会持续到检索和更新引用完成，在这次中断后，线程会继续运行。相比于其他并发算法来说，这种实现会带来巨大的性能提升，因为其他的并发立即回收算法需要等到每个线程都运行到一个安全点（safe point），然后同时挂起所有线程，再开始对所有的引用进行更新，完成后再恢复所有线程的运行。

For a concurrently compacting collector, pause time caused by garbage collection is never an issue. There is also no worse-case fragmentation scenario with C4's approach to relocation. A C4 garbage collector won't do back-to-back garbage collection cycles or stop the running application for seconds or even minutes at a time. If you ever did experience a stop-the-world scenario with this garbage collector, it would simply indicate that you had assigned your application too little memory. You can assign a garbage collector running the C4 algorithm as much memory as it needs, without ever having to worry about pause times.

对于并发压缩垃圾回收器来说，由于垃圾回收所引起的暂停永远不应该是个问题。在C4算法的重定位阶段汇总，也不会有再出现更糟的碎片化场景了。实现了C4算法的垃圾回收器也不会出现背靠背（back-to-back）式的垃圾回收周期，或者是因垃圾回收而使应用程序暂停数秒甚至数分钟。如果你曾经体验过这种stop-the-world式的垃圾回收，那么很有可能是你给应用程序设置的内存太小了。你可以试用一下实现了C4算法的垃圾回收器，并为其分配足够多的内存，而完全不必担心暂停时间过长的问题。


#Evaluating the C4 algorithm, and other alternatives#

#评估C4算法和其他可选方案#

As always, you should choose a JVM and garbage collector based on the needs of your application. The C4 algorithm is designed to guarantee consistently low pause times, no matter how much live memory occupies your heap, as long as you provide enough memory to your application. This makes C4 an excellent choice for latency-sensitive application environments deployed on modern hardware, which has plenty of RAM.

像往常一样，你需要针对应用程序的需求选择一款JVM和垃圾回收器。C4算法在设计之初就是无论堆中活动数据有多少，只要应用程序还有足够的内存可用，暂停时间都始终保持在较低的水平。正因如此，对于那些有大量内存可用，而对响应时间比较敏感的应用程来说，选择实现了C4算法的垃圾回收器正是不二之选。

C4 is a less optimal choice for client-side applications that run quickly, and usually within smaller heap sizes, with no issues. C4 is also not well-suited for applications that prioritize throughput (like static benchmarks). C4 really makes a difference for application scenarios that need to deploy 4 to 16 JVM instances per server in order to support the application load. C4 is also worth considering for an application scenario where you find yourself constantly tuning your garbage collector. Above all, consider C4 when response time is more important than throughput for your business use case. C4 is an ideal choice for applications that can't stop for a long time (be it downtime or GC pause time).

而对于那些要求快速启动，内存有限的客户端应用程序来说，C4算法并不是最好的选择。而对于那些对吞吐量有较高要求的应用程序来说，C4算法也并不适用。真正能够发挥C4算法威力的是那些为了提升应用程序工作负载而在每台服务器上部署了4到16个JVM实例的场景。此外，如果你经常要对垃圾回收器做调优的话，那么不妨考虑一下使用C4算法。综上所述，当响应时间比吞吐量占有更高的优先级时，C4算法是个不错的选择。而对那些不能接受长时间暂停的应用程序来说，C4算法是个理想的选择。

If you are considering using C4 for a production application, then you probably also want to rethink how you deploy. Instead of, for example, 16 JVM instances of 2-GB heap size each for your current application load per server, you could now consider one JVM with 64 GB (or two for failover). C4 wants as large a heap as possible to guarantee that a free page is always available for allocating threads. (Remember that memory is cheap!)

如果你正考虑在生产环境中使用C4，那么你可能还需要重新考虑一下如何部署应用程序。例如，不必为每个服务器配置16个具有2GB堆的JVM实例，而是使用一个64GB的JVM实例（或者增加一个作为备份）。C4算法需要尽可能大的内存来保证始终有一个空闲内存页来为新创建的对象分配内存。（记住，内存不再是昂贵的资源了！）

If you can't provide a server with 64 GB, 128 GB, or 1 TB RAM (or more), then a distributed multi-JVM deployment could be a better choice. In those cases you might consider using the Oracle HotSpot JVM's G1 garbage collector, or IBM JVM's Balanced Garbage Collection Policy. I'll briefly discuss both options below.

如果你没有64GB，128GB，或1TB（或更多）内存可用，那么分布式的多JVM部署可能是一个更好的选择。在这种场景中，你可以考虑使用Oracle HotSpot JVM的G1垃圾回收器，或者IBM JVM的平衡垃圾回收策略（Balanced Garbage Collection Policy）。下面将对这两种垃圾回收器做简单介绍。


#Garbage-First (G1) garbage collector#

#Gargabe-First （G1） 垃圾回收器#

G1 (Garbage-First) is a fairly new garbage collector that is part of the Oracle HotSpot JVM. G1 first appeared in the later versions of JDK 6. It is enabled by specifying -XX:+UseG1GC on your Oracle JDK startup command line.

G1垃圾回收器是新近才出现的垃圾回收器，是Oracle HotSpot JVM的一部分，在最近的JDK1.6版本中首次出现（译者注，该文章写于2012-07-11）。在启动Oracle JDK时附加命令行选项_-XX:+UseG1GC_，可以启动G1垃圾回收器。

Like C4, this mark-and-sweep collector offers an alternative approach to garbage collection in latency-sensitive applications. The G1 algorithm divides HotSpot's heap into fixed-size areas, onto which partial collection can be applied. It utilizes background threads to do heap marking concurrently with running application threads, which is similar to other concurrent marking algorithms.

与C4类似，这款标记-清理（mark-and-sweep）垃圾回收器也可作为对低延迟有要求的应用程序的备选方案。G1算法将堆分为固定大小区域，垃圾回收会作用于其中的某些区域。在应用程序线程运行的同时，启用后台线程，并发的完成标记工作。这点与其他并发标记算法相似。

G1's incremental approach results in shorter but more frequent pauses, which for some applications is enough to avoid long stop-the-world pauses. On the downside, as discussed in [Part 3][4], G1 does require you to spend time tuning the garbage collector for your current application load needs, and it does stop the world for GC interruptions. So G1 isn't a good fit for all low-latency applications. Moreover, the total time paused in G1 is high compared to in CMS, Oracle JVM's best-known concurrent mark-and-sweep collector.

G1增量方法可以是暂停时间更短，但更频繁，而这对一些力求避免长时间暂停的应用程序来说已经足够了。另一方面，正如在本系列的[Part 3][4]中介绍的，使用G1垃圾回收器需要针对应用程序的实际需求做长时间的调优，而其GC终端又是stop-the-world式的。所以对那些对低延迟有很高要求的应用程序来说，G1并不是一个好的选择。进一步说，从暂停时间总长来看，G1长于CMS（Oracle JVM中广为人知的并发垃圾回收器）。

G1 uses a copying algorithm (discussed in Part 3) for its partial collections. As a result, completely free areas are produced with each collection. The G1 garbage collector defines a set of areas as young space and the rest are designated as old space.

G1使用拷贝算法（在Part 3中介绍过）完成部分垃圾回收任务。这样，每次垃圾回收器后，都会产生完全可用的空闲空间。G1垃圾回收器定义了一些区域的集合作为年轻代，剩下的作为老年代。

G1 has received considerable attention and caused some hype, but it presents challenges in real-world deployments. Getting the tuning right is one -- recall that there is no "right tuning" for dynamic application loads. One issue is how to handle large objects that are close to the size of the partitions, because the left-over spaces cause fragmentation. There is also a performance tax associated with low-latency garbage collectors generally, which is that the collector must manage additional data structures. For me, the key issue of using G1 would be how to manage stop-the-world pauses. Stop-the-world pauses hinder any garbage collector's ability to scale with growing heap sizes and live data sizes, presenting a roadblock to Java enterprise scalability.

G1已经吸引了足够多的注意，引起了不小的轰动，但是它真正的挑战在于如何应对现实世界的需求。正确的调优就是其中一个挑战 —— 回忆一下，对于动态应用程序负载来说，没有永远“正确的调优”。一个问题是如何处理与分区大小相近的大对象，因为剩余的空间会成为碎片而无法使用。还有一个性能问题使用困扰着低延迟垃圾回收器，那就是垃圾回收器必须管理额外的数据结果。就我来说，使用G1的关键问题在于如何管理stop-the-world式垃圾回收器引起的暂停。Stop-the-world式的垃圾回收引起的暂停使任何垃圾回收器的能力都受制于堆大小和活动数据数量的增长，对企业级Java应用程序的伸缩性来说是一大困扰。


#IBM JVM Balanced Garbage Collection Policy#

#IBM JVM的平衡垃圾回收策略（Balanced Garbage Collection Policy）#

The IBM JVM Balanced Garbage Collection (BGC) Policy is enabled by specifying _-Xgcpolicy:balanced_ on your IBM JDK startup command line. BGC looks at first glance very much like G1. It splits the Java heap into many equal-sized areas called regions, each of which can be collected independently. Heuristics are applied to choose which regions to garbage-collect for the best return on effort. BGC's approach to generations is very similar to G1's.

IBM JVM的平衡垃圾回收（Balanced Garbage Collection BGC）策略通过在启动IBM JDK时指定命令行选项_-Xgcpolicy:balanced_来启用。乍一看，BGC很想G1。它也是将Java堆划分成相同大小的空间，称为区间（region），执行垃圾回收时会对每个区间单独回收。为了达到最佳性能，在选择要执行垃圾回收的区间时使用了一些启发性算法。BGC中关于代的划分也与G1相似。

IBM's Balanced Garbage Collection Policy is available only on 64-bit platforms. It is NUMA (Non-Uniform Memory Architecture) aware, and it is designed to work well with heap sizes over 4 GB. BGC's partial collections are mostly stop-the-world GCs, either due to the copying approach or to the need for compaction (which is a non-concurrent operation). So in the end BGC reproduces the tuning and scalability challenges found in G1 and other low-latency garbage collectors that don't implement concurrent compaction.

IBM的平衡垃圾回收策略仅在64位平台得到实现，是一种NUMA架构（Non-Uniform Memory Architecture），设计之初是为了用于具有4GB以上堆的应用程序。由于拷贝算法或压缩算法的需要，BGC的部分垃圾回收工作是stop-the-world式的，并非完全并发完成。所以，归根结底，BGC所遇到的问题与G1和其他没有实现并发压缩选法的垃圾回收器相似。


#In conclusion: Reflection points and highlights#

#结论：回顾#

C4 is a reference-tracing, generational, concurrent, and collaborative garbage collection algorithm. It is currently only implemented for Azul System's Zing JVM. The key values of the C4 algorithm are as follows:

C4是基于引用跟踪的、分代式的、并发的、协作式垃圾回收算法，目前只在Azul System公司的Zing JVM得到实现。C4算法的真正价值在于：

* No more re-marking loops means no more risk of running out of memory due during a marking phase.
* Compaction, automatically and continuously throughout the relocation phase eliminates the old rule: the more live data on the heap, the longer the compaction pause.
* No more stop-the-world garbage collection means significantly faster application response times
* No more sweep phase reduces the risk of running out of memory before the entire GC is finished and all free memory is reclaimed.
* Memory is immediately reclaimed on a page basis, making large spaces of memory continuously available for memory-hungry Java applications.

* 消除了重标记可能引起的无限循环，也就消除了在标记阶段出现OOM错误的风险。
* 压缩，以自动、且不断重定位的方式消除了固有限制：堆中活动数据越多，压缩所引起的暂停越长。
* 垃圾回收不再是stop-the-world式的，大大降低垃圾回收对应用程序响应时间造成的影响。
* 没有了清理阶段，降低了在完成GC之前就因为空闲内存不足而出现OOM错误的风险。
* 内存可以以页为单位立即回收，使那些需要使用较多内存的Java应用程序有足够的内存可用。


Concurrent compaction is what makes C4 unique. Letting application threads and GC threads collaboratively update object references, as they are discovered, ensures that your application will never be blocked until GC is finished. C4 fully decouples allocation rates from the ability to provide enough free and consecutive memory. The C4 algorithm enables you to make JVM instances as large as you need them, without worrying about pauses. Used appropriately, this is one JVM innovation that can bring low-latency Java applications up to speed with today's multicore and TB-size hardware.

并发压缩是C4独一无二的优势。使应用程序线程GC线程协作运行，保证了应用程序不会因GC而被阻塞。C4算法将内存分配和提供足够连续空闲内存的能力完全区分开。C4算法是你可以为JVM实例分配尽可能大的内存，而无需为应用程序暂停而烦恼。使用的当的话，这将是JVM的一项革新，它可以借助于当今的多核、TB级内存的硬件，将大大提升低延迟Java应用程序的运行速度。

G1 is a good alternative if you don't mind tuning and retuning, or frequent restarts, and if your application adapts well to a horizontal deployment model for scale, e.g., hundreds of small instances instead of few larger ones.

如果你不介意一遍又一遍的调优，以及频繁的重启的话，如果你的应用程序适用于水平部署模型的话（即部署几百个小堆JVM实例而不是几个大堆JVM实例），G1也是个不错的选择。

BGC is an innovative approach to dynamic low-latency heuristic adaption, something that JVM researchers have worked on for decades. This algorithm allows for slightly larger heap sizes. The downside of a dynamic self-tuning algorithm, is the instance when self-tuning can't keep up with sudden peaks and changes. You will still have to live with worst-case scenarios and allocate resources accordingly.

对于动态低延迟启发性自适应（dynamic low-latency heuristic adaption）算法而言，BGC是一项革新，JVM研究者对此已经研究了几十年。该算法可以应用于较大的堆。而动态自调优算法（ dynamic self-tuning algorithm）的缺陷是，它无法跟上突然出现的负载高峰。那时，你将不得不面对最糟糕的场景，并根据实际情况分配相关资源。

In the end the choice of best JVM and garbage collector for your application comes down to your priorities. What do you want to spend time and money on? From a purely technical angle,based on a decade of garbage collection experience, I am looking forward to seeing more innovation around concurrent compaction, or perhaps other approaches to moving objects or doing reallocation as a less intrusive operation. I think the key to Java enterprise scalability lies in concurrency.

最后，为你的应用程序选择最适合的JVM和垃圾回收器时，最重要的考虑因素是应用程序中吞吐量和暂停时间的优先级次序。你想把时间和金钱花在哪？从纯粹的技术角度说，基于我十年来对垃圾回收的经验，我一直在寻找更多关于并发压缩的革新性技术，或其他可以以较小代价完成移动对象或重定位的方法。我想影响企业级Java应用程序伸缩性的关键就在于并发性。




#关于作者#

Eva Andearsson对JVM计数、SOA、云计算和其他企业级中间件解决方案有着10多年的从业经验。在2001年，她以JRockit JVM开发者的身份加盟了创业公司Appeal Virtual Solutions（即BEA公司的前身）。在垃圾回收领域的研究和算法方面，EVA获得了两项专利。此外她还是提出了确定性垃圾回收（Deterministic Garbage Collection），后来形成了JRockit实时系统（JRockit Real Time）。在技术上，Eva与Sun公司和Intel公司合作密切，涉及到很多将JRockit产品线、WebLogic和Coherence整合的项目。2009年，Eva加盟了[Azul System][10]公司，担任产品经理。负责新的Zing Java平台的开发工作。最近，她改换门庭，以高级产品经理的身份加盟[Cloudera][11]公司，负责管理Cloudera公司Hadoop分布式系统，致力于高扩展性、分布式数据处理框架的开发。


#相关资源#

_JVM性能优化系列文章_

* "Java performance optimization, Part 1: [概述][2]" (August 2012)
* "Java performance optimization, Part 2: [编译器][3]" (September 2012)
* JVM performance optimization, Part 3: [垃圾回收][4]

_More about garbage collection:_

* "[C4: The Continuously Concurrent Compacting Collector][5]" (Gil Tene, Balaji Iyengar and Michael Wolf; Proceedings of the International Symposium on Memory Management, 2011): Learn more about the C4 algorithm and shattered object moves.
* ["Garbage-first garbage collection" (David Detlefs, et al., 2004, Proceedings of the 4th international Symposium on Memory Management, 2004): Learn more about the G1 algorithm. (Paid access on the ACM website.)][6]
* ["G1: Java's Garbage First Garbage Collector" (Eric J. Bruno, Dr. Dobb's, August 2009): A more in-depth overview and evaluation of G1.][7]
* The IBM Software Developers Kit (SDK) for Java documentation includes information about the [Balanced Garbage Collection Policy][8].
* "[Java VM: IBM vs Sun][9]" (Stackoverflow.com, December 2008): What factors might help you choose?




[1]:  http://www.javaworld.com/javaworld/jw-11-2012/121107-jvm-performance-optimization-low-latency-garbage-collection.html  "原文地址"
[2]:  http://www.javaworld.com/javaworld/jw-08-2012/120821-jvm-performance-optimization-overview.html  "Overview"
[3]:  http://www.javaworld.com/javaworld/jw-09-2012/120905-jvm-performance-optimization-compilers.html  "Compiler"
[4]:  http://www.javaworld.com/javaworld/jw-10-2012/121010-jvm-performance-optimization-garbage-collection.html  "Collection"
[5]:  http://www.azulsystems.com/products/zing/c4-java-garbage-collector-wp  "C4垃圾回收器"
[6]:  http://dl.acm.org/citation.cfm?id=1029879  "G1垃圾回收"
[7]:  http://www.drdobbs.com/jvm/g1-javas-garbage-first-garbage-collector/219401061  "G1垃圾回收器"
[8]:  http://publib.boulder.ibm.com/infocenter/java7sdk/v7r0/index.jsp?topic=%2Fcom.ibm.java.aix.70.doc%2Fdiag%2Funderstanding%2Fmm_gc_balanced.html  "平衡垃圾回收策略"
[9]:  http://stackoverflow.com/questions/338745/java-vm-ibm-vs-sun  "JVM对比：IBM vs. Sun"
[10]: http://www.azulsystems.com/  "Azul System"
[11]: http://www.cloudera.com/company/  "Cloudera"
