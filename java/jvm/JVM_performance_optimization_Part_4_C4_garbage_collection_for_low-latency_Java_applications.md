JVM performance optimization, Part 4: C4 garbage collection for low-latency Java applications
========================================================


原文地址: [http://www.javaworld.com/javaworld/jw-11-2012/121107-jvm-performance-optimization-low-latency-garbage-collection.html][1]


*Learn how C4's concurrently compacting garbage collection algorithm helps boost Java scalability for low-latency enterprise Java applications, in this installment of Eva Andreasson's JVM performance optimization series.*


By now in this series it is obvious that I consider stop-the-world garbage collection to be a serious roadblock to Java application scalability, which is fundamental to modern Java enterprise development. Fortunately, some newer flavors of JVMs are finding ways to either fine-tune stop-the-world garbage collection or -- better yet -- do away with its lengthy pauses altogether. I am excited about new approaches to Java scalability that fully leverage the potential of multicore systems, where memory is cheap and plentiful.

In this article I focus primarily on the C4 algorithm, which is an upgrade of the Azul Systems Pauseless GC algorithm, currently implemented only for the Zing JVM. I also briefly discuss Oracle's G1 and IBM's Balanced Garbage Collection Policy algorithms. I hope that learning about these different approaches to garbage collection will expand your sense of what is possible with Java's memory-management model and with Java application scalability. Perhaps it will even inspire you to come up with some new innovative ideas for handling compaction? You will at least know more about your options when choosing a JVM, along with some basic guidelines for different application scenarios. (Note that this article focuses on low-latency and latency-sensitive Java applications.)


#Concurrency in the C4 algorithm#

Azul Systems' Concurrent Continuously Compacting Collector (C4) algorithm takes an interesting and unique approach to low latency generational garbage collection. C4 is different from most generational garbage collectors because it is built on the assumptions that garbage is good -- meaning that applications generating garbage are doing good work -- and that compaction is inevitable. C4 is designed to satisfy varying and dynamic memory requirements, making it especially well-suited for long-running server-side applications.


>*JVM性能优化系列文章*
>
>JVM performance optimization, Part 1: [概述][2]
>JVM performance optimization, Part 2: [编译器][3]
>JVM performance optimization, Part 3: [垃圾回收][4]


The C4 algorithm decouples the process of freeing memory from application behavior and allocation rates. It is concurrent in a way that allows applications to run continuously without having to wait for the garbage collector to do its thing. This concurrency is key to providing consistently low pause times, regardless of how much live data is on the heap or how many references need to be traversed and updated during garbage collection. As I discussed in "[JVM performance optimization, Part 3][4]," most garbage collectors do stop-the-world compaction, which means that they experience increased pause times as the amount of live data and complexity on the heap increases. A garbage collector running the C4 algorithm does compaction concurrently with running application threads, thereby eliminating one of the biggest hurdles to JVM scalability.









#Resources#

_JVM性能优化系列文章_

* "Java performance optimization, Part 1: [Overview][2]" (August 2012)
* "Java performance optimization, Part 2: [Compilers][3]" (September 2012)
* JVM performance optimization, Part 3: [Garbage collection][4]

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
