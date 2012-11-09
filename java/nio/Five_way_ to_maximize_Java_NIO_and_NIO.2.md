用好Java NIO和NIO.2的五个方法
==========================================

#使用Java NIO API构建高性能应用程序
Cameron Laird, JavaWorld.com, 10/16/12

原文地址：[http://www.javaworld.com/javaworld/jw-10-2012/121016-maximize-java-nio-and-nio2-for-application-responsiveness.html](http://www.javaworld.com/javaworld/jw-10-2012/121016-maximize-java-nio-and-nio2-for-application-responsiveness.html)

2002年，J2SE 1.4引入了Java NIO，即Java New Input/Output开发包。引入Java NIO的目的是为了简化Java平台上进行IO密集型编码的复杂度。十年后的今天，有许多Java开发人员仍不清楚如何用好NIO，甚至很少有人意识到Java SE7中引入了NIO.2，即More New Input/Ooutput。在本文中，将使用5个示例程序来说明，如何在普通Java编程场景用好Java NIO和NIO.2。
NIO和NIO.2对Java平台的主要贡献是提升了IO处理的运行性能，而IO处理正是Java应用程序开发的核心领域之一。NIO开发包并不容易使用，而且也不是所有的IO场景都必须要使用，但是，在某些场景中，正确使用Java NIO和NIO.2可以大大降低IO操作所消耗的时间。本文的5个示例程序将对此进行说明：

1.	修改通知符（因为每个人都需要监听器）
2.	使用选择器实现多路复用
3.	通道 —— 承诺与现实
4.	内存映射 —— 从何处开始计算
5.	字符编码与搜索


#NIO的生态环境
一个已经出现了10年的Java开发包为何仍然是“新IO”（New Input/Output）？原因在于，对于很多Java开发人员来说，Java的基本IO操作已经足够使用了。大多数Java程序无需使用NIO来完成日常工作。更进一步说，NIO不仅仅是一个用于提升运行性能的开发包，更是一个与Java IO相关的异质工具集。NIO和NIO.2将底层操作系统的切入点暴露给开发人员，使开发人员更贴近应用程序底层实现，以此来提升Java应用程序的运行性能。对开发人员来说，在得到了对IO的更大控制权的同时，需要花费比操作基本IO更多的精力来防止各种错误的发生。另一方面原因在于NIO对应用程序表现力的关注，这一点在后续的示例程序中会进行介绍。
起步现在有很多介绍NIO的参考资料，参见“相关资源”中给出的链接。对于NIO和NIO.2的初学者来说，J2SE文档Java SE7的文档是必不可少的。运行本文中的示例程序需要JDK 7或更高版本。

对很多开发人员来说，与NIO的初次邂逅发生在项目维护中：某个应用程序的功能可以正确运行，但是响应很慢，于是一些人建议使用NIO来提升应用程序的运行性能。当使用NIO提升了应用程序性能后，你会觉得它光彩夺目，但实际的结果是你的应用程序被绑定到底层系统。（注意，NIO是依赖于底层系统实现的。）如果你是第一次使用NIO，你需要更仔细的衡量这个问题。你会发现，NIO之所以能提升应用程序的运行性能，不仅仅是依赖于底层操作系统，还依赖于具体的JVM、主机的虚拟环境、存储设备的特性，甚至是数据本身。但是，还是有一些技巧来总结一些考量方法的。在此，如果你的应用程序可能会运行在移动设备中，对于NIO要尤其注意。

闲话不多说，下面来探索一下NIO和NIO.2的5个重要特点。

#1 修改通知符（因为人人都需要监听器）
对那些熟悉NIO和NIO.2的Java开发人员来说，Java应用程序性能是一个比较普通的话题。但是，就我个人的经验来说，NIO.2中的文件修改通知符是NIO API中最引人注目的新特性。

在许多企业级应用中，需要对针对某些事件执行指定的动作，例如：

* 上传文件到FTP的某个目录中；
* 修改了配置文件；
* 更新了某个文档；
* 发生了文件系统的某个事件。
	
目前已经有一些示例程序来说明修改通知或修改相应是如何工作的。在Java（包括其他语言）的早期版本中，检查修改事件发生的典型方法是轮询。轮询是一种特殊无限循环：

* 检查文件系统或某个对象；
* 与上一次的已知状态做比较；
* 如果状态没有改变，则在一定的时间间隔（例如几百毫秒或几秒）后，重新检查；
* 无限循环这个过程。

NIO.2提供了一种更好的方法来检查修改事件的发生。Listing 1是一个简单的示例程序。

Listing 1. NIO.2中修改通知符

    import java.nio.file.attribute.*;
    import java.io.*;
    import java.util.*;
    import java.nio.file.Path;
    import java.nio.file.Paths;
    import java.nio.file.StandardWatchEventKinds;
    import java.nio.file.WatchEvent;
    import java.nio.file.WatchKey;
    import java.nio.file.WatchService;
    import java.util.List;
    
    public class Watcher {
       public static void main(String[] args) {
          Path this_dir = Paths.get(".");    
          System.out.println("Now watching the current directory ...");  
   
          try {
              WatchService watcher = this_dir.getFileSystem().newWatchService();
              this_dir.register(watcher, StandardWatchEventKinds.ENTRY_CREATE);
   
              WatchKey watckKey = watcher.take();
   
              List<WatchEvent<?>> events = watckKey.pollEvents();
              for (WatchEvent event : events) {
                  System.out.println("Someone just created the file '" + event.context().toString() + "'.");
  
             }
   
         } catch (Exception e) {
             System.out.println("Error: " + e.toString());
         }
      }
    }

编译并运行这段代码，并在同一目录下执行一些命令，例如“touch example1”或“copy Watcher.class example1”。你将会看到如下的修改通知信息：
    Someone just create the file 'example1'.

这个简单的示例程序说明了如何使用Java中NIO的语言特性，并对NIO.2中Watcher类的使用费做了简单介绍。相比于传统IO中给予轮询的解决方案，Watcher类具有更直接、更易用的特点。

##注意笔误
当你从这个Listing 1中拷贝代码时，需要谨慎注意，类StandardWatchEventKinds是一个复数词汇，而在java.net的文档中，这个词拼错了。

##提示
相比于传统IO的轮询策略，NIO的修改通知符简明易用，以至于一些开发人员因此忽略了对实际需求的分析。如果你是第一次使用监听器，那么就需要仔细分析实际需求。例如，修改结束时收到通知可能比修改发生时收到通知更重要。在某些场景中需要特别重要这些情况，例如使用FTP删除目录的场景。NIO功能强大，但在细微之后需要小心谨慎，否则会有意想不到情况出现。

#2. 选择器与异步IO：选择器实现多路复用
NIO的初学者有时会将NIO与“非阻塞IO（Non-Blocking Input/Output）”划等号。实际上，除了非阻塞IO外，NIO还包含很多其他内容，但上面的错误说法在某种程度也有一点道理：Java的基本IO操作是同步的，意思是在完成IO操作前，当前线程需要等待，而非阻塞IO或异步IO则是NIO中使用得最多的特性。正如在Listing 1的示例程序中看到的一样，NIO中的非阻塞IO是基于事件的，即选择器（或称回调、监听器）定义了一个IO通道（Channel），然后程序继续运行。当选择器上某个事件发生时，例如输入了一行文字，选择器会“醒来”并执行相应的动作。所有这些都是在单线程中完成的，这就是与传统IO最显著的区别。

Listing 2展示了如何使用NIO选择器来实现一个可以监听多个端口的Echo服务器，这段代码是对Greg Travis在2003发表的代码（参见“相关资源”）做少量修改完成的。

Listing 2. NIO selectors
    import java.io.*;
    import java.net.*;
    import java.nio.*;
    import java.nio.channels.*;
    import java.util.*;
   
    public class MultiPortEcho{
        private int ports[];
        private ByteBuffer echoBuffer = ByteBuffer.allocate( 1024 );
        
        public MultiPortEcho( int ports[] ) throws IOException {
            this.ports = ports;
            configure_selector();
        }
  
        private void configure_selector() throws IOException {
            // Create a new selector
            Selector selector = Selector.open();
      
            // Open a listener on each port, and register each one
            // with the selector
            for (int i=0; i<ports.length; ++i) {
                ServerSocketChannel ssc = ServerSocketChannel.open();
                ssc.configureBlocking(false);
                ServerSocket ss = ssc.socket();
                InetSocketAddress address = new InetSocketAddress(ports[i]);
                ss.bind(address);
                
                SelectionKey key = ssc.register(selector, SelectionKey.OP_ACCEPT);
                System.out.println("Going to listen on " + ports[i]);
            }
            while (true) {
                int num = selector.select();
                Set selectedKeys = selector.selectedKeys();
                Iterator it = selectedKeys.iterator();
                while (it.hasNext()) {
                    SelectionKey key = (SelectionKey) it.next();
                    if ((key.readyOps() & SelectionKey.OP_ACCEPT)
                            == SelectionKey.OP_ACCEPT) {
                        // Accept the new connection
                        ServerSocketChannel ssc = (ServerSocketChannel)key.channel();
                        SocketChannel sc = ssc.accept();
                        sc.configureBlocking(false); 
                        // Add the new connection to the selector
                        SelectionKey newKey = sc.register(selector, SelectionKey.OP_READ);
                        it.remove();
                        System.out.println( "Got connection from "+sc );
                    } else if ((key.readyOps() & SelectionKey.OP_READ)
                            == SelectionKey.OP_READ) {
                        // Read the data
                        SocketChannel sc = (SocketChannel)key.channel();  
                        // Echo data
                        int bytesEchoed = 0;
                        while (true) {
                            echoBuffer.clear();  
                            int number_of_bytes = sc.read(echoBuffer);
                            if (number_of_bytes <= 0) {
                                break;
                            }  
                           echoBuffer.flip();
                           sc.write(echoBuffer);
                           bytesEchoed += number_of_bytes;
                        }  
                        System.out.println("Echoed " + bytesEchoed + " from " + sc);
                        it.remove();
                    }  
                }
            }
        }
      
        static public void main( String args[] ) throws Exception {
            if (args.length<=0) {
                System.err.println("Usage: java MultiPortEcho port [port port ...]");
                System.exit(1);
            }
      
          int ports[] = new int[args.length];
      
          for (int i=0; i<args.length; ++i) {
              ports[i] = Integer.parseInt(args[i]);
          }
      
          new MultiPortEcho(ports);
        }
    }

编译这段代码，并用如下命令运行：
    java MultiPortEcho 8005 8006
运行程序后，使用telnet或其他终端模拟器访问8005和8006端口，你可以看到程序会回显接收到的字符。所有这些都是在单线程下运行的。

###JavaWorld中更多与NIO相关文章
* "Master Merlin's new I/O classes" (Michael T. Nygard, JavaWorld, September 2001)
* "Use select for high-speed networking" (Greg Travis, JavaWorld, April 2003)

#3. 通道：承诺与现实
在NIO中，管道可以是任何可读可写的对象，作用是用于对文件和套接字进行抽象。管道使用统一的方法进行读写操作，因此在编程时无需考虑操作具体的操作，无论是标准输出、网络连接，或者是其他的对象，操作方法都是相同的。管道的这个特点在Java传统IO中流也有所体现，不同的是，流是阻塞IO，而管道是异步IO。
通常情况，使用NIO来提升应用程序的整体性能主要利用了其运行性能高的优点，正准确的说是利用了其高响应性的特点。但在某些场景中，NIO的运行性能会低于传统IO。例如，对小文件进行简单的、序列化的读和写操作，使用流实现的直接读写操作的性能会比面向事件、基于管道开发的读写操作高2到3倍。此外，那些非多路复用管道（即不同线程中的管道）的运行性能会比单一线程里面注册在同一选择器的管道低很多。

以后，当你遇到编程问题，正在为选择流或管道而犹豫不决时，可以先思考几个问题再做决定：

* 你需要读写多少个IO对象？
* 对不同的IO进行读写时是否有特定的顺序要遵守？又或者说，是否有必要同时对这些IO对象进行读写操作？
* IO对象是否只能存在很短的时间？还是会在进程中一直存在？
* 对IO对象的操作更适合与单线程还是多线程？
* 网络拥堵是否与本地IO类似？还是说它们有各自不同的模式？

对这几个问题的思考可以帮你确实到底是选用流还是管道。记住，NIO和NIO.2并不是为了取代传统IO而出现的，它们的出现是对传统IO的补充。

# 4. 内存映射——从何处开始计算
在对NIO的使用中，最能显著提升性能的就是内存映射。内存映射是底层操作系统级的服务，可以将文件的某些段映射到内存中使用。

实际上，除了上面谈到的，内存映射有很多特点。概括的将，内存映射使IO操作达到了访问内存的速度，大大超出了访问文件的速度，前者通常会比后者快2个数量级。Listing 3是对内存映射使用的一个简单示例程序。

Listing 3. Memory mapping in NIO

    import java.io.RandomAccessFile;
    import java.nio.MappedByteBuffer;
    import java.nio.channels.FileChannel;
  
    public class mem_map_example {
        private static int mem_map_size = 20 * 1024 * 1024;
        private static String fn = "example_memory_mapped_file.txt";
  
        public static void main(String[] args) throws Exception {
            RandomAccessFile memoryMappedFile = new RandomAccessFile(fn, "rw");
            //Mapping a file into memory
            MappedByteBuffer out = memoryMappedFile.getChannel().map(FileChannel.MapMode.READ_WRITE, 0, mem_map_size);
        
            //Writing into Memory Mapped File
            for (int i = 0; i < mem_map_size; i++) {
                out.put((byte) 'A');
            } 
            System.out.println("File '" + fn + "' is now " + Integer.toString(mem_map_size) + " bytes full.");
        
            // Read from memory-mapped file.
            for (int i = 0; i < 30 ; i++) {
                System.out.print((char) out.get(i));
            }
            System.out.println("\nReading from memory-mapped file '" + fn + "' is complete.");
        }
    }

Listing 3中的小例子创建了一个20字节的文件，example_memory_mapped_file.txt，并用字符A进行填充，再从文件中读取30字节。在实际应用中，内存映射吸引人的地方，不仅仅在于IO的原始速度，还以为它可以将不同的读操作和写操作同时附加到同一个映像中。这项技术功能强大，使用起来也有很高的难度系数，但正确使用它可以实现出读写速度非常快的系统。众所周知，华尔街的交易操作使用此项技术就是为了能够领先竞争对手几秒，甚至是几百毫秒。

#5. 字符编码与搜索

本文中要介绍的最后一个NIO特性是通过charset包实现不同字符编码间的转换。在NIO出现之前，Java通过内建的getBytes方法来实现编码转换。相比于getBytes 方法NIO中的charset更加灵活，更容易在底层架构中实现，以便达到更好的性能。对搜索来说，编码、校对，以及其他一些非英语语言的特性，都是需要重点考虑的因素，在这方面，NIO的charset更加适用。

Listing 4中的示例程序展示了如何将Java中Unicode编码的字符转换为Latin-1编码的字符。

Listing 4. Character encoding in NIO

    String some_string = "This is a string that Java natively stores as Unicode.";
    Charset latin1_charset = Charset.forName("ISO-8859-1");
    CharsetEncode latin1_encoder = charset.newEncoder();
    ByteBuffer latin1_bbuf = latin1_encoder.encode(CharBuffer.wrap(some_string));

###注意，设计charset和管道配合使用是为了确保那些需要整合内存映射、异步IO和编码转换的应用程序可以达到足够的性能要求。

####结论：当然还有更多内容
文本的主要目的是让Java开发人员能够对NIO和NIO.2的主要功能有一个大致的了解。通过对本文中示例程序的学习，你还可以了解到NIO中其他方法，例如，对管道的学习会帮助你理解如何使用NIO的Path类来管理文件系统的符号链接。另外，在相关资源中，还有一些对NIO做深入介绍的文章。

###关于作者
在Java被称为Java之前，Cameron Laird就已经编写Java代码了，并且从那时起就偶尔撰文给JavaWorld发表。关注他的Twitter帐号@Phaseit获取他编码与著作情况。

#相关资源

1.	参见 Java 2 SDK Standard Edition (SE) documentation和Java SE 7 documentation来学习更多NIO与NIO.2的相关知识。
2.	要使用NIO.2开发包的API，你需要使用 JDK 7或更高版本。
3.	“掌握NIO”（2001年9月，由Michael T. Nygard于JavaWorld）介绍了java.nio开发包，并给出了一些有助于非阻塞编程和内存映射缓冲的注意事项。
4.	参见“使用选择器进行高性能网络编程”（2003年4月，由Greg Travis发表于JavaWorld），该文审视NIO整体，并给出了一个使用选择器实现高性能网络服务器的示例来展示如何应用选择器。
5.	本文中NIO选择器的示例是对“NIO初探”（2003年7月， 由Greg Travis发表于IBM DeveloperWorks）一文中的代码修改而得。
6.	更多有关NIO.2的内容，参见“NIO.2入门，部分1：异步管道API”（2010年9月，由Catherine Hope和Oliver Deakin发表于IBM DeveloperWorks）。
