#�ú�Java NIO��NIO.2���������

##ʹ��Java NIO API����������Ӧ�ó���
Cameron Laird, JavaWorld.com, 10/16/12

ԭ�ĵ�ַ��[http://www.javaworld.com/javaworld/jw-10-2012/121016-maximize-java-nio-and-nio2-for-application-responsiveness.html](http://www.javaworld.com/javaworld/jw-10-2012/121016-maximize-java-nio-and-nio2-for-application-responsiveness.html)

2002�꣬J2SE 1.4������Java NIO����Java New Input/Output������������Java NIO��Ŀ����Ϊ�˼�Javaƽ̨�Ͻ���IO�ܼ��ͱ���ĸ��Ӷȡ�ʮ���Ľ��죬�����Java������Ա�Բ��������ú�NIO����������������ʶ��Java SE7��������NIO.2����More New Input/Ooutput���ڱ����У���ʹ��5��ʾ��������˵�����������ͨJava��̳����ú�Java NIO��NIO.2��
NIO��NIO.2��Javaƽ̨����Ҫ������������IO������������ܣ���IO��������JavaӦ�ó��򿪷��ĺ�������֮һ��NIO��������������ʹ�ã�����Ҳ�������е�IO����������Ҫʹ�ã����ǣ���ĳЩ�����У���ȷʹ��Java NIO��NIO.2���Դ�󽵵�IO���������ĵ�ʱ�䡣���ĵ�5��ʾ�����򽫶Դ˽���˵����

1.	�޸�֪ͨ������Ϊÿ���˶���Ҫ��������
2.	ʹ��ѡ����ʵ�ֶ�·����
3.	ͨ�� ���� ��ŵ����ʵ
4.	�ڴ�ӳ�� ���� �Ӻδ���ʼ����
5.	�ַ�����������


##NIO����̬����
һ���Ѿ�������10���Java������Ϊ����Ȼ�ǡ���IO����New Input/Output����ԭ�����ڣ����ںܶ�Java������Ա��˵��Java�Ļ���IO�����Ѿ��㹻ʹ���ˡ������Java��������ʹ��NIO������ճ�����������һ��˵��NIO��������һ�����������������ܵĿ�����������һ����Java IO��ص����ʹ��߼���NIO��NIO.2���ײ����ϵͳ������㱩¶��������Ա��ʹ������Ա������Ӧ�ó���ײ�ʵ�֣��Դ�������JavaӦ�ó�����������ܡ��Կ�����Ա��˵���ڵõ��˶�IO�ĸ������Ȩ��ͬʱ����Ҫ���ѱȲ�������IO����ľ�������ֹ���ִ���ķ�������һ����ԭ������NIO��Ӧ�ó���������Ĺ�ע����һ���ں�����ʾ�������л���н��ܡ�
�������кܶ����NIO�Ĳο����ϣ��μ��������Դ���и��������ӡ�����NIO��NIO.2�ĳ�ѧ����˵��J2SE�ĵ�Java SE7���ĵ��Ǳز����ٵġ����б����е�ʾ��������ҪJDK 7����߰汾��

�Ժܶ࿪����Ա��˵����NIO�ĳ������˷�������Ŀά���У�ĳ��Ӧ�ó���Ĺ��ܿ�����ȷ���У�������Ӧ����������һЩ�˽���ʹ��NIO������Ӧ�ó�����������ܡ���ʹ��NIO������Ӧ�ó������ܺ�����������ʶ�Ŀ����ʵ�ʵĽ�������Ӧ�ó��򱻰󶨵��ײ�ϵͳ����ע�⣬NIO�������ڵײ�ϵͳʵ�ֵġ���������ǵ�һ��ʹ��NIO������Ҫ����ϸ�ĺ���������⡣��ᷢ�֣�NIO֮����������Ӧ�ó�����������ܣ��������������ڵײ����ϵͳ���������ھ����JVM�����������⻷�����洢�豸�����ԣ����������ݱ������ǣ�������һЩ�������ܽ�һЩ���������ġ��ڴˣ�������Ӧ�ó�����ܻ��������ƶ��豸�У�����NIOҪ����ע�⡣

�л�����˵��������̽��һ��NIO��NIO.2��5����Ҫ�ص㡣

1 �޸�֪ͨ������Ϊ���˶���Ҫ��������
����Щ��ϤNIO��NIO.2��Java������Ա��˵��JavaӦ�ó���������һ���Ƚ���ͨ�Ļ��⡣���ǣ����Ҹ��˵ľ�����˵��NIO.2�е��ļ��޸�֪ͨ����NIO API��������עĿ�������ԡ�

�������ҵ��Ӧ���У���Ҫ�����ĳЩ�¼�ִ��ָ���Ķ��������磺

* �ϴ��ļ���FTP��ĳ��Ŀ¼�У�
* �޸��������ļ���
* ������ĳ���ĵ���
* �������ļ�ϵͳ��ĳ���¼���
	
Ŀǰ�Ѿ���һЩʾ��������˵���޸�֪ͨ���޸���Ӧ����ι����ġ���Java�������������ԣ������ڰ汾�У�����޸��¼������ĵ��ͷ�������ѯ����ѯ��һ����������ѭ����

* ����ļ�ϵͳ��ĳ������
* ����һ�ε���֪״̬���Ƚϣ�
* ���״̬û�иı䣬����һ����ʱ���������缸�ٺ�����룩�����¼�飻
* ����ѭ��������̡�

NIO.2�ṩ��һ�ָ��õķ���������޸��¼��ķ�����Listing 1��һ���򵥵�ʾ������

Listing 1. NIO.2���޸�֪ͨ��

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

���벢������δ��룬����ͬһĿ¼��ִ��һЩ������硰touch example1����copy Watcher.class example1�����㽫�ῴ�����µ��޸�֪ͨ��Ϣ��
    Someone just create the file 'example1'.

����򵥵�ʾ������˵�������ʹ��Java��NIO���������ԣ�����NIO.2��Watcher���ʹ�÷����˼򵥽��ܡ�����ڴ�ͳIO�и�����ѯ�Ľ��������Watcher����и�ֱ�ӡ������õ��ص㡣

###ע�����
��������Listing 1�п�������ʱ����Ҫ����ע�⣬��StandardWatchEventKinds��һ�������ʻ㣬����java.net���ĵ��У������ƴ���ˡ�

###��ʾ
����ڴ�ͳIO����ѯ���ԣ�NIO���޸�֪ͨ���������ã�������һЩ������Ա��˺����˶�ʵ������ķ�����������ǵ�һ��ʹ�ü���������ô����Ҫ��ϸ����ʵ���������磬�޸Ľ���ʱ�յ�֪ͨ���ܱ��޸ķ���ʱ�յ�֪ͨ����Ҫ����ĳЩ��������Ҫ�ر���Ҫ��Щ���������ʹ��FTPɾ��Ŀ¼�ĳ�����NIO����ǿ�󣬵���ϸ΢֮����ҪС�Ľ���������������벻��������֡�

##2. ѡ�������첽IO��ѡ����ʵ�ֶ�·����
NIO�ĳ�ѧ����ʱ�ὫNIO�롰������IO��Non-Blocking Input/Output�������Ⱥš�ʵ���ϣ����˷�����IO�⣬NIO�������ܶ��������ݣ�������Ĵ���˵����ĳ�̶ֳ�Ҳ��һ�����Java�Ļ���IO������ͬ���ģ���˼�������IO����ǰ����ǰ�߳���Ҫ�ȴ�����������IO���첽IO����NIO��ʹ�õ��������ԡ�������Listing 1��ʾ�������п�����һ����NIO�еķ�����IO�ǻ����¼��ģ���ѡ��������ƻص�����������������һ��IOͨ����Channel����Ȼ�����������С���ѡ������ĳ���¼�����ʱ������������һ�����֣�ѡ�����ᡰ��������ִ����Ӧ�Ķ�����������Щ�����ڵ��߳�����ɵģ�������봫ͳIO������������

Listing 2չʾ�����ʹ��NIOѡ������ʵ��һ�����Լ�������˿ڵ�Echo����������δ����Ƕ�Greg Travis��2003����Ĵ��루�μ��������Դ�����������޸���ɵġ�

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

������δ��룬���������������У�
    java MultiPortEcho 8005 8006
���г����ʹ��telnet�������ն�ģ��������8005��8006�˿ڣ�����Կ����������Խ��յ����ַ���������Щ�����ڵ��߳������еġ�

#####JavaWorld�и�����NIO�������
* "Master Merlin's new I/O classes" (Michael T. Nygard, JavaWorld, September 2001)
* "Use select for high-speed networking" (Greg Travis, JavaWorld, April 2003)

##3. ͨ������ŵ����ʵ
��NIO�У��ܵ��������κοɶ���д�Ķ������������ڶ��ļ����׽��ֽ��г��󡣹ܵ�ʹ��ͳһ�ķ������ж�д����������ڱ��ʱ���迼�ǲ�������Ĳ����������Ǳ�׼������������ӣ������������Ķ��󣬲�������������ͬ�ġ��ܵ�������ص���Java��ͳIO����Ҳ�������֣���ͬ���ǣ���������IO�����ܵ����첽IO��
ͨ�������ʹ��NIO������Ӧ�ó��������������Ҫ���������������ܸߵ��ŵ㣬��׼ȷ��˵�������������Ӧ�Ե��ص㡣����ĳЩ�����У�NIO���������ܻ���ڴ�ͳIO�����磬��С�ļ����м򵥵ġ����л��Ķ���д������ʹ����ʵ�ֵ�ֱ�Ӷ�д���������ܻ�������¼������ڹܵ������Ķ�д������2��3�������⣬��Щ�Ƕ�·���ùܵ�������ͬ�߳��еĹܵ������������ܻ�ȵ�һ�߳�����ע����ͬһѡ�����Ĺܵ��ͺܶࡣ

�Ժ󣬵�������������⣬����Ϊѡ������ܵ�����ԥ����ʱ��������˼��������������������

* ����Ҫ��д���ٸ�IO����
* �Բ�ͬ��IO���ж�дʱ�Ƿ����ض���˳��Ҫ���أ��ֻ���˵���Ƿ��б�Ҫͬʱ����ЩIO������ж�д������
* IO�����Ƿ�ֻ�ܴ��ں̵ܶ�ʱ�䣿���ǻ��ڽ�����һֱ���ڣ�
* ��IO����Ĳ������ʺ��뵥�̻߳��Ƕ��̣߳�
* ����ӵ���Ƿ��뱾��IO���ƣ�����˵�����и��Բ�ͬ��ģʽ��

���⼸�������˼�����԰���ȷʵ������ѡ�������ǹܵ�����ס��NIO��NIO.2������Ϊ��ȡ����ͳIO�����ֵģ����ǵĳ����ǶԴ�ͳIO�Ĳ��䡣

## 4. �ڴ�ӳ�䡪���Ӻδ���ʼ����
�ڶ�NIO��ʹ���У����������������ܵľ����ڴ�ӳ�䡣�ڴ�ӳ���ǵײ����ϵͳ���ķ��񣬿��Խ��ļ���ĳЩ��ӳ�䵽�ڴ���ʹ�á�

ʵ���ϣ���������̸���ģ��ڴ�ӳ���кܶ��ص㡣�����Ľ����ڴ�ӳ��ʹIO�����ﵽ�˷����ڴ���ٶȣ���󳬳��˷����ļ����ٶȣ�ǰ��ͨ����Ⱥ��߿�2����������Listing 3�Ƕ��ڴ�ӳ��ʹ�õ�һ����ʾ������

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

Listing 3�е�С���Ӵ�����һ��20�ֽڵ��ļ���example_memory_mapped_file.txt�������ַ�A������䣬�ٴ��ļ��ж�ȡ30�ֽڡ���ʵ��Ӧ���У��ڴ�ӳ�������˵ĵط�������������IO��ԭʼ�ٶȣ�����Ϊ�����Խ���ͬ�Ķ�������д����ͬʱ���ӵ�ͬһ��ӳ���С����������ǿ��ʹ������Ҳ�кܸߵ��Ѷ�ϵ��������ȷʹ��������ʵ�ֳ���д�ٶȷǳ����ϵͳ��������֪�������ֵĽ��ײ���ʹ�ô��������Ϊ���ܹ����Ⱦ������ּ��룬�����Ǽ��ٺ��롣

##5. �ַ�����������

������Ҫ���ܵ����һ��NIO������ͨ��charset��ʵ�ֲ�ͬ�ַ�������ת������NIO����֮ǰ��Javaͨ���ڽ���getBytes������ʵ�ֱ���ת���������getBytes ����NIO�е�charset�������������ڵײ�ܹ���ʵ�֣��Ա�ﵽ���õ����ܡ���������˵�����롢У�ԣ��Լ�����һЩ��Ӣ�����Ե����ԣ�������Ҫ�ص㿼�ǵ����أ����ⷽ�棬NIO��charset�������á�

Listing 4�е�ʾ������չʾ����ν�Java��Unicode������ַ�ת��ΪLatin-1������ַ���

Listing 4. Character encoding in NIO

    String some_string = "This is a string that Java natively stores as Unicode.";
    Charset latin1_charset = Charset.forName("ISO-8859-1");
    CharsetEncode latin1_encoder = charset.newEncoder();
    ByteBuffer latin1_bbuf = latin1_encoder.encode(CharBuffer.wrap(some_string));

####ע�⣬���charset�͹ܵ����ʹ����Ϊ��ȷ����Щ��Ҫ�����ڴ�ӳ�䡢�첽IO�ͱ���ת����Ӧ�ó�����Դﵽ�㹻������Ҫ��

####���ۣ���Ȼ���и�������
�ı�����ҪĿ������Java������Ա�ܹ���NIO��NIO.2����Ҫ������һ�����µ��˽⡣ͨ���Ա�����ʾ�������ѧϰ���㻹�����˽⵽NIO���������������磬�Թܵ���ѧϰ�������������ʹ��NIO��Path���������ļ�ϵͳ�ķ������ӡ����⣬�������Դ�У�����һЩ��NIO��������ܵ����¡�

####��������
��Java����ΪJava֮ǰ��Cameron Laird���Ѿ���дJava�����ˣ����Ҵ���ʱ���ż��׫�ĸ�JavaWorld������ע����Twitter�ʺ�@Phaseit��ȡ�����������������

##�����Դ

1.	�μ� Java 2 SDK Standard Edition (SE) documentation��Java SE 7 documentation��ѧϰ����NIO��NIO.2�����֪ʶ��
2.	Ҫʹ��NIO.2��������API������Ҫʹ�� JDK 7����߰汾��
3.	������NIO����2001��9�£���Michael T. Nygard��JavaWorld��������java.nio����������������һЩ�����ڷ�������̺��ڴ�ӳ�仺���ע�����
4.	�μ���ʹ��ѡ�������и����������̡���2003��4�£���Greg Travis������JavaWorld������������NIO���壬��������һ��ʹ��ѡ����ʵ�ָ����������������ʾ����չʾ���Ӧ��ѡ������
5.	������NIOѡ������ʾ���Ƕԡ�NIO��̽����2003��7�£� ��Greg Travis������IBM DeveloperWorks��һ���еĴ����޸Ķ��á�
6.	�����й�NIO.2�����ݣ��μ���NIO.2���ţ�����1���첽�ܵ�API����2010��9�£���Catherine Hope��Oliver Deakin������IBM DeveloperWorks����
