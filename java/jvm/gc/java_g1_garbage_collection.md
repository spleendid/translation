G1: Java's Garbage First Garbage Collector
==========================================================

At JavaOne 2009, Sun released Java SE 6 Update 14, which included a version of the much-anticipated Garbage First (G1) garbage collector. G1 is a low-pause, low-latency, sometimes soft real-time, collector that allows you to set max pause time goals and collection intervals through suggestions on the Java VM command line. Although it cannot guarantee it, G1 will attempt to meet your goals, and hence introduce as little latency as possible into your application. This in turn may also make the VM run more predictably as it attempts to meet the pause time goals you provide.

在2009年的JavaOne大会上，Sun公司发布了Java SE 6 Update 14，在该版本中正式引入了万众期待的G1（Garbage First）垃圾回收器。G1是一款低暂停、低延迟、软实时的垃圾回收器，在使用该款垃圾回收器时可以用通过JVM命令行选项设置最大暂停时间和执行垃圾回收的间隔时间。尽管G1并不保证设置的参数能够完全起作用，但G1会尽力满足这些目标，以期可以尽可能降低应用程序的延迟。这样也可以使JVM在做垃圾回收可以更好预测垃圾回收的时长以满足预设的要求。


#What Is Garbage Collection?#

#什么是G1？#

Many dynamic languages, such as C, C++, Pascal, and so on, require you to manage memory explicitly. This includes memory allocation, de-allocation, and all of the accounting that occurs in between. In this time frame, you must be sure to not lose track of the memory (thereby failing to ever free it), or the result will be a memory leak. Just as dangerous is the attempt to use an object (or access memory) after it has been de-allocated, through what is called a dangling pointer. Either one of these situations can result in undefined behavior, the accidental overwriting of other data, a security hole, or an abrupt crash.

许多动态语言（译者注，这里应该是静态语言），例如c，c++，Pascal等，需要开发者显式的管理内存。管理内存包括了内存的分配、回收，以及这期间所有的相关工作。此时，开发者必须跟踪所有分配的内存，否则会无法及时释放它，进而造成内存泄漏。

Automatic memory management (garbage collection) removes the likelihood that these issues will occur since it's no longer left up to you to account for memory allocations. In C++, the concept of smart pointers is one solution, and in other languages, such as Lisp, SmallTalk, and Java, a full-featured garbage collector tracks the lifetimes of all objects in a running program. The history of garbage collection can be traced back to John McCarthy, who invented the concept as part of the Lisp programming language [McCarthy58].

In short, a garbage collector works to reclaim areas of memory within an application that will never be accessed again. At the most fundamental level, garbage collection involves two deceivingly simple steps:

Determine which objects can no longer be referenced by an application. This is done either via object reference counting, or object graphs (tracing). Reclaim the memory occupied by dead objects (the garbage).

Until recently, Java SE came with two main collectors: the parallel collector, and the concurrent-mark-sweep (CMS) collector -- see the sidebar Parallelism and Concurrency. As of the latest Java SE 6 update release, the G1 collector is another option. The plan is for G1 to eventually replace CMS as a low-pause, soft real-time collector. Let's take a look at how it works.

>Parallelism and Concurrency
>
>When speaking about garbage collection algorithms, parallelism describes the collector's ability to perform its work across multiple threads of execution. Concurrency describes its ability to do work while application threads are still running. Hence, a collector can be parallel but not concurrent, concurrent but not parallel, or both parallel and concurrent.
>
>The Java parallel collector (the default) is parallel but not concurrent as it pauses application threads to do its work. The CMS collector is parallel and partially concurrent as it pauses application threads at many points (but not all) to do its work. The G1 collector is fully parallel and mostly concurrent, meaning that it does pause applications threads momentarily, but only during certain phases of collection. For more information on garbage collection, and common algorithms used, read my latest book entitled Real-Time Java Programming with Java RTS, available from Pearson Publishing.
>
>―EJB


#How Does G1 Work?#

Garbage-First is a server-style garbage collector, targeted for multi-processors with large memories, that meets a soft real-time goal with high probability [Detlefs04]. It does this while also achieving high throughput, which is an important point when comparing it to other real-time collectors.

The G1 collector divides its work into multiple phases, each described below, which operate on a heap broken down into equally sized regions (see Figure 1). In the strictest sense, the heap doesn't contain generational areas, although a subset of the regions can be treated as such. This provides flexibility in how garbage collection is performed, which is adjusted on-the-fly according to the amount of processor time available to the collector.

![Figure 1: With garbage-first, the heap is broken into equally sized regions.](../../images/java_g1_garbage_collector-fig1.png?raw=true "Figure 1: With garbage-first, the heap is broken into equally sized regions")

Figure 1: With garbage-first, the heap is broken into equally sized regions.

Regions are further broken down into 512 byte sections called cards (see Figure 2). Each card has a corresponding one-byte entry in a global card table, which is used to track which cards are modified by mutator threads. Subsets of these cards are tracked, and referred to as Remembered Sets (RS), which is discussed shortly.

![Figure 2: Each region has a remembered set of occupied cards.](../../images/java_g1_garbage_collector-fig2.png?raw=true "Figure 2: Each region has a remembered set of occupied cards.")

Figure 2: Each region has a remembered set of occupied cards.

The G1 collector works in stages. The main stages consist of remembered set (RS) maintenance, concurrent marking, and evacuation pauses. Let's examine these stages now.


#RS Maintenance#

Each region maintains an associated subset of cards that have recently been written to, called the Remembered Set (RS). Cards are placed in a region's RS via a write barrier, which is an efficient block of code that all mutator threads must execute when modifying an object reference. To be precise, for a particular region (i.e., region a), only cards that contain pointers from other regions to an object in region a are recorded in region a's RS (see Figure 3). A region's internal references, as well as null references, are ignored.

