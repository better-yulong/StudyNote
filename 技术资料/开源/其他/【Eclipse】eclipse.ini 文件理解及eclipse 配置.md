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
2. jps可查看到pid，但未显示主类名（即上面示例）：暂未明确有解决办法

于是抱着试试看的态度，决定先调整临时目录，遂eclipse.ini文件添加 -Djava.io.tmpdir:D:/tmp（期望将临时目录调整至D盘），重新eclipse后jps 命令无效，而D:/tmp并没有效果。初步怀疑是配置错误，那具体该怎么配置呢？继续度娘但无有明确结果。但发现一新有用信息（Eclipse窗口：Help--> About Eclipse --> Installation Details --> Configuration），可查看Eclipse的全量配置信息，如下图：
[eclipse-configure](https://github.com/better-yulong/StudyNote-Resource/blob/master/StudyNote-Resource/tech/eclipse/eclipse-configure.PNG)
配置信息版本：
```language
*** Date: 2019年5月28日星期二 中国标准时间 上午10:44:08

*** Platform Details:

*** System properties:
eclipse.commands=-os
win32
-ws
win32
-arch
x86_64
-showsplash
D:\work\ide\eclipse-jee-neon\eclipse\\plugins\org.eclipse.platform_4.6.3.v20170301-0400\splash.bmp
-launcher
D:\work\ide\eclipse-jee-neon\eclipse\eclipse.exe
-name
Eclipse
--launcher.library
D:\work\ide\eclipse-jee-neon\eclipse\\plugins/org.eclipse.equinox.launcher.win32.win32.x86_64_1.1.401.v20161122-1740\eclipse_1617.dll
-startup
D:\work\ide\eclipse-jee-neon\eclipse\\plugins/org.eclipse.equinox.launcher_1.3.201.v20161025-1711.jar
--launcher.appendVmargs
-product
org.eclipse.epp.package.jee.product
-vm
C:\Program Files\Java\jre1.8.0_112\bin\server\jvm.dll
eclipse.home.location=file:/D:/work/ide/eclipse-jee-neon/eclipse/
eclipse.launcher=D:\work\ide\eclipse-jee-neon\eclipse\eclipse.exe
eclipse.launcher.name=Eclipse
eclipse.p2.data.area=@config.dir/../p2/
eclipse.p2.profile=epp.package.jee
eclipse.product=org.eclipse.epp.package.jee.product
eclipse.startTime=1559011068981
eclipse.stateSaveDelayInterval=30000
eclipse.vm=C:\Program Files\Java\jre1.8.0_112\bin\server\jvm.dll
eclipse.vmargs=-Dosgi.requiredJavaVersion=1.7
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
-Djava.class.path=D:\work\ide\eclipse-jee-neon\eclipse\\plugins/org.eclipse.equinox.launcher_1.3.201.v20161025-1711.jar
equinox.init.uuid=true
equinox.use.ds=true
file.encoding=UTF-8
file.encoding.pkg=sun.io
file.separator=\
findbugs.cloud.default=edu.umd.cs.findbugs.cloud.doNothingCloud
findbugs.home=/D:/work/ide/eclipse-jee-neon/eclipse/plugins/edu.umd.cs.findbugs.plugin.eclipse_3.0.1.20150306-5afe4d1/
gosh.args=--nointeractive
guice.disable.misplaced.annotation.check=true
https.protocols=TLSv1,TLSv1.1,TLSv1.2
java.awt.graphicsenv=sun.awt.Win32GraphicsEnvironment
java.awt.printerjob=sun.awt.windows.WPrinterJob
java.class.path=D:\work\ide\eclipse-jee-neon\eclipse\\plugins/org.eclipse.equinox.launcher_1.3.201.v20161025-1711.jar
java.class.version=52.0
java.endorsed.dirs=C:\Program Files\Java\jre1.8.0_112\lib\endorsed
java.ext.dirs=C:\Program Files\Java\jre1.8.0_112\lib\ext;C:\Windows\Sun\Java\lib\ext
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
