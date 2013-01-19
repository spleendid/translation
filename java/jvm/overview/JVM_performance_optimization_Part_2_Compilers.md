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
>所有的JVM实现都有一个共同点，即它们都试图将应用程序的字节码转换为本地机器指令。一些JVM在载入应用程序后会解释执行应用程序，同时使用性能计数器来查找“热点”代码。还有一些JVM会调用解释执行的阶段，直接编译运行。资源密集型编译任务对应用程序来说可能会产生较大影响，尤其是那些客户端模式下运行的应用程序，但是资源密集型编译任务可以执行一些比较高级的优化任务。更多相关内容请参见[相关资源][13]
>
>如果你是Java初学者，JVM本身错综复杂结构会让你晕头转向的。不过，好消息是你无需精通JVM。JVM自己会做好代码编译和优化的工作，所以你无需关心如何针对目标平台架构来编写应用程序才能编译、优化，从而生成更好的本地机器指令。


#从字节码到可运行的程序#

当你编写完Java源代码并将之编译为字节码后，下一步就是将字节码指令编译为本地机器指令。这一步会由解释器或编译器完成。


##解释##

解释是最简单的字节码编译形式。解释器查找每条字节码指令对应的硬件指令，再由CPU执行相应的硬件指令。

你可以将解释器想象为一个字典：每个单词（字节码指令）都有准确的解释（本地机器指令）。由于解释器每次读取一个字节码指令并立即执行，因此它就没有机会对某个指令集合进行优化。由于每次执行字节码时，解释器都需要做相应的解释工作，因此程序运行起来就很慢。解释执行可以准确执行字节码，但是未经优化而输出的指令集难以发挥目标平台处理器的最佳性能。


##编译##

另一方面，编译执行应用程序时，*编译器*会将加载运行时会用到的全部代码。因为编译器可以将字节码编译为本地代码，因此它可以获取到完整或部分运行时上下文信息，并依据收集到的信息决定到底应该如何编译字节码。编译器是根据诸如指令的不同执行分支和运行时上下文数据等代码信息来指定决策的。

当字节码序列被编译为机器代码指令集合时，就可以对这个指令集合做一些优化操作了，优化后的指令集合会被存储到成为code cache的数据结构中。当下一次执行这部分字节码序列时，就会执行这些经过优化后被存储到code cache的指令集合。在某些情况下，性能计数器会失效，并覆盖掉先前所做的优化，这时，编译器会执行一次新的优化过程。使用code cache的好处是优化后的指令集可以立即执行 —— 无需像解释器一样再经过查找的过程或编译过程！这可以加速程序运行，尤其是像Java应用程序这种同一个方法会被多次调用应用程序。


##优化##

随着动态编译器一起出现的是性能计数器。例如，编译器会插入性能计数器，以统计每个字节码块（对应与某个被调用的方法）的调用次数。在进行相关优化时，编译器会使用收集到的数据来判断某个字节码块有多“热”，这样可以最大程度的降低对当前应用程序的影响。运行时数据监控有助于编译器完成多种代码优化工作，进一步提升代码执行性能。随着收集到的运行时数据越来越多，编译器就可以完成一些额外的、更加复杂的代码优化工作，例如编译出更高质量的目标代码，使用运行效率更高的代码替换原代码，甚至是剔除冗余操作等。


##示例##

考虑如下代码：

    static int add7( int x ) {
        return x+7;
    }

这段代码经过javac编译后会产生如下的字节码：

    iload0
    bipush 7
    iadd
    ireturn

当调用这段代码时，字节码块会被动态的编译为本地机器指令。当性能计数器（如果这段代码应用了性能计数器的话）发现这段代码的运行次数超过了某个阈值后，动态编译器会对这段代码进行优化编译。后带的代码可能会是下面这个样子：

    lea rax,[rdx+7]
    ret


#各擅胜场#

不同的Java应用程序需要满足不同的需求。相对来说，企业级服务器端应用程序需要长时间运行，因此可以做更多的优化，而稍小点的客户端应用程序可能要求快速启动运行，占资源少。接下来我们考察三种编译器设置及其各自的优缺点。


##客户端编译器##

即大家熟知的优化编译器C1。在启动应用程序时，添加JVM启动参数“-client”可以启用C1编译器。正如启动参数所表示的，C1是一个客户端编译器，它专为客户端应用程序而设计，资源消耗更少，并且在大多数情况下，对应用程序的启动时间很敏感。C1编译器使用性能计数器来收集代码的运行时信息，执行一些简单、无侵入的代码优化任务。


