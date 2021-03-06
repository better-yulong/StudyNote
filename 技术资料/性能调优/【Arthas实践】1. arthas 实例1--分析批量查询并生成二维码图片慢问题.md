背景：
    基于业务需求，需批量查询500条数据并生成二维码图片（含logo图片、head文字、tail文字）压缩包，本地验证500条需10秒左右，分析看是否有进一步优化空间。

### 一. Arthas环境
1. 基于 https://alibaba.github.io/arthas/download.html   下载 arthas-boot-3.2.0.jar；然而执行 java -jar arthas-boot-3.2.0.jar ，提示错误：arthas-boot-3.2.0.jar中没有主清单属性 
2. 之后参考： https://www.jianshu.com/p/4e711a780aa3  从地址 https://alibaba.github.io/arthas/arthas-boot.jar 下载arthas-boot.jar 后执行 java -jar arthas-boot.jar 运行成功。

### 二.实践
说明：后续实践参考资料 https://www.jianshu.com/p/4e711a780aa3 。
d:\>java -jar arthas-boot.jar
[INFO] arthas-boot version: 3.2.0
[INFO] Found existing java process, please choose one and input the serial number of the process, eg : 1. Then hit ENTER.
* [1]: 14132 com.***.Application
  [2]: 8980 org.springframework.ide.vscode.boot.app.BootLanguagServerBootApp
  [3]: 9028 D:\Program
  [4]: 11980 D:\work\ide\eclipse\\plugins/org.eclipse.equinox.launcher_1.5.400.v20190515-0925.jar
输入1，选择监控第1个java进程：
1
[INFO] Start download arthas from remote server: https://maven.aliyun.com/repository/public/com/taobao/arthas/arthas-packaging/3.2.0/arthas-packaging-3.2.0-bi
n.zip
[INFO] File size: 10.82 MB, downloaded size: 1.09 MB, downloading ...
[INFO] File size: 10.82 MB, downloaded size: 2.21 MB, downloading ...
[INFO] File size: 10.82 MB, downloaded size: 3.34 MB, downloading ...
[INFO] File size: 10.82 MB, downloaded size: 4.47 MB, downloading ...
[INFO] File size: 10.82 MB, downloaded size: 5.59 MB, downloading ...
[INFO] File size: 10.82 MB, downloaded size: 6.72 MB, downloading ...
[INFO] File size: 10.82 MB, downloaded size: 7.85 MB, downloading ...
[INFO] File size: 10.82 MB, downloaded size: 8.98 MB, downloading ...
[INFO] File size: 10.82 MB, downloaded size: 10.11 MB, downloading ...
[INFO] Download arthas success.
[INFO] arthas home: C:\Users\M70CLJL1\.arthas\lib\3.2.0\arthas
[INFO] Try to attach process 14132
[INFO] Found java home from System Env JAVA_HOME: D:\Program Files\Java\jdk1.8.0_231
[INFO] Attach process 14132 success.
[INFO] arthas-client connect 127.0.0.1 3658
  ,---.  ,------. ,--------.,--.  ,--.  ,---.   ,---.  
 /  O  \ |  .--. ''--.  .--'|  '--'  | /  O  \ '   .-' 
|  .-.  ||  '--'.'   |  |   |  .--.  ||  .-.  |`.  `-. 
|  | |  ||  |\  \    |  |   |  |  |  ||  | |  |.-'    |
`--' `--'`--' '--'   `--'   `--'  `--'`--' `--'`-----' 
                                                       

wiki      https://alibaba.github.io/arthas
tutorials https://alibaba.github.io/arthas/arthas-tutorials
version   3.2.0
pid       14132
time      2020-05-13 10:12:16

[arthas@14132]$

命令行输入 help 查看帮助信息：
[arthas@14132]$ help
 NAME         DESCRIPTION
 help         Display Arthas Help
 keymap       Display all the available keymap for the specified connection.
 sc           Search all the classes loaded by JVM
 sm           Search the method of classes loaded by JVM
 classloader  Show classloader info
 jad          Decompile class
 getstatic    Show the static field of a class
 monitor      Monitor method execution statistics, e.g. total/success/failure count, average rt, fail rate, etc.
 stack        Display the stack trace for the specified class and method
 thread       Display thread info, thread stack
 trace        Trace the execution time of specified method invocation.
 watch        Display the input/output parameter, return object, and thrown exception of specified method invocation
 tt           Time Tunnel
 jvm          Display the target JVM information
 perfcounter  Display the perf counter infornation.
 ognl         Execute ognl expression.
 mc           Memory compiler, compiles java files into bytecode and class files in memory.
 redefine     Redefine classes. @see Instrumentation#redefineClasses(ClassDefinition...)
 dashboard    Overview of target jvm's thread, memory, gc, vm, tomcat info.
 dump         Dump class byte array from JVM
 heapdump     Heap dump
 options      View and change various Arthas options
 cls          Clear the screen
 reset        Reset all the enhanced classes
 version      Display Arthas version
 session      Display current session information
 sysprop      Display, and change the system properties.
 sysenv       Display the system env.
 vmoption     Display, and update the vm diagnostic options.
 logger       Print logger info, and update the logger level
 history      Display command history
 cat          Concatenate and print files
 echo         write arguments to the standard output
 pwd          Return working directory name
 mbean        Display the mbean information
 grep         grep command for pipes.
 tee          tee command for pipes.
 profiler     Async Profiler. https://github.com/jvm-profiling-tools/async-profiler
 stop         Stop/Shutdown Arthas server and exit the console.



