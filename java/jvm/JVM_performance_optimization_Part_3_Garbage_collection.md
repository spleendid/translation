The Java platform's garbage collection mechanism greatly increases developer productivity, but a poorly implemented garbage collector can over-consume application resources. In this third article in the JVM performance optimization series, Eva Andreasson offers Java beginners an overview of the Java platform's memory model and GC mechanism. She then explains why fragmentation (and not GC) is the major "gotcha!" of Java application performance, and why generational garbage collection and compaction are currently the leading (though not most innovative) approaches to managing heap fragmentation in Java applications.


*Garbage collection (GC)* is the process that aims to free up occupied memory that is no longer referenced by any reachable Java object, and is an essential part of the Java virtual machine's (JVM's) dynamic memory management system. In a typical garbage collection cycle all objects that are still referenced, and thus reachable, are kept. The space occupied by previously referenced objects is freed and reclaimed to enable new object allocation.

In order to understand garbage collection and the various GC approaches and algorithms, you must first know a few things about the Java platform's memory model.


#Garbage collection and the Java platform memory model#

When you specify the startup option -Xmx on the command line of your Java application (for instance: java -Xmx:2g MyApp) memory is assigned to a Java process. This memory is referred to as the *Java heap* (or just *heap*). This is the dedicated memory address space where all objects created by your Java program (or sometimes the JVM) will be allocated. As your Java program keeps running and allocating new objects, the Java heap (meaning that address space) will fill up.

Eventually, the Java heap will be full, which means that an allocating thread is unable to find a large-enough consecutive section of free memory for the object it wants to allocate. At that point, the JVM determines that a garbage collection needs to happen and it notifies the garbage collector. A garbage collection can also be triggered when a Java program calls System.gc(). Using System.gc() does not guarantee a garbage collection. Before any garbage collection can start, a GC mechanism will first determine whether it is safe to start it. It is safe to start a garbage collection when all of the application's active threads are at a safe point to allow for it, e.g. simply explained it would be bad to start garbage collecting in the middle of an ongoing object allocation, or in the middle of executing a sequence of optimized CPU instructions (see my previous article on compilers), as you might lose context and thereby mess up end results.

A garbage collector should never reclaim an actively referenced object; to do so would break the [Java virtual machine specification][1]. A garbage collector is also not required to immediately collect dead objects. Dead objects are eventually collected during subsequent garbage collection cycles. While there are many ways to implement garbage collection, these two assumptions are true for all varieties. The real challenge of garbage collection is to identify everything that is live (still referenced) and reclaim any unreferenced memory, but do so without impacting running applications any more than necessary. A garbage collector thus has two mandates:





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
