1. https://blog.csdn.net/Testfan_zhou/article/details/92579810
2. https://alibaba.github.io/arthas/index.html

根据Arthas官网，选择全量安装模式下载arthas-packaging-3.1.1-bin.7z 并解压缩；以cmd模式进入对应目录，根据文档使用arthas-boot.jar，直接用java -jar的方式启动：
```language
C:\Users\***\arthas-packaging-3.1.1-bin>java -jar arthas-boot.jar
[INFO] arthas-boot version: 3.1.1
[INFO] Found existing java process, please choose one and hit RETURN.
* [1]: 2780
  [2]: 3692 org.apache.catalina.startup.Bootstrap
  [3]: 8052 org.apache.zookeeper.server.quorum.QuorumPeerMain
  [4]: 6108 org.apache.catalina.startup.Bootstrap
1
[INFO] arthas home: C:\Users\483879\Desktop\arthas-packaging-3.1.1-bin
[INFO] Try to attach process 2780
[INFO] Found java home from System Env JAVA_HOME: D:\work\java\jdk
[ERROR] Start arthas failed, exception stack trace:
com.sun.tools.attach.AttachNotSupportedException: Unable to attach to 64-bit pro
cess
        at sun.tools.attach.WindowsVirtualMachine.openProcess(Native Method)
        at sun.tools.attach.WindowsVirtualMachine.<init>(WindowsVirtualMachine.j
ava:56)
        at sun.tools.attach.WindowsAttachProvider.attachVirtualMachine(WindowsAt
tachProvider.java:69)
        at com.sun.tools.attach.spi.AttachProvider.attachVirtualMachine(AttachPr
ovider.java:193)
        at com.sun.tools.attach.VirtualMachine.attach(VirtualMachine.java:255)
        at com.taobao.arthas.core.Arthas.attachAgent(Arthas.java:75)
        at com.taobao.arthas.core.Arthas.<init>(Arthas.java:28)
        at com.taobao.arthas.core.Arthas.main(Arthas.java:113)
[ERROR] attach fail, targetPid: 2780

```
直接报错如上：com.sun.tools.attach.AttachNotSupportedException: Unable to attach to 64-bit pro
cess；之前在研究TPrfile或者DTrace时也有遇到，不过当时忽略了；此次决定解决。网上资料较多，但并没有找到可解决的。从日志来看涉及两个：1.JAVA_HOME: D:\work\java\jdk；2.Unable to attach to 64-bit pro
cess.而确认后D:\work\java\jdk确实是64位JDK，具体为什么呢？此处也是突然的灵感：根据之前使用JVisualJVM或者JConsole时，其连接远程JVM进程也有Atttach模式。

### Attach机制
- 那Attach机制是什么？说简单点就是jvm提供一种jvm进程间通信的能力，能让一个进程传命令给另外一个进程，并让它执行内部的一些操作，比如说我们为了让另外一个jvm进程把线程dump出来，那么我们跑了一个jstack的进程，然后传了个pid的参数，告诉它要哪个进程进行线程dump，既然是两个进程，那肯定涉及到进程间通信，以及传输协议的定义，比如要执行什么操作，传了什么参数等
- ”Attach Listener”和“Signal Dispatcher”，这两个线程是我们这次要讲的attach机制的关键，Attach Listener这个线程在jvm起来的时候可能并没有的。jvm在启动过程中可能并没有启动Attach Listener这个线程（而线程“Signal Dispatcher”了，顾名思义是处理信号的，这个线程是在jvm启动的时候就会创建的），可以通过jvm参数来启动；Attach Listener 线程是负责接收到外部的命令，而对该命令进行执行的并且把结果返回给发送者。在JVM启动的时候，如果没有指定 +StartAttachListener，该Attach Listener线程是不会启动的。

### Arthas问题分析
回到上面的运行java -jar arthas-boot.jar时报错Unable to attach to 64-bit process。那么原理可简单理解为，