monitor/watch/trace相关
请注意，这些命令，都通过字节码增强技术来实现的，会在指定类的方法中插入一些切面来实现数据统计和观测，因此在线上、预发使用时，请尽量明确需要观测的类、方法以及条件，诊断结束要执行或将增强过的类执行 命令。

monitor——方法执行监控
watch——方法执行数据观测
trace——方法内部调用路径，并输出方法路径上的每个节点上耗时
stack——输出当前方法被调用的调用路径
tt——方法执行数据的时空隧道，记录下指定方法每次调用的入参和返回信息，并能对这些不同的时间下调用进行观测
options
options——查看或设置Arthas全局开关

命令行执行：
[arthas@14132]$ monitor -help
 USAGE:                                                                                                                                                     
   monitor [-c <value>] [-h] [-n <value>] [-E <value>] class-pattern method-pattern

 SUMMARY:                                                                                                                                                   
   Monitor method execution statistics, e.g. total/success/failure count, average rt, fail rate, etc.
                                                                                                                                                            
 Examples:                                                                                                                                                  
   monitor org.apache.commons.lang.StringUtils isBlank
   monitor org.apache.commons.lang.StringUtils isBlank -c 5
   monitor -E org\.apache\.commons\.lang\.StringUtils isBlank
                                                                                                                                                            
 WIKI:                                                                                                                                                      
   https://alibaba.github.io/arthas/monitor

 OPTIONS:                                                                                                                                                   
 -c, --cycle <value>                                 The monitor interval (in seconds), 60 seconds by default
 -h, --help                                          this help
 -n, --limits <value>                                Threshold of execution times
 -E, --regex <value>                                 Enable regular expression to match (wildcard matching by default)
 <class-pattern>                                     Path and classname of Pattern Matching
 <method-pattern>                                    Method of Pattern Matching

[arthas@14132]$

执行  [arthas@14132]$ monitor com.****.merchant.service.impl.AbcServiceImpl batchExport  之后，同时调用 batchExport方法，等待一段时间后结果如下：
[arthas@14132]$ monitor com.****.merchant.service.impl.AbcServiceImpl batchExport
Press Q or Ctrl+C to abort.
Affect(class-cnt:1 , method-cnt:1) cost in 171 ms.
 timestamp            class                                                                          method       total  success  fail  avg-rt(ms)  fail-rat
                                                                                                                                                    e       
-------------------------------------------------------------------------------------------------------------------------------------------------------------
 2020-05-13 10:22:20  com.****.merchant.service.impl.AbcServiceImpl batchExport  1      1        0     17730.41    0.00%

使用 trace 命令分析对应方法内部各方法执行时间：
[arthas@14132]$ trace com.****.merchant.service.impl.AbcServiceImpl batchExport
Press Q or Ctrl+C to abort.
Affect(class-cnt:1 , method-cnt:1) cost in 129 ms.
`---ts=2020-05-13 10:25:36;thread_name=XNIO-2 task-50;id=5d;is_daemon=false;priority=5;TCCL=sun.misc.Launcher$AppClassLoader@18b4aac2
    `---[18037.4857ms] com.****.merchant.service.impl.AbcServiceImpl:batchExport()
        +---[0.0131ms] org.springframework.data.domain.Sort:<init>() #112
        +---[102.8094ms] com.***.merchant.dao.PicCodeDao:queryPicCodes() #113
        +---[min=0.0013ms,max=0.0084ms,total=0.0111ms,count=3] com.***.nosql.mongo.bean.PageResult:getList() #114
        +---[0.0018ms] com.***.nosql.mongo.bean.PageResult:getList() #120
        +---[0.0251ms] com.***.util.DateUtil:format() #121
        +---[0.0122ms] com.***.util.RandomUtils:randomStr() #121
        +---[min=9.0E-4ms,max=0.0132ms,total=0.275ms,count=200] com.***.merchant.model.PicCode:setStatus() #132
        +---[min=6.0E-4ms,max=0.0054ms,total=0.1964ms,count=200] com.***.merchant.model.PicCode:setExportTime() #133
        +---[min=1.5699ms,max=5.0556ms,total=780.6569ms,count=200] com.***.merchant.dao.PicCodeDao:updateByByCoded() #134
        +---[min=0.0012ms,max=0.0382ms,total=0.5144ms,count=200] com.***.merchant.model.PicCode:getBranchId() #136
        +---[min=7.0E-4ms,max=0.0079ms,total=0.2668ms,count=200] org.apache.commons.lang.StringUtils:isEmpty() #136
        +---[min=3.0E-4ms,max=0.224ms,total=0.7063ms,count=600] com.***.merchant.model.PicCode:getCodeId() #136
        +---[min=78.139ms,max=181.0064ms,total=16993.2396ms,count=200] com.****.merchant.service.impl.AbcServiceImpl:generateC
