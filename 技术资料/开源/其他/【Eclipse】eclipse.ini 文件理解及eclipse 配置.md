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
applicationXMI=org.eclipse.ui.workbench/LegacyIDE.e4xmi
awt.toolkit=sun.awt.windows.WToolkit
bytes_display=Bytes
eclipse.application=org.eclipse.ui.ide.workbench
eclipse.buildId=4.6.3.M20170301-0400
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
java.library.path=D:\work\ide\eclipse-jee-neon\eclipse;C:\Windows\Sun\Java\bin;C:\Windows\system32;C:\Windows;C:/Program Files/Java/jre1.8.0_112/bin/server;C:/Program Files/Java/jre1.8.0_112/bin;C:/Program Files/Java/jre1.8.0_112/lib/amd64;C:\ProgramData\Oracle\Java\javapath;D:\work\java\jdk;/bin;D:\work\java\jdk;/lib;C:\Windows\system32;C:\Windows;C:\Windows\System32\Wbem;C:\Windows\System32\WindowsPowerShell\v1.0\;C:\Program Files\TortoiseSVN\bin;d:\Program Files (x86)\Oracle\odac_client;d:\Program Files (x86)\Oracle\odac_client\bin;C:\Program Files\MySQL\MySQL Utilities 1.6\;D:\Program Files\PuTTY\;.;D:\work\java\jdk\bin;D:\work\maven3\bin;E:\Git\cmd;D:\work\ide\eclipse-jee-neon\eclipse;;.
java.runtime.name=Java(TM) SE Runtime Environment
java.runtime.version=1.8.0_112-b15
java.specification.name=Java Platform API Specification
java.specification.vendor=Oracle Corporation
java.specification.version=1.8
java.vendor=Oracle Corporation
java.vendor.url=http://java.oracle.com/
java.vendor.url.bug=http://bugreport.sun.com/bugreport/
java.version=1.8.0_112
java.vm.info=mixed mode
java.vm.name=Java HotSpot(TM) 64-Bit Server VM
java.vm.specification.name=Java Virtual Machine Specification
java.vm.specification.vendor=Oracle Corporation
java.vm.specification.version=1.8
java.vm.vendor=Oracle Corporation
java.vm.version=25.112-b15
line.separator=

