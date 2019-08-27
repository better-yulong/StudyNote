1. https://blog.csdn.net/Testfan_zhou/article/details/92579810
2. https://alibaba.github.io/arthas/index.html



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