##服务器端编译器##

对于那些需要长时间运行的应用程序，例如服务器端的企业级Java应用程序来说，客户端编译器所实现的功能还略有不足，因此服务器端的编译会使用类似C2这类的编译器。启动应用程序时添加命令行参数“-server”可以启用C2编译器。由于大多数服务器端应用程序都会长时间运行，因此相对于运行时间稍短的轻量级客户端应用程序，在服务器端应用程序中启用C2编译器可以收集到更多的运行时数据，也就可以执行一些更高级的编译技术与算法。


>*提示：给服务器端编译器热身*
>
>对于服务器端编译器来说，在应用程序开始运行之后，编译器可能会在一段时间之后才开始优化“热点”代码，所以服务器端编译器通常需要经过一个“热身”阶段。在服务器端编译器执行性能优化任务之前，要确保应用程序的各项准备工作都已就绪。给予编译器足够多的时间来完成编译、优化的工作才能取得更好的效果。（更多关于编译器热身与监控原理的内容请参见JavaWorld的文章"[Watch your HotSpot compiler go][6]"。）

在执行编译任务优化任务时，服务器端编译器要比客户端编译器综合考虑更多的运行时信息，执行更复杂的分支分析，即对哪种优化路径能取得更好的效果作出判断。获取的运行时数据越多，编译优化所产生的效果越好。当然，要完成一些复杂的、高级的性能分析任务，编译器就需要消耗更多的资源。使用了C2编译器的JVM会消耗更多的资源，例如更多的线程，更多的CPU指令周期，以及更大的code cache等。


##层次编译##

*层次编译*综合了服务器端编译器和客户端编译器的特点。Azul首先在其Zing JVM中实现了层次编译。最近（就是Java SE 7版本），Oracle Java HotSpot VM也采用了这种设计。在应用程序启动阶段，客户端编译器最为活跃，执行一些由较低的性能计数器阈值出发的性能优化任务。此外，客户端编译器还会插入性能计数器，为一些更复杂的性能优化任务准备指令集，这些任务将在后续的阶段中由服务器端编译器完成。层次编译可以更有效的利用资源，因为编译器在执行一些对应用程序影响较小的编译活动时仍可以继续收集运行时信息，而这些信息可以在将来用于完成更高级的优化任务。使用层次编译可以比解释性的代码性能计数器手机到更多的信息。

Figure 1中展示了纯解释运行、客户端模式运行、服务器端模式运行和层次编译模式运行下性能之间的区别。X轴表示运行时间（单位时间）Y轴表示性能（每单位时间内的操作数）。

![Figure 1. Performance differences between compilers](../../../images/jvmseries2-fig1.png?raw=true "Figure 1. Performance differences between compilers")

*Figure 1. Performance differences between compilers*


##编译性能对比##

相比于纯解释运行的的代码，以客户端模式编译运行的代码在性能（指单位时间执行的操作）上可以达到约5到10倍，因此而提升了应用程序的运行性能。其间的区别主要在于编译器的效率、编译器所作的优化，以及应用程序在设计实现时针对目标平台做了何种程度的优化。实际上，最后一条不在Java程序员的考虑之列。

相比于客户端编译器，使用服务器端编译器通常会有30%到50%的性能提升。在大多数情况下，这种程度的性能提升足以弥补使用服务器端编译所带来的额外资源消耗。

层次编译综合了服务器端编译器和客户端编译器的优点，使用客户端编译模式实现快速启动和快速优化，使用服务器端编译模式在后续的执行周期中完成高级优化的编译任务。


#常用编译优化手段#

到目前为止，已经介绍了优化代码的价值，以及常用JVM编译器是如何以及何时编译代码的。接下来，将用一些实际的例子做个总结。JVM所作的性能优化通常在字节码这一层级（或者是更底层的语言表示），但这里我将使用Java编程语言对优化措施进行介绍。在这一节中，我无法涵盖JVM中所作的所有性能优化，相反，我希望可以激发你的兴趣，使你主动挖掘并学习编译器技术中所包含了数百种高级优化技术（参见[相关资源][13]）。


##死代码剔除##

*死代码剔除*指的是，将用于无法被调用的代码，即“死代码”，从源代码中剔除。如果编译器在运行时发现某些指令是不必要的，它会简单的将其从可执行指令集中剔除。例如，在Listing 1中，变量被赋予了确定值，却从未被使用，因此可以在执行时将其完全忽略掉。在字节码这一层级，也就不会有将数值载入到寄存器的操作。没有载入操作意味着可以更少的CPU时间，更好的运行性能，尤其是当这段代码是“热点”代码的时候。

