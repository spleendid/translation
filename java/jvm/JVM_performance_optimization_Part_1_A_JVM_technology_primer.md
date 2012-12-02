JVM性能优化， Part 1 ―― JVM简介
================================================================

原文地址    [http://www.javaworld.com/javaworld/jw-08-2012/120821-jvm-performance-optimization-overview.html](http://www.javaworld.com/javaworld/jw-08-2012/120821-jvm-performance-optimization-overview.html)

众所周知，Java应用程序是运行在JVM上的，但是你对JVM有所了解么？作为这个系列文章的第一篇，本文将对经典Java虚拟机的运行机制做简单介绍，内容包括“一次编写，到处运行”的利弊、垃圾回收的基本原理、常用垃圾回收算法的示例和编译器优化等。后续的系列文章将会JVM性能优化的内容进行介绍，包括新一代JVM的设计思路，以及如何支持当今Java应用程序对高性能和高扩展性的要求。

如果你是一名程序员，那么毫无疑问，你肯定有过某种兴奋的感觉，就像是当一束灵感之光照亮了你思考方向，又像是神经元最终建立连接，又像是你解放思想开拓了新的局面。就我个人来说，我喜欢这种学习新知识的感觉。我在工作时就常常会有这种感觉，我的工作会涉及到一些JVM的相关技术，这着实令我兴奋，尤其是工作涉及到垃圾回收和JVM性能优化的时候。在这个系列中，我希望可以与你分享一些这方面的经验，希望你也会像我一样热爱JVM相关技术。

这个系列文章主要面向那些想要裂解JVM底层运行原理的Java程序员。文章立足于较高的层面展开讨论，内容涉及到垃圾回收和在不影响应用程序运行的情况下对安全快速的释放/分配内存。你将对JVM的核心模块有所了解：垃圾回收、GC算法、编译器行为，以及一些常用优化技巧。此外，还会讨论为什么对Java做基准测试（benchmark）是件很困难的事，并提供一些建议来帮助做基准测试。最后，将会介绍一些JVM和GC的前沿技术，内容涉及到Azul的Zing JVM，IBM JVM和Oracle的Garbage First（G1）垃圾回收器。

希望在阅读此系列文章后，你能对影响Java伸缩性的因素有所了解，并且知道这些因素是如何影响Java开发的，如何使Java难以优化的。希望会你有那种发自内心的惊叹，并且能够激励你为Java做一点事情：拒绝限制，努力改变。如果你还没准备好为开源事业贡献力量，希望本系列文章可以为你指明方向。

>*JVM职业生涯*
>
>在我职业生涯的早期，垃圾回收的问题曾经很难解决。垃圾回收问题和JVM的跨平台问题我更加为JVM和中间件的相关技术而着迷。我对JVM的热情源于十年前在[JRockit][1]团队工作的经历，当时要编码实现一种新的、能够自动学习、自动调优的垃圾回收算法（参见[相关资源][5]）。从那个项目开始，我踏上了JVM技术之旅，期间在BEA System公司工作的很多年，与Intel公司和Sun公司有过合作关系，在Oracle收购BEA公司和Sun公司之后为Oracle工作了一年。另外，我的硕士论文深入分析了JRockit的试验性特性，为[Deterministic Garbage Collection算法][2]打下了基础。当我加入Azul公司的团队后，我的工作陷入僵局，负责管理维护[Zing JVM][3]的垃圾回收算法（My work came full circle when I joined the team at Azul Systems and got to manage Zing JVM with its unique approach to garbage collection.）。现在我的工作有了一点小变化，负责日程安排与资源管理，关注分布式的可伸缩数据处理框架，目前在Cloudera公司工作，负责开源项目[Hadoop][4]的开发。


#Java的性能与“一次编写，到处运行”的挑战#

有不少人认为，Java平台本身就挺慢。其主要观点简单来说就是，Java性能低已经有些年头了 ―― 最早可以追溯到Java第一次用于企业级应用程序开发的时候。但这早就是老黄历了。事实是，如果你对不同的开发平台上运行简单的、静态的、确定性任务的运行结果做比较，你就会发现使用经过机器级优化（machine-optimized）代码的平台比任何使用虚拟环境进行运算的都要强，JVM也不例外。但是，在过去的10年中，Java的性能有了大幅提升。市场上不断增长的需求催生了垃圾回收算法的出现和编译技术的革新，在不断探索与优化的过程中，JVM茁壮成长。在这个系列文章中，我将介绍其中的一些内容。

JVM技术中最迷人的地方也正是其最具挑战性的地方：“一次编写，到处运行”。JVM并不对具体的用例、应用程序或用户负载进行优化，而是在应用程序运行过程中不断收集运行时信息，并以此为根据动态的进行优化。这种动态的运行时特性带来了很多动态问题。在设计优化方案时，以JVM为工作平台的程序无法依靠静态编译和可预测的内存分配速率（predictable allocation rates）对应用程序做性能评估，至少在对生产环境进行性能评估时是不行的。

机器级优化过的代码有时可以达到更好的性能，但它是以牺牲可移植性为代价的，在企业级应用程序中，动态负载和快速迭代更新是更加重要的。大多数企业会愿意牺牲一点机器级优化代码带来的性能，以此换取Java平台的诸多优势：

* 编码简单，易于实现（意味着可以更快的推向市场）
* 有很多非常有才的程序员
* 使用Java API和标准库实现快速开发
* 可移植性 ―― 无需为每个平台都编写一套代码


#从源代码到字节码#

作为一名Java程序员，你可以已经对编码、编译和运行这一套流程比较熟悉了。假如说，现在你写了一个程序代码MyApp.java，准备编译运行。为了运行这个程序，首先，你需要使用JDK内建的Java语言编译器，javac，对这个文件进行编译，它可以将Java源代码编译为字节码。javac将根据Java程序的源代码生成对应的可执行字节码，并将其保存为同名类文件：MyApp.class。在经过编译阶段后，你就可以在命令行中使用java命令或其他启动脚本载入可执行的类文件来运行程序，并且可以为程序添加启动参数。之后，类会被载入到运行时（这里指的是正在运行的JVM），程序开始运行。

上面所描述的就是在运行Java应用程序时的表面过程，但现在，我们要深入挖掘一下，在调用Java命令时，到底发生了什么？JVM到底是什么？大多数程序员是通过不断的调优，即使用相应的启动参数，与JVM进行交互，使Java程序运行的更快，同时避免程序出现“out of memory”错误。但你是否想过，为什么我们必须要通过JVM来运行Java应用程序呢？


#什么JVM

简单来说，JVM是用于执行Java应用程序和字节码的软件模块，并且可以将字节码转换为特定硬件和特定操作系统的本地代码。正因如此，JVM使Java程序做到了“一次编写，到处运行”。Java语言的可移植性是得到企业级应用程序开发者青睐的关键：开发者无需因平台不同而把程序重新编写一遍，因为有JVM负责处理字节码到本地代码的转换和平台相关优化的工作。

>基本上来说，JVM是一个虚拟运行环境，对于字节码来说就像是一个机器一样，可以执行任务，并通过底层实现执行内存相关的操作。

JVM也可以在运行java应用程序时，很好的管理动态资源。这指的是他可以正确的分配、回收内存，在不同的上维护一个具有一致性的线程模型，并且可以为当前的CPU架构组织可执行指令。JVM解放了程序员，使程序员不必再关系对象的生命周期，使程序员不必再关心应该在何时释放内存。而这，正是使用着类似C语言的非动态语言的程序员心中永远的痛。

你可以将JVM当做是一种专为Java而生的特殊的操作系统，它的工作是管理运行Java应用程序的运行时环境。简单来说，JVM就是运行字节码指令的虚拟执行环境，并且可以分配执行任务，或通过底层实现对内存进行操作。


#JVM组件简介

关于JVM内部原理与性能优化有很多内容可写。作为这个系列的开篇文章，我简单介绍JVM的内部组件。这个简要介绍对于那些JVM新手比较有帮助，也是为后面的深入讨论做个铺垫。


##从一种语言到另一种 ―― 关于Java编译器##

`编译器`以一种语言为输入，生成另一种可执行语言作为输出。Java编译器主要完成2个任务：

1. 实现Java语言的可移植性，不必局限于某一特定平台；
2. 确保输出代码可以在目标平台能够有效率的运行。

编译器可以是静态的，也可以是动态的。静态编译器，如javac，它以Java源代码为输入，将其编译为字节码（一种可以运行JVM中的语言）。*静态编译器*解释输入的源代码，而生成可执行输出代码则会在程序真正运行时用到。因为输入是静态的，所有输出结果总是相同的。只有当你修改的源代码并重新编译时，才有可能看到不同的编译结果。

*动态编译器*，如使用[Just-In-Time(JIT，即是编译)][5]技术的编译器，会动态的将一种编程语言编译为另一种语言，这个过程是在程序运行中同时进行的。JIT编译器会收集程序的运行时数据（在程序中插入性能计数器），再根据运行时数据和当前运行环境数据动态规划编译方案。动态编译可以生成更好的序列指令，使用更有效率的指令集合替换原指令集合，或剔除冗余操作。收集到的运行时数据的越多，动态编译的效果就越好；这通常称为代码优化或重编译。

动态编译使你的程序可以应对在不同负载和行为下对新优化的需求。这也是为什么动态编译器非常适合Java运行时。这里需要注意的地方是，动态编译器需要动用额外的数据结构、线程资源和CPU指令周期，才能收集运行时信息和优化的工作。若想完成更高级点的优化工作，就需要更多的资源。但是在大多数运行环境中，相对于获得的性能提升来说，动态编译的带来的性能损耗其实是非常小的 ―― 动态编译后的代码的运行效率可以比纯解释执行（即按照字节码运行，不做任何修改）快5到10倍。


#内存分配与垃圾回收#
`内存分配`是以线程为单位，在“Java进程专有内存地址空间”中，也就是Java堆中分配的。在普通的客户端Java应用程序中，内存分配都是单线程进行的。但是，在企业级应用程序和服务器端应用程序中，单线程内存分配却并不是个好办法，因为它无法充分利用现代多核时代的并行特性。

并行应用程序设计要求JVM确保多线程内存分配不会在同一时间将同一块地址空间分配给多个线程。你可以在整个内存空间中加锁来解决这个问题，但是这个方法（即所谓的“堆锁”）开销较大，因为它迫使所有线程在分配内存时逐个执行，对资源利用和应用程序性能有较大影响。多核程序的一个额外特点是需要有新的资源分配方案，避免出现单线程、序列化资源分配的性能瓶颈。

常用的解决方案是将堆划分为几个区域，每个区域都有适当的大小，当然具体的大小需要根据实际情况做相应的调整，因为不同应用程序之间，内存分配速率、对象大小和线程数量的差别是非常大的。Thread Local Allocation Buffer（TLAB），有时也称为Thraed Local Area（TLA），是线程自己使用的专用内存分配区域，在使用的时候无需获取堆锁。当这个区域用满的时候，线程会申请新的区域，直到堆中所有预留的区域都用光了。当堆中没有足够的空间来分配内存时，堆就“满”了，即堆上剩余的空间装不下待分配空间的对象。当堆满了的时候，垃圾回收就开始了。

#碎片化#

使用TLAB的一个风险是，由于堆上内存碎片的增加，使用内存的效率会下降。如果应用程序创建的对象的大小无法填满TLAB，而这块TLAB中剩下的空间又太小，无法分配给新的对象，那么这块空间就被浪费了，这就是所谓的“碎片”。如果“碎片”周围已分配出去的内存长时间无法回收，那么这块碎片研究长时间无法得到利用。

`碎片化`是指堆上存在了大量的`碎片`，由于这些小碎片的存在而使堆无法得到有效利用，浪费了堆空间。为应用程序设置TLAB的大小时，若是没有对应用程序中对象大小和生命周期和合理评估，导致TLAB的大小设置不当，就会是使堆逐渐碎片化。随着应用程序的运行，被浪费的碎片空间会逐渐增多，导致应用程序性能下降。这是因为系统无法为新线程和新对象分配空间，于是为防止出现OOM（out-of-memory）错误，而频繁GC的缘故。

对于TLAB产生的空间浪费这个问题，可以采用“曲线救国”的策略来解决。例如，可以根据应用程序的具体环境调整TLAB的大小。这个方法既可以临时，也可以彻底的避免堆空间的碎片化，但需要随着应用程序内存分配行为的变化而修改TLAB的值。此外，还可以使用一些复杂的JVM算法和其他的方法来组织堆空间来获得更有效率的内存分配行为。例如，JVM可以实现空闲列表（free-list），空闲列表中保存了堆中指定大小的空闲块。具有类似大小空闲块保存在一个空闲列表中，因此可以创建多个空闲列表，每个空闲列表保存某个范围内的空闲块。在某些事例中，使用空闲列表会比使用按实际大小分配内存的策略更有效率。线程为某个对象分配内存时，可以在空闲列表中寻找与对象大小最接近的空间块使用，相对于使用固定大小的TLAB，这种方法更有利于避免碎片化的出现。

>#GC往事#
>早期的垃圾回收器有多个老年代，但实际上，存在多个老年代是利大于弊的。

另一种对抗碎片化的方法是创建一个所谓的年轻代，在这个专有的堆空间中，保存了所有新创建的对象。堆空间中剩余的空间就是所谓的老年代。老年代用于保存具有较长生命周期的对象，即当对象能够挺过几轮GC而不被回收，或者对象本身很大（一般来说，大对象都具有较长的寿命周期）时，它们就会被保存到老年代。为了让你能够更好的理解这个方法，我们有必要谈谈垃圾回收。

#垃圾回收与应用程序性能#

垃圾回收就是JVM释放那些没有引用指向的堆内存的操作。当垃圾回收首次触发时，有引用指向的对象会被保存下来，那些没有引用指向的对象占用的空间会被回收。当所有可回收的内存都被回收后，这些空间就可以被分配给新的对象了。

垃圾回收不会回收仍有引用指向的对象；否则就会违反JVM规范。这个规则有一个例外，就是对软引用或弱引用的使用，当垃圾回收器发现内存快要用完时，会回收只有软引用或[弱引用][14]指向的对象所占用的内存。我的建议是，尽量避免使用弱引用，因为Java规范中存在的模糊的表述可能会使你对弱引用的使用产生误解。此外，Java本身是动态内存管理的，你没必要考虑什么时候该释放哪块内存。



The challenge for a garbage collector is to reclaim memory without impacting running applications more than necessary. If you don't garbage-collect enough, your application will run out of memory; if you collect too frequently you'll lose throughput and response time, which will negatively impact running applications.

#GC algorithms#

There are many different garbage collection algorithms. Later in this series we will deep dive into a few. On the highest level, the main two approaches to garbage collection are reference counting and tracing collectors.

* Reference counting collectors keep track of how many references an object has pointing to it. Once the count for an object becomes zero, the memory can immediately be reclaimed, which is one of the advantages of this approach. The difficulties with a reference-counting approach are circular structures and keeping all the reference counts up to date.
* Tracing collectors mark each object that is still referenced, iteratively following and marking all objects referenced by already marked objects. Once all still referenced objects are marked "live," all non-marked space can be reclaimed. This approach handles circular structures, but in most cases the collector has to wait until marking is complete before it can reclaim the unreferenced memory.

There are different ways to implement the above approaches. The more famous algorithms are marking or copying algorithms and parallel or concurrent algorithms. I'll talk about these in detail later in the series.

Generational garbage collection means dedicating separate address spaces on the heap for new objects and older ones. By "older objects" I mean objects that have survived a number of garbage collections. Having a young generation for new allocations and an old generation for surviving objects reduces fragmentation by quickly reclaiming memory occupied by short-lived objects, and by moving long-living objects closer together as they are promoted to the old generation address space. All of this reduces the the risk of fragments between long-living objects and protects the heap from fragmentation. A positive side-effect of having a young generation is also that it delays the need for the more costly collection of the old generation, as you are constantly reusing the same space for short-lived objects. (Old-space collection is more costly because the long-lived objects that live there contain more references to be traversed.)


A final algorithm improvement worth mentioning is compaction, which is a way to manage heap fragmentation. Compaction basically means moving objects together to free up larger consecutive chunks of memory. If you are familiar with disk fragmentation and the tools for handling it then you will find that compaction is similar, but works on Java heap memory. I'll discuss compaction in more detail later in this series.

#In conclusion: Reflection points and highlights#

A JVM enables portability ("write once, run anywhere") and dynamic memory management, both key features of the Java platform and reasons for its popularity and productivity.

In this first article in the JVM performance optimization series I've explained how a compiler translates bytecode to target-platform instruction languages and helps optimize the execution of your Java program `dynamically`. There are different compilers for different application needs.

I've also briefly discussed memory allocation and garbage collection, and how both relate to Java application performance. Basically, the higher the allocation rate of a Java application, the faster your heap fills up and the more frequently garbage collection is triggered. The challenge of garbage collection is to reclaim enough memory for your application needs without impacting running applications more than necessary, but to do so before the application runs out of memory. In future articles we'll explore the details of both traditional and more novel approaches to garbage collection for JVM performance optimization.

#About the author#

Eva Andreasson has been involved with Java virtual machine technologies, SOA, cloud computing, and other enterprise middleware solutions for 10 years. She joined the startup Appeal Virtual Solutions (later acquired by BEA Systems) in 2001 as a developer of the JRockit JVM. Eva has been awarded two patents for garbage collection heuristics and algorithms. She also pioneered Deterministic Garbage Collection which later became productized through JRockit Real Time. Eva has worked closely with Sun and Intel on technical partnerships, as well as various integration projects of JRockit Product Group, WebLogic, and Coherence (post Oracle acquisition in 2008). In 2009 Eva joined [Azul Systems][15] as product manager for the new Zing Java Platform. Recently she switched gears and joined the team at [Cloudera][16] as senior product manager for Cloudera's Hadoop distribution, where she is engaged in the exciting future and innovation path of highly scalable, distributed data processing frameworks.

[Read more about Core Java][17] in JavaWorld's Core Java section.


#相关资源

* ["To Colelct or Not To Collect"][7] (Eva Andreasson, Frank Hoffmann, Olof Lindholm; JVM-02: Proceedings of the Java Virtual Machine Research and Technology Symposium, 2002): 文章介绍了作者对自适应决策过程的研究，改过程用于确定应该使用哪种垃圾回收器技术，以及如何应用该技术。
* ["Reinforcement Learning for a dynamic JVM"][8] (Eva Andreasson, KTH Royal Institute of Technology, 2002): 一篇硕士论文，介绍了如何运用增强学习（reinforcement learning）优化决策，以决定对于一个动态工作负载来说，何时开始垃圾回收的决策更加合适。   
* ["Deterministic Garbage Collection: Unleash the Power of Java with Oracle JRockit Real Time"][9] (An Oracle White Paper, August 2008): 介绍了更多JRockit实时（JRockit Real Time）系统中Deterministic Garbage Collection算法的内容。
* ["Why is Java faster when using a JIT vs. compiling to machine code?"][10] (Stackoverflow, December 2009): 一个关于JIT的讨论。
* [Zing][11]: Zing是一个完整实现了Java相关规范，具有高伸缩性的软件平台，其中包含了应用程序级资源控制器、无损监控工具、以及诊断工具（这里原文是'includes an application-aware resource controller and zero overhead, always-on production visibility and diagnostic tools'，[Zing官网给出的描述][6]是'Zing also includes a runtime monitoring and diagnostics tool called Zing Vision. It is a zero overhead, always-on production time monitoring, diagnostic and tuning tool instrumented into the Zing JVM.'，怀疑是本文作者将"vision"和"visibility"弄混了）。 Zing整合了业界领先技术，使得每个JVM实例可以拥有TB级的堆内存，使其在动态负载和极限内存分配情况下仍可以保持较高的吞吐量 。
* ["G1: Java's Garbage First Garbage Collector"][12] (Eric Bruno, Dr. Dobb's, August 2009): 文章对GC做了回顾，并介绍了G1垃圾回收器。
* [Oracle JRockit: The Definitive Guide][13] (Marcus Hirt, Marcus Lagergren; Packt Publishing, 2010): JRcokit权威指南。




[1]: http://www.infoworld.com/d/developer-world/oracle-moving-merge-jrockit-hotspot-jvms-448  "JRockit" 
[2]: http://www.javaworld.com/javaworld/jw-08-2012/120821-jvm-performance-optimization-overview.html#resources  "Deterministic Garbage Collection算法"  
[3]: http://www.infoworld.com/d/developer-world/azul-systems-searches-managed-runtime-breakthroughs-228 "Zing JVM"  
[4]: http://www.infoworld.com/d/business-intelligence/cloudera-moves-hadoop-beyond-mapreduce-194941 "Hadoop"  
[5]: https://github.com/caoxudong/translation/blob/master/java/jvm/JVM_performance_optimization_Part_1_A_JVM_technology_primer.md#%E7%9B%B8%E5%85%B3%E8%B5%84%E6%BA%90  "相关资源"
[6]: http://www.azulsystems.com/products/zing/diagnostics "Zing官网给出的描述"
[7]: https://www.usenix.org/conference/java-vm-02/collect-or-not-collect-machine-learning-memory-management  "To Colelct or Not To Collect"
[8]: http://www.nada.kth.se/utbildning/grukth/exjobb/rapportlistor/2002/Rapporter02/andreasson_eva_02041.pdf  "Reinforcement Learning for a dynamic JVM"
[9]: http://www.oracle.com/us/technologies/java/oracle-jrockit-real-time-1517310.pdf  "Deterministic Garbage Collection: Unleash the Power of Java with Oracle JRockit Real Time"
[10]: http://stackoverflow.com/questions/1878696/why-is-java-faster-when-using-a-jit-vs-compiling-to-machine-code  "Why is Java faster when using a JIT vs. compiling to machine code?"
[11]: http://www.azulsystems.com/products/zing/virtual-machine  "Zing"
[12]: http://www.drdobbs.com/jvm/g1-javas-garbage-first-garbage-collector/219401061  "G1: Java's Garbage First Garbage Collector"
[13]: http://www.packtpub.com/oracle-jrockit-definitive-guide/book?tag=  "Oracle JRockit: The Definitive Guide"
[14]: http://java.sun.com/docs/books/performance/1st_edition/html/JPAppGC.fm.html  "weak reference"
[15]: http://www.azulsystems.com/  "Azul Systems"
[16]: http://www.cloudera.com/company/  "Cloudera"
[17]: http://www.javaworld.com/channel_content/jw-core-index.html  "Read more about Core Java"
