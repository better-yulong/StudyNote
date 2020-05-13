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




