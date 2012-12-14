JVM性能优化， Part 2 ―― 编译器
================================================================

原文地址    [http://www.javaworld.com/javaworld/jw-09-2012/120905-jvm-performance-optimization-compilers.html][1]


>作为[JVM性能优化][2]系列文章的第2篇，本文将着重介绍Java编译器。Eva Andreasson将对不同种类的编译器做介绍，并比较客户端、服务器端和层次编译产生的编译结果在性能上的区别，此外将对通用的JVM优化做介绍，包括死代码剔除、内联以及循环优化。

Java编译器是Java编程语言能独立于平台的根本原因。软件开发者可以尽全力编写程序，然后由Java编译器将源代码编译为针对于特定平台的高效、可运行的代码。不同类型的编译器适合于不同应用程序的需求，使编译结果可以满足期望的性能要求。对编译器基本原理了解得越多，在优化Java应用程序性能时就越能得心应手。

作为*JVM性能优化*系列文章的第2篇，本文着重介绍JVM中的各种编译器，此外还将对JIT编译器常用的一些优化措施进行讨论。（参见["JVM性能优化，Part 1"][3]中对JVM的介绍。）


#什么是编译器#

简单来说，编译器就是将一种编程语言作为输入，输出另一种可执行语言的工具。大家都熟悉的`javac`就是一个编译器，所有标准版的JDK中都带有这个工具。`javac`以Java源代码作为输入，将其翻译为可由JVM执行的字节码。翻译后的字节码存储在.class文件中，在启动Java进程的时候，被载入到Java运行时中。

普通CPU并不能识别字节码，它需要被转换为当前平台所能理解的本地指令。在JVM中，有专门的组件负责将字节码编译为平台相关指令，实际上，这也是一种编译器。有些JVM编译器可以处理多层级的编译工作，例如，编译器在最终将字节码转换为平台相关指令前，会为相关的字节码建立多层级的中间表示（intermediate representation）。

>*字节码与JVM*
>
>如果你想了解更多有关字节码与JVM的信息，请阅读 ["Bytecode basics"][4] (Bill Venners, JavaWorld)

以平台未知的角度看，我们希望尽可能的保持平台独立性，因此，最后一级的编译，也就是从最低级表示到实际机器码的转换，是与具体平台的处理器架构息息相关的。在最高级的表示上，会因使用静态编译器还是动态编译器而有所区别。在这里，我们可以选择应用程序所以来的可执行环境，期望达到的性能要求，以及我们所面临的资源限制。在本系列的第1篇文章的[静态编译器与动态编译器][5]一节中，已经对此有过简要介绍。我将在本文的后续章节中详细介绍这部分内容。


#静态编译器与动态编译器#

前文提到的javac就是使用静态编译器的例子。静态编译器解释输入的源代码，并输出程序运行时所需的可执行文件。如果你修改了源代码，那么就需要使用编译器来重新编译代码，否则输出的可执行性文件不会发生变化；这是因为静态编译器的输入是静态的普通文件。

使用静态编译器时，下面的Java代码

    static int add7( int x ) {
        return x+7;
    }

会生成类似如下的字节码：

    iload0
    bipush 7
    iadd
    ireturn

动态编译器会动态的将一种编程语言编译为另一种，即在程序运行时执行编译工作。动态编译与优化使运行时可以根据当前应用程序的负载情况而做出相应的调整。动态编译器非常适合于用于Java运行时中，因为Java运行时通常运行在无法预测而又会随着运行而有所变动的环境中。大部分JVM都会使用诸如Just-In-Time编译器的动态编译器。这里面需要注意的是，大部分动态编译器和代码优化有时需要使用额外的数据结构、线程和CPU资源。要做的优化或字节码上下文分析越高级，编译过程所消耗的资源就越多。在大多数运行环境中，相比于经过动态编译和代码优化所获得的性能提升，这些损耗微不足道。

>JVM的多样性与Java平台的独立性
>所有的JVM实现都有一个共同点，即它们都试图将应用程序的字节码转换为本地机器指令。一些JVM在载入应用程序后会解释执行应用程序，同时使用性能计数器来查找“热点”代码。还有一些JVM会调用解释执行的阶段，直接编译运行。资源密集型编译任务对应用程序来说可能会产生较大影响，尤其是那些客户端模式下运行的应用程序，但是资源密集型编译任务可以执行一些比较高级的优化任务。更多相关内容请参见[相关资源][7]
>
>如果你是Java初学者，JVM本身错综复杂结构会让你晕头转向的。不过，好消息是你无需精通JVM。JVM自己会做好代码编译和优化的工作，所以你无需关心如何针对目标平台架构来编写应用程序才能编译、优化，从而生成更好的本地机器指令。


#从字节码到可运行的程序#

当你编写完Java源代码并将之编译为字节码后，下一步就是将字节码指令编译为本地机器指令。这一步会由解释器或编译器完成。


##解释##

字节码编译的最简单形式称为解释。解释器查找每条字节码指令对应的硬件指令，再由CPU执行相应的硬件指令。

你可以将解释器想象为一个字典：每个单词（字节码指令）都有准确的解释（本地机器指令）。由于解释器每次读取一个字节码指令并立即执行，因此它就没有机会对某个指令集合进行优化。由于每次执行字节码时，解释器都需要做相应的解释工作，因此程序运行起来就很慢。解释执行可以准确执行字节码，但是未经优化而输出的指令集难以发挥目标平台处理器的最佳性能。


##编译##

另一方面，编译执行应用程序时，*编译器*会将加载运行时会用到的全部代码。因为编译器可以将字节码编译为本地代码，因此它可以获取到完整或部分运行时上下文信息，并依据收集到的信息决定到底应该如何编译字节码。编译器是根据诸如指令的不同执行分支和运行时上下文数据等代码信息来指定决策的。

当字节码序列被编译为机器代码指令集合时，就可以对这个指令集合做一些优化操作了，优化后的指令集合会被存储到成为code cache的数据结构中。当下一次执行这部分字节码序列时，就会执行这些经过优化后被存储到code cache的指令集合。在某些情况下，性能计数器会失效，并覆盖掉先前所做的优化，这时，编译器会执行一次新的优化过程。使用code cache的好处是优化后的指令集可以立即执行 —— 无需像解释器一样再经过查找的过程或编译过程！这可以加速程序运行，尤其是像Java应用程序这种同一个方法会被多次调用应用程序。


##优化##

随着动态编译器一起出现的是性能计数器。例如，编译器会插入性能计数器，以统计每个字节码块（对应与某个被调用的方法）的调用次数。在进行相关优化时，编译器会使用收集到的数据来判断某个字节码块有多“热”，这样可以最大程度的降低对当前应用程序的影响。

Along with dynamic compilation comes the opportunity to insert performance counters. The compiler might, for instance, insert a performance counter to count every time a bytecode block (e.g, corresponding to a specific method) was called. Compilers use data about how "hot" a given bytecode is to determine where in the code optimizations will best impact the running application. Runtime profiling data enables the compiler to make a rich set of code optimization decisions on the fly, further improving code-execution performance. As more refined code-profiling data becomes available it can be used to make additional and better optimization decisions, such as: how to better sequence instructions in the compiled-to language, whether to replace a set of instructions with more efficient sets, or even whether to eliminate redundant operations.


##Example##

Consider the Java code:

    static int add7( int x ) {
        return x+7;
    }

This could be statically compiled by javac to the bytecode:

    iload0
    bipush 7
    iadd
    ireturn

When the method is called the bytecode block will be dynamically compiled to machine instructions. When a performance counter (if present for the code block) hits a threshold it might also get optimized. The end result could look like the following machine instruction set for a given execution platform:

    lea rax,[rdx+7]
    ret


#Different compilers for different applications#

Different Java applications have different needs. Long-running enterprise server-side applications could allow for more optimizations, while smaller client-side applications may need fast execution with minimal resource consumption. Let's consider three different compiler settings and their respective pros and cons.


##Client-side compilers##

A well-known optimizing compiler is C1, the compiler that is enabled through the -client JVM startup option. As its startup name suggests, C1 is a client-side compiler. It is designed for client-side applications that have fewer resources available and are, in many cases, sensitive to application startup time. C1 use performance counters for code profiling to enable simple, relatively unintrusive optimizations.


##Server-side compilers##

For long-running applications such as server-side enterprise Java applications, a client-side compiler might not be enough. A server-side compiler like C2 could be used instead. C2 is usually enabled by adding the JVM startup option -server to your startup command-line. Since most server-side programs are expected to run for a long time, enabling C2 means that you will be able to gather more profiling data than you would with a short-running light-weight client application. So you'll be able to apply more advanced optimization techniques and algorithms.

>*Tip: Warm up your server-side compiler*
>
>For server-side deployments it may take some time before the compiler has optimized the initial "hot" parts of the code, so server-side deployments often require a "warm up" phase. Before doing any kind of performance measurement on a server-side deployment, make sure that your application has reached the steady state! Allowing the compiler enough time to compile properly will work to your benefit! (See the JavaWorld article "[Watch your HotSpot compiler go][6]" for more about warming up your compiler and the mechanics of profiling.)

A server compiler accounts for more profiling data than a client-side compiler does, and allows more complex branch analysis, meaning that it will consider which optimization path would be more beneficial. Having more profiling data available yields better application results. Of course, doing more extensive profiling and analysis requires expending more resources on the compiler. A JVM with C2 enabled will use more threads and more CPU cycles, require a a larger code cache, and so on.


##Tiered compilation##

*Tiered compilation* combines client-side and server-side compilation. Azul first made tiered compilation available in its Zing JVM. More recently (as of Java SE 7) it has been adopted by Oracle Java Hotspot JVM. Tiered compilation takes advantage of both client and server compiler advantages in your JVM. The client compiler is most active during application startup and handles optimizations triggered by lower performance-counter thresholds. The client-side compiler also inserts performance counters and prepares instruction sets for more advanced optimizations, which will be addressed at a later stage by the server-side compiler. Tiered compilation is a very resource-efficient way of profiling because the compiler is able to collect data during low-impact compiler activity, which can be used for more advanced optimizations later. This approach also yields more information than you'll get from using interpreted code profile counters alone.

The chart schema in Figure 1 depicts the performance differences between pure interpretation, client-side, server-side, and tiered compilation. The X-axis shows execution time (time unit) and the Y-axis performance (ops/time unit).

![Figure 1. Performance differences between compilers](images/jvmseries2-fig1.png?raw=true "Figure 1. Performance differences between compilers")

*Figure 1. Performance differences between compilers*


##Compiler performance comparisons##

Compared to purely interpreted code, using a client-side compiler leads to approximately 5 to 10 times better execution performance (in ops/s), thus improving application performance. The variation in gain is of course dependent on how efficient the compiler is, what optimizations are enabled or implemented, and (to a lesser extent) how well-designed the application is with regard to the target platform of execution. The latter is really something a Java developer should never have to worry about, though.

As compared to a client-side compiler, a server-side compiler usually increases code performance by a measurable 30 percent to 50 percent. In most cases that performance improvement will balance the additional resource cost.

Tiered compilation combines the best features of both compilers. Client-side compilation yields quick startup time and speedy optimization, while server-side compilation delivers more advanced optimizations later in the execution cycle.


#Some common compiler optimizations#

I've so far discussed the value of optimizing code and how and when common JVM compilers optimize code. I'll conclude with some of the actual optimizations available to compilers. JVM optimization actually happens at the bytecode level (or on lower representative language levels), but I'll demonstrate the optimizations using the Java language. I couldn't possibly cover all of the JVM optimizations in this section; rather, I mean to inspire you to explore on your own and learn about the hundreds of advanced optimizations and innovations in compiler technology (see [Resources][7]).


##Dead code elimination##

*Dead code elimination* is what it sounds like: the elimination of code that has never been called -- i.e., "dead" code. If a compiler discovers during runtime that some instructions are unnecessary, it will simply eliminate them from the execution instruction set. For example, in Listing 1 a certain value assignment for a variable is never used and can be fully ignored at execution time. On a bytecode level this could correspond to never needing to execute the load of the value into a register. Not having to do the load means less CPU time, and hence a quicker code execution, and therefore the application -- especially if the code is hot and called several times per second.

Listing 1 shows Java code exemplifying a variable that is never used, an unnecessary operation.

_Listing 1. Dead code_

    int timeToScaleMyApp(boolean endlessOfResources) {
    int reArchitect = 24;
    int patchByClustering = 15;
    int useZing = 2;
    
    if(endlessOfResources)
        return reArchitect + useZing;
    else
        return useZing;
    }

On a bytecode level, if a value is loaded but never used, the compiler can detect this and eliminate the dead code, as shown in Listing 2. Never executing the load saves CPU time and thus improves the program's execution speed.

_Listing 2. The same code following optimization_

    int timeToScaleMyApp(boolean endlessOfResources) {
    int reArchitect = 24;
    //unnecessary operation removed here...
    int useZing = 2;
    
    if(endlessOfResources)
        return reArchitect + useZing;
    else
        return useZing;
    }

Redundancy elimination is a similar optimization that removes duplicate instructions to improve application performance.


##Inlining##

Many optimizations try to eliminate machine-level jump instructions (e.g., JMP for x86 architectures). A jump instruction changes the instruction pointer register and thereby transfers the execution flow. This is an expensive operation relative to other ASSEMBLY instructions, which is why it is a common target to reduce or eliminate. A very useful and well-known optimization that targets this is called inlining. Since jumping is expensive, it can be helpful to inline many frequent calls to small methods, with different entry addresses, into the calling function. The Java code in Listings 3 through 5 exemplifies the benefits of inlining.

_Listing 3. Caller method_

    int whenToEvaluateZing(int y) {
        return daysLeft(y) + daysLeft(0) + daysLeft(y+1);
    }

_Listing 4. Called method_

    int daysLeft(int x){
    if (x == 0)
        return 0;
    else
        return x - 1;
    }

_Listing 5. Inlined method_

    int whenToEvaluateZing(int y){
        int temp = 0;
        
        if(y == 0) temp += 0; else temp += y - 1;
        if(0 == 0) temp += 0; else temp += 0 - 1;
        if(y+1 == 0) temp += 0; else temp += (y + 1) - 1;
        
        return temp; 
    }

In Listings 3 through 5 the calling method makes three calls to a small method, which we assume for this example's sake is more beneficial to inline than to jump to three times.

It might not make much difference to inline a method that is called rarely, but inlining a so-called "hot" method that is frequently called could mean a huge difference in performance. Inlining also frequently makes way for further optimizations, as shown in Listing 6.

_Listing 6. After inlining, more optimizations can be applied_

    int whenToEvaluateZing(int y){
        if(y == 0) return y;
        else if (y == -1) return y - 1;
        else return y + y - 1;
    }


##Loop optimization##

*Loop optimization* plays a big role when it comes to reducing the overhead that comes with executing loops. Overhead in this case means expensive jumps, number of checks of the condition, non-optimal instruction pipeline (i.e., an order of instructions that causes no-operations or extra cycles in the CPU). There are many kinds of loop optimizations, amounting to a vast set of optimizations. Notables include:

* Combining loops: When two nearby loops are iterated the same amount of times, the compiler can try to combine the bodies of the loops, to be executed at the same time (in parallel) in the case where nothing in the bodies reference each other, i.e., they are fully independent of each other.
* Inversion loops: Basically you replace a regular while loop with a do-while loop. And the do-while loop is set within an if clause. This replacement leads to two less jumps. However, it adds to the condition check and hence increases the code size. This optimization is an excellent example of how using slightly more resources leads to a more efficient code - a cost-gain balance the compiler has to evaluate and decide on dynamically during runtime.
* Tiling loops: Reorganizes the loop so that it iterates over blocks of data that are sized to fit in the cache.
* Unrolling loops: Reduces the number of times the loop condition has to be evaluated and also the number of jumps. You can think of this as "inlining" several iterations of the body to be executed without crossing the loop condition. Unrolling loops comes with risk, as it might decrease performance by impairing the pipeline and causing multiple redundant instruction fetches. Again, this is a judgment call by the compiler to make at runtime, i.e., if the gain is enough, the cost might be worth it.

This has been an overview of what a compiler does on a bytecode level (and below) to improve an application's execution performance on a target platform. The optimizations discussed are common and popular, but only a brief sampling of the available options. These have been very simple and broad explanations, which hopefully serve to pique your interest for more in-depth exploration. See Resources for further reading.


#In conclusion: Reflection points and highlights#

Use different compilers for different needs.

* Interpretation is the simplest form of bytecode translation to machine instructions, and works based on an instruction lookup table.
* Compilers allow for optimization based on performance counters, but will require some additional resources (code cache, optimization threads, etc.)
* Client-side compilers improve the performance of execution code by an order of magnitude (5 to 10 times better) when compared to interpreted code.
* Server-side compilers improve application performance by 30 percent to 50 percent over client-side compilers, but utilize more resources.
* Tiered compilation provides the best of two worlds. Enable client compilation to get your code performing well quickly, and server compilation over time, to make frequently called code execute even better.

There are many possible code optimizations. An important task for the compiler is to analyze all possibilities and weigh the cost of using an optimization against the execution speed benefit of the output machine code.


##About the author##

Eva Andreasson has been involved with Java virtual machine technologies, SOA, cloud computing, and other enterprise middleware solutions for 10 years. She joined the startup Appeal Virtual Solutions (later acquired by BEA Systems) in 2001 as a developer of the JRockit JVM. Eva has been awarded two patents for garbage collection heuristics and algorithms. She also pioneered Deterministic Garbage Collection which later became productized through JRockit Real Time. Eva has worked closely with Sun and Intel on technical partnerships, as well as various integration projects of JRockit Product Group, WebLogic, and Coherence (post Oracle acquisition in 2008). In 2009 Eva joined Azul Systems as product manager for the new Zing Java Platform. Recently she switched gears and joined the team at Cloudera as senior product manager for Cloudera's Hadoop distribution, where she is engaged in the exciting future and innovation path of highly scalable, distributed data processing frameworks.





#Resource#
"JVM performance optimization, Part 1: A JVM technology primer" (Eva Andreasson, JavaWorld, August 2012) launches the JVM performance optimization series with an overview of how a classic Java virtual machine works, including Java's write-once, run-anywhere engine, garbage collection basics, and some common GC algorithms.
See "Watch your HotSpot compiler go" (Vladimir Roubtsov, JavaWorld.com, April 2003) for more about the mechanics of hotspot optimization and why it pays to warm up your compiler.
If you want to learn more about bytecode and the JVM, see "Bytecode basics" (Bill Venners, JavaWorld, 1996), which takes an initial look at the bytecode instruction set of the Java virtual machine, including primitive types operated upon by bytecodes, bytecodes that convert between types, and bytecodes that operate on the stack.
The Java compiler javac is fully discussed in the formal Java platform documentation.
Get more of the basics of JVM (JIT) compilers, see the IBM Research Java JIT compiler page.
Also see Oracle JRockit's "Understanding Just-In-Time Compilation and Optimization" (Oracle? JRockit Introduction Release R28).
Dr. Cliff Click gives a complete tutorial on tiered compilation in his Azul Systems blog (July 2010).
Learn more about using performance counters for JVM performance optimization: "Using Platform-Specific Performance Counters for Dynamic Compilation" (Florian Schneider and Thomas R. Gross; Proceedings of the 18th international conference on Languages and Compilers for Parallel Computing, published by ACM Digital Library).
Oracle JRockit: The Definitive Guide (Marcus Hirt, Marcus Lagergren; Packt Publishing, 2010): A complete guide to the JRockit JVM.



[1]:  http://www.javaworld.com/javaworld/jw-09-2012/120905-jvm-performance-optimization-compilers.html  "http://www.javaworld.com/javaworld/jw-09-2012/120905-jvm-performance-optimization-compilers.html"
[2]:  http://www.javaworld.com/javaworld/jw-08-2012/120821-jvm-performance-optimization-overview.html  "Java性能优化"
[3]:  http://www.javaworld.com/javaworld/jw-08-2012/120821-jvm-performance-optimization-overview.html  "Java性能优化，第1部分"
[4]:  http://www.javaworld.com/javaworld/jw-09-1996/jw-09-bytecodes.html  "字节码基础"
[5]:  http://www.javaworld.com/javaworld/jw-08-2012/120821-jvm-performance-optimization-overview.html  "静态与动态编译器"
[6]:  http://www.javaworld.com/javaqa/2003-04/01-qa-0411-hotspot.html  "Watch your HotSpot compiler go"
[7]:  https://github.com/caoxudong/translation/blob/master/java/jvm/JVM_performance_optimization_Part_2_Compilers.md#resource "相关资源"