Listing 1中展示了示例代码，其中被赋予了固定值的代码从未被使用，属于无用不必要的操作。

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

在字节码这一层级，如果变量被载入但从未使用，编译器会检测到并剔除这个死代码，如Listing 2所示。剔除死代码可以节省CPU时间，从而提升应用程序的运行速度。

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

冗余剔除是一种类似的优化手段，通过剔除掉重复的指令来提升应用程序性能。


##内联##

许多优化手段都试图消除机器级跳转指令（例如，x86架构的JMP指令）。跳转指令会修改指令指针寄存器，因此而改变了执行流程。相比于其他汇编指令，跳转指令是一个代价高昂的指令，这也是为什么大多数优化手段会试图减少甚至是消除跳转指令。内联是一种家喻户晓而且好评如潮的优化手段，这是因为跳转指令代价高昂，而内联技术可以将经常调用的、具有不容入口地址的小方法整合到调用方法中。Listing 3到Listing 5中的Java代码展示了使用内联的用法。

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

在Listing 3到Listing 5的代码中，展示了将调用3次小方法进行内联的示例，这里我们认为使用内联比跳转有更多的优势。

如果被内联的方法本身就很少被调用的话，那么使用内联也没什么意义，但是对频繁调用的“热点”方法进行内联在性能上会有很大的提升。此外，经过内联处理后，就可以对内联后的代码进行进一步的优化，正如Listing 6中所展示的那样。

_Listing 6. After inlining, more optimizations can be applied_

    int whenToEvaluateZing(int y){
        if(y == 0) return y;
        else if (y == -1) return y - 1;
        else return y + y - 1;
    }


##循环优化##

当涉及到需要减少执行循环时的性能损耗时，循环优化起着举足轻重的作用。执行循环时的性能损耗包括代价高昂的跳转操作，大量的条件检查，和未经优化的指令流水线（即引起CPU空操作或额外周期的指令序列）等。循环优化可以分为很多种，在各种优化手段中占有重要比重。其中值得注意的包括以下几种：

* 合并循环：当两个相邻循环的迭代次数相同时，编译器会尝试将两个循环体进行合并。当两个循环体中没有相互引用的情况，即各自独立时，可以同时执行（并行执行）。
* 反转循环：基本上将就是用do-while循环体换掉常规的while循环，这个do-while循环嵌套在if语句块中。这个替换操作可以节省两次跳转操作，但是，会增加一个条件检查的操作，因此增加的代码量。这种优化方式完美的展示了以少量增加代码量为代价换取较大性能的提升 —— 编译器需要在运行时需要权衡这种得与失，并制定编译策略。
* 分块循环：重新组织循环体，以便迭代数据块时，便于缓存的应用。
* 展开循环：减少判断循环条件和跳转的次数。你可以将之理解为将一些迭代的循环体“内联”到一起，而无需跨越循环条件。展开循环是有风险的，它有可能会降低应用程序的运行性能，因为它会影响流水线的运行，导致产生了冗余指令。再强调一遍，展开循环是编译器在运行时根据各种信息来决定是否使用的优化手段，如果有足够的收益的话，那么即使有些性能损耗也是值得的。

至此，已经简要介绍了编译器对字节码层级（以及更底层）进行优化，以提升应用程序在目标平台的执行性能的几种方式。这里介绍的几种优化手段是比较常用的几种，只是众多优化技术中的几种。在介绍优化方法时配以简单示例和相关解释，希望可以洗发你进行深度探索的兴趣。更多相关内容请参见[相关资源][13]。


#总结：回顾#

为满足不同需要而使用不同的编译器。

* 解释是将字节码转换为本地机器指令的最简单方式，其工作方式是基于对本地机器指令表的查找。
* 编译器可以基于性能计数器进行性能优化，但是需要消耗更多的资源（如code cache，优化线程等）。
* 相比于纯解释执行代码，客户端编译器可以将应用程序的执行性能提升一个数量级（约5到10倍）。
* 相比于客户端编译器，服务器端编译器可以将应用程序的执行性能提升30%到50%，但会消耗更多的资源。
* 层次编译综合了客户端编译器和服务器端编译器的优点，既可以像客户端编译器那样快速启动，又可以像服务器端编译器那样，在长时间收集运行时信息的基础上，优化应用程序的性能。

目前，已经出现了很多代码优化的手段。对编译器来说，一个主要的任务就是分析所有的可能性，权衡使用某种优化手段的利弊，在此基础上编译代码，优化应用程序的性能。


##关于作者##

