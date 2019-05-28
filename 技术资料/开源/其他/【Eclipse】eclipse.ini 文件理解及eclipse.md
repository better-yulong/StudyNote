### Eclipse初步了解
之前，只简单的理解eclipse是基于Java语言开发的集成开发工具，了解到可通过修改eclipse.ini文件来指定eclipse启动使用的jdk、堆内存Xmx等来优化。但具体并未深入发解，而近期在研究Mybatis源码时发现在windows7平台使用jps -l 命令无法查看到主类名：
```language
C:\>jps -l
	7800
	6976
	6716 sun.tools.jps.Jps
	5920 com.zyl.base.io.SocketServer
```
查找度娘，网上主要有两类问题：
1. jps命令无法查看到 pid（Java进程ID）：jps工具原理是基于操作系统临时目录的pid文件来返回java进程信息，故没有权限读写临时目录或者pid已生成但没有权限读取均有可能出现该问题；---java启动时提供了参数(-Djava.io.tmpdir)可调整临时目录（https://trinea.iteye.com/blog/1196400）
2. jps可查看到pid，但未显示主类名（即上面示例）：暂未明确有解决办法（同时该jinfo 7800会异常，无法attach上；初步分析为jdk版本不匹配）

- 于是抱着试试看的态度，决定先调整临时目录，遂eclipse.ini文件添加 -Djava.io.tmpdir:D:/tmp（期望将临时目录调整至D盘），重新eclipse后jps 命令无效，而D:/tmp并没有效果。初步怀疑是配置错误，那具体该怎么配置呢？继续度娘但无有明确结果。但发现一新有用信息（Eclipse窗口：Help--> About Eclipse --> Installation Details --> Configuration），可查看Eclipse的全量配置信息，如下图：
[eclipse-configure](https://github.com/better-yulong/StudyNote-Resource/blob/master/StudyNote-Resource/tech/eclipse/eclipse-configure.PNG)
配置信息版本：
```language
-launcher
D:\work\ide\eclipse-jee-neon\eclipse\eclipse.exe
-name
Eclipse
-XX:+UseStringDeduplication
-Dosgi.requiredJavaVersion=1.7
-XX:MaxPermSize=1024M
-XX:PermSize=96m
-Xms512m
-Xmx512m
-Xverify:none
-Xmn128m
-XX:+DisableExplicitGC
-Xnoclassgc
-Dfile.encoding=UTF-8
-XX:+UseParNewGC
-XX:+UseConcMarkSweepGC
-XX:CMSInitiatingOccupancyFraction=85
-Xms256m
-Xmx2048m
-XX:-UseGCOverheadLimit
-Dhttps.protocols=TLSv1,TLSv1.1,TLSv1.2
-Djava.io.tmpdir:D:/tmp

java.home=C:\Program Files\Java\jre1.8.0_112
java.io.tmpdir=C:\Users\483879\AppData\Local\Temp\
java.io.tmpdir:D:/tmp=

java.version=1.8.0_112
java.vm.info=mixed mode
java.vm.name=Java HotSpot(TM) 64-Bit Server VM
java.vm.specification.name=Java Virtual Machine Specification
java.vm.specification.vendor=Oracle Corporation
java.vm.specification.version=1.8
java.vm.vendor=Oracle Corporation
java.vm.version=25.112-b15
line.separator=


*** Features:
com.collabnet.subversion.merge.feature (4.1.0) "CollabNet Subversion Merge Client"
de.loskutov.BytecodeOutline.feature (2.5.0.201711011753-5a57fdf) "Bytecode Outline"
```
其中这三行：
```language
-Djava.io.tmpdir:D:/tmp

java.home=C:\Program Files\Java\jre1.8.0_112
java.io.tmpdir=C:\Users\483879\AppData\Local\Temp\
java.io.tmpdir:D:/tmp=
user.home=C:\Users\483879
```
可看到eclipse.ini的修改有读取到新增配置，但貌似没生效；于是修改eclipse.ini文件重新Eclipse生效：
```language
	-Djava.io.tmpdir=D:\tmp
	-Duser.home=D:\tmp
```
但是仍然无法通过jps -l命令确认到主类名，其实是想确认哪个进程是eclispe的java进程，无奈选择中办法：1. jps -l 确认无其他Java进程；2.启动Eclipse; 3.jps -lmv 命令查看除jps之外的Java进程:
```language
5076  -Dosgi.requiredJavaVersion=1.7 -XX:+UseStringDeduplication -Dosgi.requiredJavaVersion=1.7 -XX:MaxPermSize=256M -XX:PermSize=96m -Xms512m -X
mx1024m -Xverify:none -Xmn128m -XX:+DisableExplicitGC -Xnoclassgc -Dfile.encoding=UTF-8 -XX:+UseParNewGC -XX:+UseConcMarkSweepGC -XX:CMSInitiatin
gOccupancyFraction=85 -XX:-UseGCOverheadLimit -Dhttps.protocols=TLSv1,TLSv1.1,TLSv1.2 
```
如果还觉得不完全确认，则可以修改eclipse.ini文件，根据参数来确认是否确实为eclise 自身Java进程（添加或修改参数 ）：
```language
	-Djava.io.tmpdir=D:\tmp
	-Dfile.encoding=UTF-8
	-Duser.home=D:\tmp
	-Xms256m
	-Xmx1024m
	-XX:MaxPermSize=256M
	-XX:PermSize=128m
	
```
再次 jps -lvm 命令查看：
```language
7668  -Dosgi.requiredJavaVersion=1.7 -XX:+UseStringDeduplication -Dosgi.requiredJavaVersion=1.7 -Xverify:none -XX:+DisableExplicitGC -Xnoclassgc
-XX:+UseParNewGC -XX:+UseConcMarkSweepGC -XX:CMSInitiatingOccupancyFraction=85 -XX:-UseGCOverheadLimit -Dhttps.protocols=TLSv1,TLSv1.1,TLSv1.2 -D
java.io.tmpdir=D:\tmp -Dfile.encoding=UTF-8 -Duser.home=D:\tmp -Xms256m -Xmx1024m -XX:MaxPermSize=256M -XX:PermSize=128m
```
此时虽然jps命令未打印出主类名，但可完全确认该进程即为eclipse的 jvm进程。
### Eclipse 调优
在eclipse.ini文件添加GC日志参数，并：
```language
	-verbose:gc
	-XX:+PrintGCDetails
	-XX:+PrintGCDateStamps
	-Xloggc:D:/tmp/eclipse-jee-gc.log
```


