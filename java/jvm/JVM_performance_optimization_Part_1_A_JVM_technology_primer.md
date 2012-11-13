JVM性能优化， Part 1 —— JVM简介
================================================================

原文地址[http://www.javaworld.com/javaworld/jw-08-2012/120821-jvm-performance-optimization-overview.html](http://www.javaworld.com/javaworld/jw-08-2012/120821-jvm-performance-optimization-overview.html)

众所周知，Java应用程序是运行在JVM上的，但是你对JVM有所了解么？作为这个系列文章的第一篇，本文将对经典Java虚拟机的运行机制做简单介绍，内容包括“一次编写，到处运行”的利弊、垃圾回收的基本原理、常用垃圾回收算法的示例和编译器优化等。后续的系列文章将会JVM性能优化的内容进行介绍，包括新一代JVM的设计思路，以及如何支持当今Java应用程序对高性能和高扩展性的要求。

如果你是一名程序员，那么毫无疑问，你肯定有过某种兴奋的感觉，就像是当一束灵感之光照亮了你思考进程，又像是神经元最终建立连接，又像是你解放思想开拓了新的局面。就我个人来说，我喜欢这种学习新知识的感觉。我在工作时就常常会有这种感觉，我的工作会涉及到一些JVM的相关技术，这着实令我兴奋，尤其是工作涉及到垃圾回收和JVM性能优化的时候。在这个系列中，我希望可以与你分享一些这方面的经验，希望你也会像我一样热爱JVM相关技术。

这个系列文章主要面向那些想要裂解JVM底层运行原理的Java程序员。文章立足于较高的层面展开讨论，内容涉及到垃圾回收和在不影响应用程序运行的情况下对安全快速的释放/分配内存。你将对JVM的核心模块有所了解：垃圾回收、GC算法、编译器行为，以及一些常用优化技巧。此外，还会讨论为什么对Java做基准测试（benchmark）是件很困难的事，并提供一些建议来帮助做基准测试。最后，将会介绍一些JVM和GC的前沿技术，内容涉及到Azul的Zing JVM，IBM JVM和Oracle的Garbage First（G1）垃圾回收器。

希望在阅读此系列文章后，你能对影响Java伸缩性的因素有所了解，并且知道这些因素是如何影响Java开发的，如何使Java难以优化的。希望会你有那种发自内心的惊叹，并且能够激励你为Java做一点事情：拒绝限制，努力改变。如果你还没准备好为开源事业贡献力量，希望本系列文章可以为你指明方向。

>*JVM职业生涯*
>在我职业生涯的早期，垃圾回收的问题曾经很难解决。垃圾回收问题和JVM的跨平台问题我更加为JVM和中间件的相关技术而着迷。我对JVM的热情源于十年前在[JRockit][1]团队工作的经历，当时要编码实现一种新的、能够自动学习、自动调优的垃圾回收算法（参见`相关资源`）。从那个项目开始，我踏上了JVM技术之旅，期间在BEA System公司工作的很多年，与Intel公司和Sun公司有过合作关系，在Oracle收购BEA公司和Sun公司之后为Oracle工作了一年。另外，我的硕士论文深入分析了JRockit的试验性特性，为[Deterministic Garbage Collection算法][2]打下了基础。当我加入Azul公司的团队后，我的工作陷入僵局，负责管理维护[Zing JVM][3]的垃圾回收算法（My work came full circle when I joined the team at Azul Systems and got to manage Zing JVM with its unique approach to garbage collection.）。现在我的工作有了一点小变化，负责日程安排与资源管理，关注分布式的可伸缩数据处理框架，目前在Cloudera公司工作，负责开源项目[Hadoop][4]的开发。


[1] http://www.infoworld.com/d/developer-world/oracle-moving-merge-jrockit-hotspot-jvms-448  "JRockit" 
[2] http://www.javaworld.com/javaworld/jw-08-2012/120821-jvm-performance-optimization-overview.html#resources  "Deterministic Garbage Collection算法"  
[3] http://www.infoworld.com/d/developer-world/azul-systems-searches-managed-runtime-breakthroughs-228 "Zing JVM"  
[4] http://www.infoworld.com/d/business-intelligence/cloudera-moves-hadoop-beyond-mapreduce-194941 "Hadoop"  