![Figure 3: A region's RS tracks live references from outside the region.](../../images/java_g1_garbage_collector-fig3.png?raw=true "Figure 3: A region's RS tracks live references from outside the region.")

Figure 3: A region's RS tracks live references from outside the region.

In reality, each region's remembered set is implemented as a group of collections, with the dirty cards distributed amongst them according to the number of references contained within. Three levels of courseness are maintained: sparse, fine, and course. It's broken up this way so that parallel GC threads can operate on one RS without contention, and can target the regions that will yield the most garbage. However, it's best to think of the RS as one logical set of dirty cards, as the diagrams show.


#Concurrent Marking#

Concurrent marking identifies live data objects per region, and maintains the pointer to the next free byte, called top. There are, however, small stop-the-world pauses (described further below) that occur to ensure the correct heap state. A marking bitmap is maintained to create a summary view of the live objects within the heap. Each bit in the bitmap corresponds to one word within the heap (an area large enough to contain an object pointer; see Figure 4). A bit in the bitmap is set when the object it represents is determined to be a live object. In reality there are two bitmaps: one for the current collection, and a second for the previously completed collection. This is one way that changes to the heap are tracked over time.

![Figure 4: Live objects are indicated with a marking bitmap.](../../images/java_g1_garbage_collector-fig4.png?raw=true "Figure 4: Live objects are indicated with a marking bitmap.")

Figure 4: Live objects are indicated with a marking bitmap.

Marking is done in three stages:

* Marking Stage. The heap regions are traversed and live objects are marked:
    1. First, since this is the beginning of a new collection, the current marking bitmap is copied to the previous marking bitmap, and then the current marking bitmap is cleared.
    2. Next, all mutator threads are paused while the current TAMS pointer is moved to point to the same byte in the region as the top (next free byte) pointer.
    3. Next, all objects are traced from their roots, and live objects are marked in the marking bitmap. We now have a snapshot of the heap.
    4. Next, all mutator threads are resumed.
    5. Next, a write buffer is inserted for all mutator threads. This barrier records all new object allocations that take place after the snapshot into change buffers.
* Re-marking Stage. When the heap reaches a certain percentage filled, as indicated by the number of allocations since the snapshot in the Marking Stage, the heap is re-marked:
    1. As buffers of changed objects fill up, the contained objects are marked in the marking bitmap concurrently.
    2. When all filled buffers have been processed, the mutator threads are paused.
    3. Next, the remaining (partially filled) buffers are processed, and those objects are marked also.
* Cleanup Stage. When the Re-mark Stage completes, counts of live objects are maintained:
    1. All live objects are counted and recorded, per region, using the marking bitmap.
    2. Next, all mutator threads are paused.
    3. Next, all live-object counts are finalized per region.
    4. The TAMS pointer for the current collection is copied to the previous TAMS pointer (since the current collection is basically complete).
    5. The heap regions are sorted for collection priority according to a cost algorithm. As a result, the regions that will yield the highest numbers of reclaimed objects, at the smallest cost in terms of time, will be collected first. This forms what is called a collection set of regions.
    6. All mutator threads are resumed.

All of this work is done so that objects that are in the collection set are reclaimed as part of the evacuation process. Let's examine this process now.


#Evacuation and Collection#

This step is what it's all about -- reclaiming dead objects and shaping the heap for efficient object allocation. The collection set of regions (from the Concurrent Marking process defined above) forms a subset of the heap that is used during this process. When evacuation begins, all mutator threads are paused, and live objects are moved from their respective regions and compacted (moved together) into other regions. Although other garbage collectors might perform compaction concurrently with mutator threads, it's far more efficient to pause them. Since this operation is only performed on a portion of the heap -- it compacts only the collection set of regions -- it's a relatively quick, low-pause, operation. Once this phase completes, the GC cycle is complete.

To help limit the total pause time, much of the evacuation is done in parallel with multiple GC threads. The strategy for parallelization involves the following techniques:

* GC TLABS: The use of thread local allocation buffers (TLAB) for the GC threads eliminates memory-related contention amongst the GC threads. Forwarding pointers are inserted in the GC TLABs for evacuated live objects.
* Work Competition: GC threads compete to perform any of a number of GC-related tasks, such as maintaining remembered sets, root object scanning to determine reachability (dead objects are ignored), and evacuating live objects.
* Work Stealing: Part of mathematical systems theory, the work done by the GC threads is unsynchronized and executed arbitrarily by all of the threads simultaneously. This chaos-based algorithm equates to a group of threads that race to complete the list of GC-related tasks as quickly as they can without regard to one another. The end result, despite the apparent chaos, is a properly collected group of heap regions.

Note: The CMS and parallel collectors, described earlier, also use work competition and work stealing techniques to achieve greater efficiency.

#Conclusion#

The G1 collector is still considered experimental, but can be enabled in Java SE 6 Update 14 with the following two command-line parameters:

```shell
    -XX:+UnlockExperimentalVMOptions 
    -XX:+UseG1GC
```

Much of the G1 processing and behavior can be controlled by explicitly setting optional command-line parameters. See the sidebar entitled "Tuning the G1 Collector" to tune G1 behavior.

>*Tuning the G1 Collector*
>
>Let's review some command-line parameters that enable you to tune the behavior of G1. For instance, to suggest a GC pause time goal, use the following parameter:
>```shell
>    -XX:MaxGCPauseMillis=50 (for a target of 50 milliseconds)
>```
>With G1, a time interval can be specified during which a GC pause applies. In other words, no more than 50 milliseconds out of every second:
>```shell
>    -XX:GCPauseIntervalMillis=1000 (for a target of 1000 milliseconds)
>```
>Of course, these are only targets and there are no guarantees they will be met in all situations. However, G1 will attempt to meet these targets where it can.
>Alternatively, the size of the young generation can be specified explicitly to alter evacuation pause times:
>```shell
>     -XX:+G1YoungGenSize=512m (for a 512 megabyte young generation)
>```
>To run G1 at its full potential, add the following two parameters:
>```shell
>    -XX:+G1ParallelRSetUpdatingEnabled 
>    -XX:+G1ParallelRSetScanningEnabled
>```
>However, Sun warns that as of this version of G1, the use of these parameters may produce in a race condition and result in an error. However, it's worth a try to see if your application works safely with them set. If so, you'll benefit from the best GC performance that G1 can offer.
>
>―EJB

In terms of GC pause times, Sun states that G1 is sometimes better and sometimes worse than CMS. As G1 is still under development, the goal is to make G1 perform better than CMS and eventually replace it in a future version of Java SE (the current target is Java SE 7). While the G1 collector is successful at limiting total pause time, it's still only a soft real-time collector. In other words, it cannot guarantee that it will not impact the application threads' ability to meet its deadlines, all of the time. However, it can operate within a well-defined set of bounds that make it ideal for soft real-time systems that need to maintain high-throughput performance.

If your application requires guaranteed real-time behavior even with garbage collection, your only choice is a real-time garbage collector such as those that come with Sun's Java RTS or IBM's WebSphere RT products. However, if low pause times and soft real-time behavior is your goal, the G1 collector should suit it well.


#References#

* Bruno, Eric, and Bollella, Greg, Real-Time Java Development with Java RTS, Pearson Publishing, 2009
* Detlefs, et. al., [Garbage-First Garbage Collection][1], Sun Microsystems Research Laboratories, 2004.
* McCarthy, John, [LISP: A Programming System for Symbolic Manipulations][2], Communications of the ACM, 1958.




[1]:    http://research.sun.com/jtech/pubs/04-g1-paper-ismm.pdf    "Garbage-First Garbage Collection"
[2]:    http://portal.acm.org/citation.cfm?id=612201.612243        "LISP: A Programming System for Symbolic Manipulations"