odePic() #136  （红色显示，该方法执行时间占比过长）
        +---[0.0143ms] org.slf4j.Logger:info() #139
        `---[111.1347ms] com.***.merchant.utils.ZipUtils:toZip() #146

结果可发现方法方法：         +---[min=78.139ms,max=181.0064ms,total=16993.2396ms,count=200] com.****.merchant.service.impl.AbcServiceImpl:generateC
odePic() #136  （红色显示，该方法执行时间占比过长）  执行时间过长。


再次监控 trace com.****.merchant.service.impl.AbcServiceImpl generateCodePic  ，发现结果如下：
`---ts=2020-05-13 10:31:45;thread_name=XNIO-2 task-23;id=3f;is_daemon=false;priority=5;TCCL=sun.misc.Launcher$AppClassLoader@18b4aac2
    `---[79.6039ms] com.***.service.impl.AbcServiceImpl:generateCodePic()
        +---[16.974ms] com.***.service.impl.AbcServiceImpl:createQRCodePic() #203
        +---[23.5487ms] com.***.service.impl.AbcServiceImpl:addLogo2QRCode() #204
        +---[18.9098ms] com.***.service.impl.AbcServiceImpl:pressText() #205
        `---[20.0809ms] com.***.service.impl.AbcServiceImpl:pressText() #206

`---ts=2020-05-13 10:31:45;thread_name=XNIO-2 task-23;id=3f;is_daemon=false;priority=5;TCCL=sun.misc.Launcher$AppClassLoader@18b4aac2
    `---[80.0009ms] com.***.service.impl.AbcServiceImpl:generateCodePic()
        +---[17.2176ms] com.***.service.impl.AbcServiceImpl:createQRCodePic() #203
        +---[23.4798ms] com.***.service.impl.AbcServiceImpl:addLogo2QRCode() #204
        +---[19.2752ms] com.***.service.impl.AbcServiceImpl:pressText() #205
        `---[19.925ms] com.***.service.impl.AbcServiceImpl:pressText() #206

可发现内部4个方法执行时间都非常长，而红色提示： AbcServiceImpl:addLogo2QRCode 方法则始终所花时间最长。但如果想减少整体执行时间，则还需要往下分析。



`---ts=2020-05-13 11:48:49;thread_name=XNIO-2 task-48;id=5a;is_daemon=false;priority=5;TCCL=sun.misc.Launcher$AppClassLoader@18b4aac2
    `---[23.415ms] com.***e.impl.AbcServiceImpl:addLogo2QRCode()
        +---[1.169ms] javax.imageio.ImageIO:read() #318
        +---[7.1518ms] javax.imageio.ImageIO:read() #322
        +---[0.0011ms] com.***.LogoConfig:getLogoPart() #324
        +---[3.0E-4ms] com.***.LogoConfig:getLogoPart() #325
        +---[9.0E-4ms] com.***.LogoConfig:getBorder() #334
        +---[4.0E-4ms] com.***.LogoConfig:getBorderColor() #335
        +---[14.6963ms] javax.imageio.ImageIO:write() #340
        `---[0.0445ms] org.slf4j.Logger:info() #343

