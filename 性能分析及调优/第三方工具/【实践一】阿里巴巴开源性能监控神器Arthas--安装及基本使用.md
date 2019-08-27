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
cess.而确认后D:\work\java\jdk确实是64位JDK，具体为什么呢？此处也是突然的灵感：根据之前使用JVisualJVM或者JConsole时，其连接远程JVM进程也有这种模式；此模式为JDk