org.apache.commons.logging.Log=org.apache.commons.logging.impl.NoOpLog
org.eclipse.equinox.launcher.splash.location=D:\work\ide\eclipse-jee-neon\eclipse\\plugins\org.eclipse.platform_4.6.3.v20170301-0400\splash.bmp
org.eclipse.equinox.simpleconfigurator.configUrl=file:org.eclipse.equinox.simpleconfigurator/bundles.info
org.eclipse.m2e.log.dir=D:\work\workspace\study-code-effective-java\.metadata\.plugins\org.eclipse.m2e.logback.configuration
org.eclipse.swt.internal.deviceZoom=100
org.eclipse.update.reconcile=false
org.osgi.framework.executionenvironment=OSGi/Minimum-1.0,OSGi/Minimum-1.1,OSGi/Minimum-1.2,JavaSE/compact1-1.8,JavaSE/compact2-1.8,JavaSE/compact3-1.8,JRE-1.1,J2SE-1.2,J2SE-1.3,J2SE-1.4,J2SE-1.5,JavaSE-1.6,JavaSE-1.7,JavaSE-1.8
org.osgi.framework.language=zh
org.osgi.framework.os.name=Windows7
org.osgi.framework.os.version=6.1.0
org.osgi.framework.processor=x86-64
org.osgi.framework.system.capabilities=osgi.ee; osgi.ee="OSGi/Minimum"; version:List<Version>="1.0, 1.1, 1.2",osgi.ee; osgi.ee="JRE"; version:List<Version>="1.0, 1.1",osgi.ee; osgi.ee="JavaSE"; version:List<Version>="1.0, 1.1, 1.2, 1.3, 1.4, 1.5, 1.6, 1.7, 1.8",osgi.ee; osgi.ee="JavaSE/compact1"; version:List<Version>="1.8",osgi.ee; osgi.ee="JavaSE/compact2"; version:List<Version>="1.8",osgi.ee; osgi.ee="JavaSE/compact3"; version:List<Version>="1.8"
org.osgi.framework.system.packages=javax.accessibility,javax.activation,javax.activity,javax.annotation,javax.annotation.processing,javax.crypto,javax.crypto.interfaces,javax.crypto.spec,javax.imageio,javax.imageio.event,javax.imageio.metadata,javax.imageio.plugins.bmp,javax.imageio.plugins.jpeg,javax.imageio.spi,javax.imageio.stream,javax.jws,javax.jws.soap,javax.lang.model,javax.lang.model.element,javax.lang.model.type,javax.lang.model.util,javax.management,javax.management.loading,javax.management.modelmbean,javax.management.monitor,javax.management.openmbean,javax.management.relation,javax.management.remote,javax.management.remote.rmi,javax.management.timer,javax.naming,javax.naming.directory,javax.naming.event,javax.naming.ldap,javax.naming.spi,javax.net,javax.net.ssl,javax.print,javax.print.attribute,javax.print.attribute.standard,javax.print.event,javax.rmi,javax.rmi.CORBA,javax.rmi.ssl,javax.script,javax.security.auth,javax.security.auth.callback,javax.security.auth.kerberos,javax.security.auth.login,javax.security.auth.spi,javax.security.auth.x500,javax.security.cert,javax.security.sasl,javax.sound.midi,javax.sound.midi.spi,javax.sound.sampled,javax.sound.sampled.spi,javax.sql,javax.sql.rowset,javax.sql.rowset.serial,javax.sql.rowset.spi,javax.swing,javax.swing.border,javax.swing.colorchooser,javax.swing.event,javax.swing.filechooser,javax.swing.plaf,javax.swing.plaf.basic,javax.swing.plaf.metal,javax.swing.plaf.multi,javax.swing.plaf.nimbus,javax.swing.plaf.synth,javax.swing.table,javax.swing.text,javax.swing.text.html,javax.swing.text.html.parser,javax.swing.text.rtf,javax.swing.tree,javax.swing.undo,javax.tools,javax.transaction,javax.transaction.xa,javax.xml,javax.xml.bind,javax.xml.bind.annotation,javax.xml.bind.annotation.adapters,javax.xml.bind.attachment,javax.xml.bind.helpers,javax.xml.bind.util,javax.xml.crypto,javax.xml.crypto.dom,javax.xml.crypto.dsig,javax.xml.crypto.dsig.dom,javax.xml.crypto.dsig.keyinfo,javax.xml.crypto.dsig.spec,javax.xml.datatype,javax.xml.namespace,javax.xml.parsers,javax.xml.soap,javax.xml.stream,javax.xml.stream.events,javax.xml.stream.util,javax.xml.transform,javax.xml.transform.dom,javax.xml.transform.sax,javax.xml.transform.stax,javax.xml.transform.stream,javax.xml.validation,javax.xml.ws,javax.xml.ws.handler,javax.xml.ws.handler.soap,javax.xml.ws.http,javax.xml.ws.soap,javax.xml.ws.spi,javax.xml.ws.spi.http,javax.xml.ws.wsaddressing,javax.xml.xpath,org.ietf.jgss,org.omg.CORBA,org.omg.CORBA_2_3,org.omg.CORBA_2_3.portable,org.omg.CORBA.DynAnyPackage,org.omg.CORBA.ORBPackage,org.omg.CORBA.portable,org.omg.CORBA.TypeCodePackage,org.omg.CosNaming,org.omg.CosNaming.NamingContextExtPackage,org.omg.CosNaming.NamingContextPackage,org.omg.Dynamic,org.omg.DynamicAny,org.omg.DynamicAny.DynAnyFactoryPackage,org.omg.DynamicAny.DynAnyPackage,org.omg.IOP,org.omg.IOP.CodecFactoryPackage,org.omg.IOP.CodecPackage,org.omg.Messaging,org.omg.PortableInterceptor,org.omg.PortableInterceptor.ORBInitInfoPackage,org.omg.PortableServer,org.omg.PortableServer.CurrentPackage,org.omg.PortableServer.POAManagerPackage,org.omg.PortableServer.POAPackage,org.omg.PortableServer.portable,org.omg.PortableServer.ServantLocatorPackage,org.omg.SendingContext,org.omg.stub.java.rmi,org.w3c.dom,org.w3c.dom.bootstrap,org.w3c.dom.css,org.w3c.dom.events,org.w3c.dom.html,org.w3c.dom.ls,org.w3c.dom.ranges,org.w3c.dom.stylesheets,org.w3c.dom.traversal,org.w3c.dom.views,org.w3c.dom.xpath,org.xml.sax,org.xml.sax.ext,org.xml.sax.helpers
org.osgi.framework.uuid=4f07154a-9468-4533-a6d9-f9ba1e6f62fa
org.osgi.framework.vendor=Eclipse
org.osgi.framework.version=1.8.0
org.osgi.supports.framework.extension=true
org.osgi.supports.framework.fragment=true
org.osgi.supports.framework.requirebundle=true
os.arch=amd64
os.name=Windows 7
os.version=6.1
osgi.arch=x86_64
osgi.bundles=reference:file:org.eclipse.osgi.compatibility.state_1.0.200.v20160504-1419.jar,reference:file:org.eclipse.wst.jsdt.nashorn.extension_1.0.2.v201610280128.jar,reference:file:org.eclipse.equinox.simpleconfigurator_1.1.200.v20160504-1450.jar@1:start
osgi.bundles.defaultStartLevel=4
osgi.compatibility.bootdelegation=true
osgi.compatibility.bootdelegation.default=true
osgi.configuration.area=file:/D:/work/ide/eclipse-jee-neon/eclipse/configuration/
osgi.framework=file:/d:/work/ide/eclipse-jee-neon/eclipse/plugins/org.eclipse.osgi_3.11.3.v20170209-1843.jar
osgi.framework.extensions=reference:file:org.eclipse.osgi.compatibility.state_1.0.200.v20160504-1419.jar,reference:file:org.eclipse.wst.jsdt.nashorn.extension_1.0.2.v201610280128.jar
osgi.framework.shape=jar
osgi.framework.useSystemProperties=true
osgi.frameworkClassPath=., file:d:/work/ide/eclipse-jee-neon/eclipse/plugins/org.eclipse.osgi.compatibility.state_1.0.200.v20160504-1419.jar, file:d:/work/ide/eclipse-jee-neon/eclipse/plugins/org.eclipse.wst.jsdt.nashorn.extension_1.0.2.v201610280128.jar
osgi.install.area=file:/D:/work/ide/eclipse-jee-neon/eclipse/
osgi.instance.area=file:/D:/work/workspace/study-code-effective-java/
osgi.instance.area.default=file:/C:/Users/483879/workspace/
osgi.logfile=D:\work\workspace\study-code-effective-java\.metadata\.log
osgi.nl=zh_CN
osgi.os=win32
osgi.requiredJavaVersion=1.7
osgi.splashLocation=D:\work\ide\eclipse-jee-neon\eclipse\\plugins\org.eclipse.platform_4.6.3.v20170301-0400\splash.bmp
osgi.splashPath=platform:/base/plugins/org.eclipse.platform
osgi.syspath=d:\work\ide\eclipse-jee-neon\eclipse\plugins
osgi.tracefile=D:\work\workspace\study-code-effective-java\.metadata\trace.log
osgi.ws=win32
path.separator=;
sun.arch.data.model=64
sun.boot.class.path=C:\Program Files\Java\jre1.8.0_112\lib\resources.jar;C:\Program Files\Java\jre1.8.0_112\lib\rt.jar;C:\Program Files\Java\jre1.8.0_112\lib\sunrsasign.jar;C:\Program Files\Java\jre1.8.0_112\lib\jsse.jar;C:\Program Files\Java\jre1.8.0_112\lib\jce.jar;C:\Program Files\Java\jre1.8.0_112\lib\charsets.jar;C:\Program Files\Java\jre1.8.0_112\lib\jfr.jar;C:\Program Files\Java\jre1.8.0_112\classes
sun.boot.library.path=C:\Program Files\Java\jre1.8.0_112\bin
sun.cpu.endian=little
sun.cpu.isalist=amd64
sun.desktop=windows
sun.io.unicode.encoding=UnicodeLittle
sun.jnu.encoding=GBK
sun.management.compiler=HotSpot 64-Bit Tiered Compilers
sun.os.patch.level=Service Pack 1
user.country=CN
user.dir=D:\work\ide\eclipse-jee-neon\eclipse
user.home=C:\Users\483879
user.language=zh
user.name=483879
user.script=
user.timezone=Asia/Shanghai
user.variant=

*** Features:
com.collabnet.subversion.merge.feature (4.1.0) "CollabNet Subversion Merge Client"
de.loskutov.BytecodeOutline.feature (2.5.0.201711011753-5a57fdf) "Bytecode Outline"
```