`---ts=2020-05-13 11:48:50;thread_name=XNIO-2 task-48;id=5a;is_daemon=false;priority=5;TCCL=sun.misc.Launcher$AppClassLoader@18b4aac2
    `---[23.4341ms] com.***e.impl.AbcServiceImpl:addLogo2QRCode()
        +---[1.1841ms] javax.imageio.ImageIO:read() #318
        +---[7.0887ms] javax.imageio.ImageIO:read() #322
        +---[9.0E-4ms] com.***.LogoConfig:getLogoPart() #324
        +---[3.0E-4ms] com.***.LogoConfig:getLogoPart() #325
        +---[6.0E-4ms] com.***.LogoConfig:getBorder() #334
        +---[4.0E-4ms] com.***.LogoConfig:getBorderColor() #335
        +---[14.7828ms] javax.imageio.ImageIO:write() #340
        `---[0.0402ms] org.slf4j.Logger:info() #343



+---[7.0887ms] javax.imageio.ImageIO:read() #322      +---[14.7828ms] javax.imageio.ImageIO:write() #340

原因其实就在每次文件读取和写入。简单说下最原始的方法分几步：1. 生成二维码图片；2.基于二维码图片添加logo ；3.基于前一步图片添加title文件；4. 基于前一步图片添加 tail文件。整个过程中每步都读取文件、且在磁盘写入文件， 此种方法好处在于理解简单，而且在前期调试非常方便，可以确认每步处理结果；但从上面分析就可以看出频繁的read、write IO导致的性能极差。于是想到，真正需要的图片仅是最后一张，而之前的每步操作均为临时图片，是否可考虑不做本地IO，而是缓存内存中直接给后一步使用呢？
还真可以：
1. 由 MatrixToImageWriter.writeToFile 调整为  MatrixToImageWriter.toBufferedImage ，即每次处理图片生成对象均为 BufferedImage 对象，而不是直接存储到文件File文件；
2. 多步骤处理图片，也不是 ImageIO:read() 从 File 读取，而是基于前一步返回的 BufferedImage 对象 ；直到最后一步才写入文件。
3. Logo图片由每次读取也调整为使用 BufferedImage  循环使用。

再次验证，快了不只一星半点
    `---[0.1569ms] com.***.service.impl.CollectionCodeServiceImpl:addLogo2QRCode()
        +---[0.0024ms] com.***.config.LogoConfig:getLogoPart() #317
        +---[4.0E-4ms] com.***.config.LogoConfig:getLogoPart() #318
        +---[8.0E-4ms] com.***.config.LogoConfig:getBorder() #327
        +---[5.0E-4ms] com.***.config.LogoConfig:getBorderColor() #328
        `---[0.029ms] org.slf4j.Logger:info() #332

`---ts=2020-05-13 16:47:21;thread_name=XNIO-2 task-21;id=46;is_daemon=false;priority=5;TCCL=sun.misc.Launcher$AppClassLoader@18b4aac2
    `---[0.1526ms] com.***.service.impl.CollectionCodeServiceImpl:addLogo2QRCode()
        +---[0.0023ms] com.***.config.LogoConfig:getLogoPart() #317
        +---[5.0E-4ms] com.***.config.LogoConfig:getLogoPart() #318
        +---[7.0E-4ms] com.***.config.LogoConfig:getBorder() #327
        +---[7.0E-4ms] com.***.config.LogoConfig:getBorderColor() #328
        `---[0.0266ms] org.slf4j.Logger:info() #332

切换到父方法分析：
`---ts=2020-05-13 16:49:55;thread_name=XNIO-2 task-54;id=71;is_daemon=false;priority=5;TCCL=sun.misc.Launcher$AppClassLoader@18b4aac2
    `---[21.4896ms] com.***.service.impl.AbcCodeServiceImpl:generateCodePic()
        +---[2.8097ms] com.***.service.impl.AbcCodeServiceImpl:createQRCodePic() #204
        +---[0.1305ms] com.***.service.impl.AbcCodeServiceImpl:addLogo2QRCode() #205
        +---[0.1679ms] com.***.service.impl.AbcCodeServiceImpl:pressText() #206
        +---[0.1214ms] com.***.service.impl.AbcCodeServiceImpl:pressText() #207
        +---[17.7372ms] javax.imageio.ImageIO:write() #211
        `---[0.0406ms] org.slf4j.Logger:info() #212

现在慢的方法主要就是 javax.imageio.ImageIO:write() #211 。此时写入的图片是PNG格式，而调整成 JPEG格式后结果如下：
`---ts=2020-05-13 16:52:51;thread_name=XNIO-2 task-10;id=38;is_daemon=false;priority=5;TCCL=sun.misc.Launcher$AppClassLoader@18b4aac2
    `---[10.8664ms] com.***.service.impl.AbcCodeServiceImpl:generateCodePic()
        +---[3.0305ms] com.***.service.impl.AbcCodeServiceImpl:createQRCodePic() #204
        +---[0.1269ms] com.***.service.impl.AbcCodeServiceImpl:addLogo2QRCode() #205
        +---[0.1879ms] com.***.service.impl.AbcCodeServiceImpl:pressText() #206
        +---[0.1425ms] com.***.service.impl.AbcCodeServiceImpl:pressText() #207
        +---[6.8575ms] javax.imageio.ImageIO:write() #211
        `---[0.0444ms] org.slf4j.Logger:info() #212

小知识点： JPEG图片生成要比PNG图片生成快很多，但同时JPEG 图片的大小则差不多是PNG图片的2倍，总归是有代价的，PNG需要对图片分析做压缩，故花的时间更多，文件大小则减小。性能调优古老的选择，时间 or 空间?
 