Eva Andearsson对JVM计数、SOA、云计算和其他企业级中间件解决方案有着10多年的从业经验。在2001年，她以JRockit JVM开发者的身份加盟了创业公司Appeal Virtual Solutions（即BEA公司的前身）。在垃圾回收领域的研究和算法方面，EVA获得了两项专利。此外她还是提出了确定性垃圾回收（Deterministic Garbage Collection），后来形成了JRockit实时系统（JRockit Real Time）。在技术上，Eva与SUn公司和Intel公司合作密切，涉及到很多将JRockit产品线、WebLogic和Coherence整合的项目。2009年，Eva加盟了Azul System公，担任产品经理。负责新的Zing Java平台的开发工作。最近，她改换门庭，以高级产品经理的身份加盟Cloudera公司，负责管理Cloudera公司Hadoop分布式系统，致力于高扩展性、分布式数据处理框架的开发。


#相关资源#

* ["JVM性能优化， Part 1 ——JVM简介"][2](作者Eva Andreasson, 于2012年8约发表于JavaWorld)是该系列的第一篇，对经典JVM的工作原理做了简单介绍，包括Java“一次编写，到处运行”的优势，垃圾回收基础和一些常用的垃圾回收算法。
* 更多有关HotSpot优化原理以及JVM热身的内容请参见Vladimir Roubtsov与2003年4约发表于JavaWorld.com的文章["Watch your HotSpot compiler go"][6]
* 如果你想对JVM和字节码有更深入的了解，请参见Bill Venners在1996年发表于JavaWorld的文章["Bytecode basics"][4]。文章对JVM中的字节码指令集做了介绍，内容包括原生类型操作、类型转换以及栈上操作等。
* 在Java平台的官方文档中有对[Java编译器][8]javac的详细描述。
* 更多有关JVM中JIT编译器的内容，请参见IBM Research中有关[Java JIT Compiler][9]的内容。
* 或者参见Oracle JRockit文档中["Understanding Just-In-Time Compilation and Optimization"][10]的相关内容.
* Cliff Click博士在其博客上有关于[层次编译][11]的完整教程。
* 更多有关使用性能计数器完成JVM性能优化的文章：["Using Platform-Specific Performance Counters for Dynamic Compilation"][12] (作者Florian Schneider与Thomas R. Gross;由ACM Digital Lirary发表在第18届Languages and Compilers for Parallel Computing会议上)
* Oracle JRockit: The Definitive Guide (Marcus Hirt, Marcus Lagergren; Packt Publishing, 2010): Oracle JRockit权威指南



[1]:  http://www.javaworld.com/javaworld/jw-09-2012/120905-jvm-performance-optimization-compilers.html  "http://www.javaworld.com/javaworld/jw-09-2012/120905-jvm-performance-optimization-compilers.html"
[2]:  http://www.javaworld.com/javaworld/jw-08-2012/120821-jvm-performance-optimization-overview.html  "Java性能优化"
[3]:  http://www.javaworld.com/javaworld/jw-08-2012/120821-jvm-performance-optimization-overview.html  "Java性能优化，第1部分"
[4]:  http://www.javaworld.com/javaworld/jw-09-1996/jw-09-bytecodes.html  "字节码基础"
[5]:  http://www.javaworld.com/javaworld/jw-08-2012/120821-jvm-performance-optimization-overview.html  "静态与动态编译器"
[6]:  http://www.javaworld.com/javaqa/2003-04/01-qa-0411-hotspot.html  "Watch your HotSpot compiler go"
[7]:  https://github.com/caoxudong/translation/blob/master/java/jvm/JVM_performance_optimization_Part_2_Compilers.md#resource "相关资源"
[8]:  http://docs.oracle.com/javase/6/docs/technotes/guides/javac/index.html  "Java Compiler"
[9]:  http://researchweb.watson.ibm.com/trl/projects/jit/index_e.htm  "Java JIT Compiler"
[10]: http://docs.oracle.com/cd/E15289_01/doc.40/e15058/underst_jit.htm  "Understanding Just-In-Time Compilation and Optimization"
[11]: http://www.azulsystems.com/blog/cliff/2010-07-16-tiered-compilation "层次编译"
[12]: http://dl.acm.org/citation.cfm?id=2081919  "Using Platform-Specific Performance Counters for Dynamic Compilation"
[13]: https://github.com/caoxudong/translation/blob/master/java/jvm/JVM_performance_optimization_Part_2_Compilers.md#%E7%9B%B8%E5%85%B3%E8%B5%84%E6%BA%90 "相关资源"