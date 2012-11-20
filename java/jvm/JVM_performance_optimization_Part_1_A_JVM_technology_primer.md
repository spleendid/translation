JVM性能优化， Part 1 —— JVM简介
================================================================

原文地址    [http://www.javaworld.com/javaworld/jw-08-2012/120821-jvm-performance-optimization-overview.html](http://www.javaworld.com/javaworld/jw-08-2012/120821-jvm-performance-optimization-overview.html)

众所周知，Java应用程序是运行在JVM上的，但是你对JVM有所了解么？作为这个系列文章的第一篇，本文将对经典Java虚拟机的运行机制做简单介绍，内容包括“一次编写，到处运行”的利弊、垃圾回收的基本原理、常用垃圾回收算法的示例和编译器优化等。后续的系列文章将会JVM性能优化的内容进行介绍，包括新一代JVM的设计思路，以及如何支持当今Java应用程序对高性能和高扩展性的要求。

如果你是一名程序员，那么毫无疑问，你肯定有过某种兴奋的感觉，就像是当一束灵感之光照亮了你思考进程，又像是神经元最终建立连接，又像是你解放思想开拓了新的局面。就我个人来说，我喜欢这种学习新知识的感觉。我在工作时就常常会有这种感觉，我的工作会涉及到一些JVM的相关技术，这着实令我兴奋，尤其是工作涉及到垃圾回收和JVM性能优化的时候。在这个系列中，我希望可以与你分享一些这方面的经验，希望你也会像我一样热爱JVM相关技术。

这个系列文章主要面向那些想要裂解JVM底层运行原理的Java程序员。文章立足于较高的层面展开讨论，内容涉及到垃圾回收和在不影响应用程序运行的情况下对安全快速的释放/分配内存。你将对JVM的核心模块有所了解：垃圾回收、GC算法、编译器行为，以及一些常用优化技巧。此外，还会讨论为什么对Java做基准测试（benchmark）是件很困难的事，并提供一些建议来帮助做基准测试。最后，将会介绍一些JVM和GC的前沿技术，内容涉及到Azul的Zing JVM，IBM JVM和Oracle的Garbage First（G1）垃圾回收器。

希望在阅读此系列文章后，你能对影响Java伸缩性的因素有所了解，并且知道这些因素是如何影响Java开发的，如何使Java难以优化的。希望会你有那种发自内心的惊叹，并且能够激励你为Java做一点事情：拒绝限制，努力改变。如果你还没准备好为开源事业贡献力量，希望本系列文章可以为你指明方向。

>*JVM职业生涯*
>
>在我职业生涯的早期，垃圾回收的问题曾经很难解决。垃圾回收问题和JVM的跨平台问题我更加为JVM和中间件的相关技术而着迷。我对JVM的热情源于十年前在[JRockit][1]团队工作的经历，当时要编码实现一种新的、能够自动学习、自动调优的垃圾回收算法（参见[相关资源][5]）。从那个项目开始，我踏上了JVM技术之旅，期间在BEA System公司工作的很多年，与Intel公司和Sun公司有过合作关系，在Oracle收购BEA公司和Sun公司之后为Oracle工作了一年。另外，我的硕士论文深入分析了JRockit的试验性特性，为[Deterministic Garbage Collection算法][2]打下了基础。当我加入Azul公司的团队后，我的工作陷入僵局，负责管理维护[Zing JVM][3]的垃圾回收算法（My work came full circle when I joined the team at Azul Systems and got to manage Zing JVM with its unique approach to garbage collection.）。现在我的工作有了一点小变化，负责日程安排与资源管理，关注分布式的可伸缩数据处理框架，目前在Cloudera公司工作，负责开源项目[Hadoop][4]的开发。


#Java的性能与“一次编写，到处运行”的挑战#

有不少人认为，Java平台本身就挺慢。其主要观点简单来说就是，Java性能低已经有些年头了 —— 最早可以追溯到Java第一次用于企业级应用程序开发的时候。但这早就是老黄历了。事实是，如果你对不同的开发平台上运行简单的、静态的、确定性任务的运行结果做比较，你就会发现使用经过机器级优化（machine-optimized）代码的平台比任何使用虚拟环境进行运算的都要强，JVM也不例外。但是，在过去的10年中，Java的性能有了大幅提升。市场上不断增长的需求催生了垃圾回收算法的出现和编译技术的革新，在不断探索与优化的过程中，JVM茁壮成长。在这个系列文章中，我将介绍其中的一些内容。

JVM技术中最迷人的地方也正是其最具挑战性的地方：“一次编写，到处运行”。JVM并不对具体的用例、应用程序或用户负载进行优化，而是在应用程序运行过程中不断收集运行时信息，并以此为根据动态的进行优化。这种动态的运行时特性带来了很多动态问题。在设计优化方案时，以JVM为工作平台的程序无法依靠静态编译和可预测的内存分配速率（predictable allocation rates）对应用程序做性能评估，至少在对生产环境进行性能评估时是不行的。

机器级优化过的代码有时可以达到更好的性能，但它是以牺牲可移植性为代价的，在企业级应用程序中，动态负载和快速迭代更新是更加重要的。大多数企业会愿意牺牲一点机器级优化代码带来的性能，以此换取Java平台的诸多优势：

* 编码简单，易于实现（意味着可以更快的推向市场）
* 有很多非常有才的程序员
* 使用Java API和标准库实现快速开发
* 可移植性 —— 无需为每个平台都编写一套代码



#From Java code to bytecode

As a Java programmer, you are probably familiar with coding, compiling, and executing Java applications. For the sake of example, let's assume that you have a program, MyApp.java and you want to run it. To execute this program you need to first compile it with javac, the JDK's built-in static Java language-to-bytecode compiler. Based on the Java code, javac generates the corresponding executable bytecode and saves it into a same-named class file: MyApp.class. After compiling the Java code into bytecode, you are ready to run your application by launching the executable class file with the java command from your command-line or startup script, with or without startup options. The class is loaded into the runtime (meaning the running Java virtual machine) and your program starts executing.

That's what happens on the surface of an everyday application execution scenario, but now let's explore what really happens when you call that java command. What is this thing called a Java virtual machine? Most developers have interacted with a JVM through the continuous process of tuning -- aka selecting and value-assigning startup options to make your Java program run faster, while deftly avoiding the infamous JVM "out of memory" error. But have you ever wondered why we need a JVM to run Java applications in the first place?


#What is a Java virtual machine?

Simply speaking, a JVM is the software module that executes Java application bytecode and translates the bytecode into hardware- and operating system-specific instructions. By doing so, the JVM enables Java programs to be executed in different environments from where they were first written, without requiring any changes to the original application code. Java's portability is key to its popularity as an enterprise application language: developers don't have to rewrite application code for every platform because the JVM handles the translation and platform-optimization.


>A JVM basically is a virtual execution environment acting as a machine for bytecode instructions, while assigning execution tasks and performing memory operations through interaction with underlying layers.

A JVM also takes care of dynamic resource management for running Java applications. This means it handles allocating and de-allocating memory, maintaining a consistent thread model on each platform, and organizing the executable instructions in a way that is suited for the CPU architecture where the application is executed. The JVM frees the programmer from keeping track of references between objects and knowing how long they should be kept in the system. It also frees us from having to decide exactly when to issue explicit instructions to free up memory -- an acknowledged pain point of non-dynamic programming languages like C.


You could think about the JVM as a specialized operating system for Java; its job is to manage the runtime environment for Java applications. A JVM basically is a virtual execution environment acting as a machine for bytecode instructions, while assigning execution tasks and performing memory operations through interaction with underlying layers.


#JVM components overview

There's a lot more to write about JVM internals and performance optimization. As foundation for upcoming articles in this series, I'll conclude with an overview of JVM components. This brief tour will be especially helpful for developers new to the JVM, and should prime your appetite for more in-depth discussions later in the series.

##From one language to another -- about Java compilers

A `compiler` takes one language as an input and produces an executable language as an output. A Java compiler has two main tasks:

1. Enable the Java language to be more portable, not tied into any specific platform when first written
2. Ensure that the outcome is efficient execution code for the intended target execution platform


Compilers are either static or dynamic. An example of a static compiler is javac. It takes Java code as input and translates it into bytecode -- a language that is executable by the Java virtual machine. *Static compilers* interpret the input code once and the output executable is in the form that will be used when the program executes. Because the input is static you will always see the same outcome. Only when you make changes to your original source and recompile will you see a different result.


*Dynamic compilers*, such as [Just-In-Time (JIT)][5] compilers, perform the translation from one language to another dynamically, meaning they do it as the code is executed. A JIT compiler lets you collect or create runtime profiling data (by the means of inserting performance counters) and make compiler decisions on the fly, using the environment data at hand. Dynamic compilation makes it possible to better sequence instructions in the compiled-to language, replace a set of instructions with more efficient sets, or even eliminate redundant operations. Over time you can collect more code-profiling data and make additional and better compilation decisions; altogether this is usually referred to as code optimization and recompilation.


Dynamic compilation gives you the advantage of being able to adapt to dynamic changes in behavior or application load over time that drive the need for new optimizations. This is why dynamic compilers are very well suited to Java runtimes. The catch is that dynamic compilers can require extra data structures, thread resources, and CPU cycles for profiling and optimization. For more advanced optimizations you'll need even more resources. In most environments, however, the overhead is very small for the execution performance improvement gained -- five or 10 times better performance than what you would get from pure interpretation (meaning, executing the bytecode as-is, without modification).


#Allocation leads to garbage collection#

`Allocation` is done on a per-thread basis in each "Java process dedicated memory address space," also known as the Java heap, or heap for short. Single-threaded allocation is common in the client-side application world of Java. Single-threaded allocation quickly becomes non-optimal in the enterprise application and workload-serving side, however, because it doesn't take advantage of the parallelism in modern multicore environments.


Parallell application design also forces the JVM to ensure that multiple threads do not allocate the same address space at the same time. You could control this by putting a lock on the entire allocation space. But this technique (a so-called heap lock) comes at a cost, as holding or queuing threads can cause a performance hit to resource utilization and application performance. A plus side of multicore systems is that they've created a demand for various new approaches to resource allocation in order to prevent the bottlenecking of single-thread, serialized allocation.

A common approach is to divide the heap into several partitions, where each partition is of a "decent size" for the application -- obviously something that would need tuning, as allocation rate and object sizes vary significantly for different applications, as well as by number of threads. A Thread Local Allocation Buffer (TLAB), or sometimes Thread Local Area (TLA), is a dedicated partition that a thread allocates freely within, without having to claim a full heap lock. Once the area is full, the thread is assigned a new area until the heap runs out of areas to dedicate. When there's not enough space left to allocate the heap is "full," meaning the empty space on the heap is not large enough for the object that needs to be allocated. When the heap is full, garbage collection kicks in.


#Fragmentation#

A catch with the use of TLABs is the risk of inducing memory inefficiency by fragmenting the heap. If an application happens to allocate object sizes that do not add up to or fully allocate a TLAB size, there is a risk that a tiny empty space too small to host a new object will be left. This leftover space is referred to as a "fragment." If the application also happens to keep references to objects that are allocated next to these leftover spaces then the space could remain un-used for a long time.

Fragmentation is what happens when fragments are scattered over the heap -- wasting heap space with tiny pieces of un-used memory. Configuring the "wrong" TLAB size for your application allocation behavior (with regard to object sizes and mix of object sizes and reference holding rate) is one path to an increasingly fragmented heap. As the application continues to run, the number of wasted fragmented spaces will come to hold an increasing portion of the free memory on the heap. Fragmentation causes decreased performance as the system is unable to allocate enough space for new application threads and objects. The garbage collector subsequently works harder to prevent out-of-memory exceptions.

TLAB waste can be worked around. One way to temporarily or completely avoid fragmentation is to tune TLAB size on a per-application basis. This approach typically requires re-tuning as soon as your application allocation behavior changes. It is also possible to use sophisticated JVM algorithms and other approaches to organize heap partitions for more efficient memory allocation. For instance, a JVM could implement free-lists, which are linked lists of free memory chunks of specific sizes. A consecutive free chunk of memory is linked to a linked list of other chunks of a similar size, thus creating a handful of lists, each with its own size range. In some case using free-lists leads to a better fitted-memory allocation approach. Threads allocating a certain-sized object are enabled to allocate it in chunks close to the object's size, generating potentially less fragmentation than if you just relied on fixed-sized TLABs.

>#GC trivia#
>
>Some early garbage collectors had multiple old generations, but it emerged that more than two old generation spaces resulted in more overhead than value.

Another way to optimize allocation versus fragmentation is to create a so-called young generation, which is a dedicated heap area (e.g., address space) where you will allocate all new objects. The rest of the heap becomes the so-called old generation. The old generation is left for the allocation of longer lived objects, meaning objects that have survived garbage collection or large objects that are assumed to live for a very long time. In order to better understand this approach to allocation we need to talk a bit about garbage collection.

#Garbage collection and application performance

Garbage collection is the operation performed by the JVM's garbage collector to free up occupied heap memory that is no longer referenced. When a garbage collection is first triggered, all objects that are still referenced are kept, and the space occupied by previously referenced objects is freed or reclaimed. When all reclaimable memory has been collected, the space is up for grabs and ready to be allocated again by new objects.


A garbage collector should never reclaim a referenced object; doing so would break the JVM standard specification. An exception to this rule is a soft or [weak reference][14] (if defined as such) that could be collected if the garbage collector were approaching a state of running out of memory. I strongly recommend that you try to avoid weak references as much as possible, however, because ambiguity in the Java specification has led to misinterpretation and error in their use. And besides, Java is designed for dynamic memory management, so you shouldn't have to think about where and when memory should be released.

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