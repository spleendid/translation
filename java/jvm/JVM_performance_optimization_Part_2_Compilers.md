JVM性能优化， Part 2 ―― 编译器
================================================================

原文地址    [http://www.javaworld.com/javaworld/jw-09-2012/120905-jvm-performance-optimization-compilers.html][1]


>作为[JVM性能优化][2]系列文章的第2篇，本文将着重介绍Java编译器。Eva Andreasson将对不同种类的编译器做介绍，并比较客户端、服务器端和层次编译产生的编译结果在性能上的区别，此外将对通用的JVM优化做介绍，包括死代码剔除、内联以及循环优化。

A Java compiler is the source of Java's famous platform independence. A software developer writes the best Java application that he or she can, and then the compiler works behind the scenes to produce efficient and well-performing execution code for the intended target platform. Different kinds of compilers meet various application needs, thus yielding specific desired performance results. The more that you understand about compilers, in terms of how they work and what kinds are available, the more you'll be able to optimize Java application performance.

This second article in the *JVM performance optimization* series highlights and explains the differences between various Java virtual machine compilers. I'll also discuss some common optimizations used by Just-In-Time (JIT) compilers for Java. (See ["JVM performance optimization, Part 1"][3] for a JVM overview and introduction to the series.)


#What is a compiler?#

Simply speaking a compiler takes a programming language as an input and produces an executable language as an output. One commonly known compiler is `javac`, which is included in all standard Java development kits (JDKs). `javac` takes Java code as input and translates it into bytecode -- the executable language for a JVM. The bytecode is stored into .class files that are loaded into the Java runtime when the Java process is started.

Bytecode can't be read by standard CPUs and needs to be translated into an instruction language that the underlying execution platform can understand. The component in the JVM that is responsible for translating bytecode to executable platform instructions is yet another compiler. Some JVM compilers handle several levels of translation; for instance, a compiler might create various levels of intermediate representation of the bytecode before it turns into actual machine instructions, the final step of translation.

>*Bytecode and the JVM*
>
>If you want to learn more about bytecode and the JVM, see ["Bytecode basics"][4] (Bill Venners, JavaWorld).

From a platform-agnostic perspective we want to keep code platform-independent as far as possible, so that the last translation level -- from the lowest representation to actual machine code -- is the step that locks the execution to a specific platform's processor architecture. The highest level of separation is between static and dynamic compilers. From there, we have options depending on what execution environment we're targeting, what performance results we desire, and what resource restrictions we need to meet. I briefly discussed [static and dynamic compilers][5] in Part 1 of this series. In the following sections I'll explain a bit more.


#Static vs dynamic compilation#

An example of a static compiler is the previously mentioned javac. With static compilers the input code is interpreted once and the output executable is in the form that will be used when the program is executed. Unless you make changes to your original source and recompile the code (using the compiler), the output will always result in the same outcome; this is because the input is a static input and the compiler is a static compiler.

In a static compilation, the following Java code

    static int add7( int x ) {
        return x+7;
    }

would result in something similar to this bytecode:

    iload0
    bipush 7
    iadd
    ireturn

A dynamic compiler translates from one language to another dynamically, meaning that it happens as the code is executed -- during runtime! Dynamic compilation and optimization give runtimes the advantage of being able to adapt to changes in application load. Dynamic compilers are very well suited to Java runtimes, which commonly execute in unpredictable and ever-changing environments. Most JVMs use a dynamic compiler such as a Just-In-Time (JIT) compiler. The catch is that dynamic compilers and code optimization sometimes need extra data structures, thread, and CPU resources. The more advanced the optimization or bytecode-context analyzing, the more resources are consumed by compilation. In most environments the overhead is still very small compared to the significant performance gain of the output code.

>JVM varieties and Java platform independence
>
>All JVM implementations have one thing in common, which is their attempt to get application bytecode translated into machine instructions. Some JVMs interpret application code on load and use performance counters to focus on "hot" code. Some JVMs skip interpretation and rely on compilation alone. The resource intensiveness of compilation can be a bigger hit (especially for client-side applications) but it also enables more advanced optimizations. See [Resources](#resource) for more information.
>
>If you are a beginner to Java, the intricacies of JVMs will be a lot to wrap your head around. The good news is you don't really need to! The JVM manages code compilation and optimization, so you don't have to worry about machine instructions and the optimal way of writing application code for an underlying platform architecture.


#From Java bytecode to execution#

Once you have your Java code compiled into bytecode, the next steps are to translate the bytecode instructions to machine code. This can be done by either an interpreter or a compiler.


##Interpretation##

The simplest form of bytecode compilation is called interpretation. An interpreter simply looks up the hardware instructions for every bytecode instruction and sends it off to be executed by the CPU.

You could think of interpretation similar to using a dictionary: for a specific word (bytecode instruction) there is an exact translation (machine code instruction). Since the interpreter reads and immediately executes one bytecode instruction at a time, there is no opportunity to optimize over an instructions set. An interpreter also has to do the interpretation every time a bytecode is invoked, which makes it fairly slow. Interpretation is an accurate way of executing code, but the un-optimized output instruction set will likely not be the highest-performing sequence for the target platform's processor.


##Compilation##

A *compiler* on the other hand loads the entire code to be executed into the runtime. As it translates bytecode, it has ability to look at the entire or partial runtime context and make decisions about how to actually translate the code. Its decisions are based on analysis of code graphs such as different execution branches of instructions and runtime-context data.

When a bytecode sequence is translated into a machine-code instruction set and optimizations can be done to this instruction set, the replacing instruction set (e.g., the optimized sequence) is stored into a structure called the code cache. The next time that bytecode is executed, the previously optimized code can be immediately located in the code cache and used for execution. In some cases a performance counter might kick in and override the previous optimization, in which case the compiler will run a new optimization sequence. The advantage of a code cache is that the resulting instruction set can be executed at once -- no need for interpretive lookups or compilation! This speeds up execution time, especially for Java applications where the same methods are called multiple times.


##Optimization##

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

I've so far discussed the value of optimizing code and how and when common JVM compilers optimize code. I'll conclude with some of the actual optimizations available to compilers. JVM optimization actually happens at the bytecode level (or on lower representative language levels), but I'll demonstrate the optimizations using the Java language. I couldn't possibly cover all of the JVM optimizations in this section; rather, I mean to inspire you to explore on your own and learn about the hundreds of advanced optimizations and innovations in compiler technology (see [Resources](#resource)).


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





#Resource#    {#resource}
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
