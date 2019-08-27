- 近期基于业务需要，正好在对交易进行性能测试，其中交易研发同学根据服务调用日志反馈记账接口偶尔处理时长超过200ms，于是此基于此，决定使用Arthas进行定位。
- 参考资料：https://blog.csdn.net/Testfan_zhou/article/details/92579810

### 一.Linux Arthas安装及验证
1. 根据Arthas官网（https://alibaba.github.io/arthas/index.html），选择全量安装模式下载arthas-packaging-3.1.1-bin.7z 并解压缩；
2. 以cmd模式进入对应目录，根据文档使用arthas-boot.jar，直接用java -jar的方式启动：
```language
[tomcat@localhost arthas]$ java -jar arthas-boot.jar
[INFO] arthas-boot version: 3.1.1
[INFO] Found existing java process, please choose one and hit RETURN.
* [1]: 411 org.apache.catalina.startup.Bootstrap
1
[INFO] arthas home: /tmp/arthas
[INFO] Try to attach process 411
[INFO] Attach process 411 success.
[INFO] arthas-client connect 127.0.0.1 3658
  ,---.  ,------. ,--------.,--.  ,--.  ,---.   ,---.                           
 /  O  \ |  .--. ''--.  .--'|  '--'  | /  O  \ '   .-'                          
|  .-.  ||  '--'.'   |  |   |  .--.  ||  .-.  |`.  `-.                          
|  | |  ||  |\  \    |  |   |  |  |  ||  | |  |.-'    |                         
`--' `--'`--' '--'   `--'   `--'  `--'`--' `--'`-----'                          
                                                                                

wiki      https://alibaba.github.io/arthas                                      
tutorials https://alibaba.github.io/arthas/arthas-tutorials                     
version   3.1.1                                                                 
pid       411                                                                   
time      2019-08-27 15:32:28                                          
```
3. 运行直接成功（因为服务器并不像我本地有多个JDK版本，故不会出现之前Windows平台的问题）

近期基于业务需要，正好在对交易进行性能测试，其中交易研发同学根据服务调用日志反馈记账接口偶尔处理时长超过200ms，于是此基于此，决定使用Arthas进行定位。

#### 1.1 整体dashboard数据
输入dashboard，会实时展示当前tomcat的多线程状态、Jvm各区域、GC情况等信息；默认会定时刷新。
```language
$ dashboard
ID      NAME             GROUP    PRIORITY      STATE         %CPU    TIME    INTERRUPTED   DAEMON        
151     Timer-for-arthas-dashboard-6638fc13-c253- system          10      RUNNABLE      94      0:0     false         true    
145     nioEventLoopGroup-3-1         system          10      RUNNABLE      5       0:0     false         false         
16      AsyncAppender-Worker-Thread-4       main     5       WAITING       0       31:11         false         true    
17      AsyncAppender-Worker-Thread-5       main     5       WAITING       0       114:40        false         true    
18      AsyncAppender-Worker-Thread-6       main     5       WAITING       0       0:0     false         true    
19      AsyncAppender-Worker-Thread-7       main     5       WAITING       0       7:36    false         true    
141     AsyncAppender-Worker-arthas-cache.result. system          9       WAITING       0       0:0     false         true    
139     Attach Listener        system          9       RUNNABLE      0       0:0     false         true    
21      ContainerBackgroundProcessor[StandardEngi main     5       TIMED_WAITING 0       1:12    false         true    
3       Finalizer        system          8       WAITING       0       0:0     false         true    
11      GC Daemon        system          2       TIMED_WAITING 0       0:0     false         true    
20      IdleConnectionMonitor         main     5       TIMED_WAITING 0       0:4     false         true    
12      NioBlockingSelector.BlockPoller-1         main     5       RUNNABLE      0       5:53    false         true    
13      PoolCleaner[1294508613:1565939850879]     main     5       TIMED_WAITING 0       0:48    false         true    
2       Reference Handler      system          10      WAITING       0       0:0     false         true    
5       Signal Dispatcher      system          9       RUNNABLE      0       0:0     false         true    
25      ajp-bio-8031-Acceptor-0             main     5       RUNNABLE      0       0:0     false         true    
26      ajp-bio-8031-AsyncTimeout           main     5       TIMED_WAITING 0       0:45    false         true    
150     as-command-execute-daemon           system          10      TIMED_WAITING 0       0:0     false         true    
28      catalina-exec-1        main     5       WAITING       0       6:36    false         true    
37      catalina-exec-10       main     5       WAITING       0       6:32    false         true    
127     catalina-exec-100      main     5       WAITING       0       6:30    false         true    
137     catalina-exec-101      main     5       WAITING       0       0:14    false         true    
138     catalina-exec-102      main     5       WAITING       0       0:12    false         true    
38      catalina-exec-11       main     5       WAITING       0       6:31    false         true    
39      catalina-exec-12       main     5       WAITING       0       6:30    false         true    
40      catalina-exec-13       main     5       WAITING       0       6:29    false         true    
41      catalina-exec-14       main     5       WAITING       0       6:27    false         true    
42      catalina-exec-15       main     5       WAITING       0       6:28    false         true    
43      catalina-exec-16       main     5       WAITING       0       6:31    false         true    
Memory     used        total       max         usage       GC             
heap       532M        1981M       1981M       26.87%      gc.parnew.count        48935            
par_eden_space          290M        532M        532M        54.54%      gc.parnew.time(ms)     1048874          
par_survivor_space            408K        68096K      68096K      0.60%       gc.concurrentmarksweep.count        0          
cms_old_gen      241M        1382M       1382M       17.47%      gc.concurrentmarksweep.time(ms)     0          
nonheap    50M         265M        304M        16.76%               
code_cache       8M    9M    48M         18.45%               
cms_perm_gen     42M         256M        256M        16.46%               
direct     816K        816K        -     100.00%              
mapped     0K    0K    -     NaN%          
                
                 
Runtime                
os.name        Linux          
os.version           2.6.32-642.el6.x86_64             
java.version         1.7.0_99             
java.home            /usr/lib/jvm/java-1.7.0-openjdk-1.7.0.99.x86_64/jre        
systemload.average                0.21           
processors           4              
uptime         951539s        
                   
```

#### 1.2 线程监控
##### 输入thread会显示所有线程的状态信息
```language
$ thread
Threads Total: 129, NEW: 0, RUNNABLE: 31, BLOCKED: 0, WAITING: 7, TIMED_WAITING: 91, TERMINATED: 0     
ID       NAME        GROUP        PRIORITY      STATE    %CPU     TIME     INTERRUPTED   DAEMON        
17       AsyncAppender-Worker-Thread-5        main    5        WAITING       16       115:14        false    true     
16       AsyncAppender-Worker-Thread-4        main    5        WAITING       5        31:23    false    true     
127      catalina-exec-100     main    5        TIMED_WAITING 5        6:32     false    true     
43       catalina-exec-16      main    5        TIMED_WAITING 5        6:33     false    true     
30       catalina-exec-3       main    5        TIMED_WAITING 5        6:36     false    true     
82       catalina-exec-55      main    5        TIMED_WAITING 5        6:31     false    true     
39       catalina-exec-12      main    5        TIMED_WAITING 4        6:32     false    true     
57       catalina-exec-30      main    5        TIMED_WAITING 4        6:33     false    true     
58       catalina-exec-31      main    5        TIMED_WAITING 4        6:33     false    true     
66       catalina-exec-39      main    5        TIMED_WAITING 4        6:30     false    true     
31       catalina-exec-4       main    5        TIMED_WAITING 4        6:32     false    true     
69       catalina-exec-42      main    5        TIMED_WAITING 4        6:32     false    true     
38       catalina-exec-11      main    5        TIMED_WAITING 1        6:33     false    true     
44       catalina-exec-17      main    5        TIMED_WAITING 1        6:32     false    true     
29       catalina-exec-2       main    5        TIMED_WAITING 1        6:35     false    true     
62       catalina-exec-35      main    5        TIMED_WAITING 1        6:35     false    true     
65       catalina-exec-38      main    5        TIMED_WAITING 1        6:33     false    true     
32       catalina-exec-5       main    5        TIMED_WAITING 1        6:33     false    true     
77       catalina-exec-50      main    5        TIMED_WAITING 1        6:32     false    true     
81       catalina-exec-54      main    5        TIMED_WAITING 1        6:31     false    true     
83       catalina-exec-56      main    5        TIMED_WAITING 1        6:33     false    true     
92       catalina-exec-65      main    5        TIMED_WAITING 1        6:33     false    true     
102      catalina-exec-75      main    5        TIMED_WAITING 1        6:33     false    true     
103      catalina-exec-76      main    5        TIMED_WAITING 1        6:29     false    true     
108      catalina-exec-81      main    5        TIMED_WAITING 1        6:32     false    true     
120      catalina-exec-93      main    5        TIMED_WAITING 1        6:33     false    true     
18       AsyncAppender-Worker-Thread-6        main    5        WAITING       0        0:0      false    true     
19       AsyncAppender-Worker-Thread-7        main    5        WAITING       0        7:38     false    true     
141      AsyncAppender-Worker-arthas-cache.result. system       9        WAITING       0        0:0      false    true     
139      Attach Listener       system       9        RUNNABLE      0        0:0      false    true     
21       ContainerBackgroundProcessor[StandardEngi main    5        TIMED_WAITING 0        1:12     false    true     
3        Finalizer   system       8        WAITING       0        0:0      false    true     
11       GC Daemon   system       2        TIMED_WAITING 0        0:0      false    true     
20       IdleConnectionMonitor      main    5        TIMED_WAITING 0        0:4      false    true     
12       NioBlockingSelector.BlockPoller-1    main    5        RUNNABLE      0        5:55     false    true     
13       PoolCleaner[1294508613:1565939850879]     main    5        TIMED_WAITING 0        0:48     false    true     
2        Reference Handler     system       10       WAITING       0        0:0      false    true     
5        Signal Dispatcher     system       9        RUNNABLE      0        0:0      false    true     
25       ajp-bio-8031-Acceptor-0    main    5        RUNNABLE      0        0:0      false    true     
26       ajp-bio-8031-AsyncTimeout  main    5        TIMED_WAITING 0        0:45     false    true     
156      as-command-execute-daemon  system       10       RUNNABLE      0        0:0      false    true     
28       catalina-exec-1       main    5        TIMED_WAITING 0        6:38     false    true     
37       catalina-exec-10      main    5        TIMED_WAITING 0        6:34     false    true     
137      catalina-exec-101     main    5        TIMED_WAITING 0        0:16     false    true     
138      catalina-exec-102     main    5        TIMED_WAITING 0        0:14     false    true     
152      catalina-exec-103     main    5        TIMED_WAITING 0        0:2      false    true     
153      catalina-exec-104     main    5        TIMED_WAITING 0        0:2      false    true     
40       catalina-exec-13      main    5        TIMED_WAITING 0        6:31     false    true     
41       catalina-exec-14      main    5        TIMED_WAITING 0        6:29     false    true     
42       catalina-exec-15      main    5        TIMED_WAITING 0        6:30     false    true     
46       catalina-exec-19      main    5        TIMED_WAITING 0        6:31     false    true     
47       catalina-exec-20      main    5        TIMED_WAITING 0        6:32     false    true     
48       catalina-exec-21      main    5        TIMED_WAITING 0        6:33     false    true     
49       catalina-exec-22      main    5        TIMED_WAITING 0        6:32     false    true     
50       catalina-exec-23      main    5        TIMED_WAITING 0        6:32     false    true     
51       catalina-exec-24      main    5        TIMED_WAITING 0        6:35     false    true     
52       catalina-exec-25      main    5        TIMED_WAITING 0        6:33     false    true     
53       catalina-exec-26      main    5        TIMED_WAITING 0        6:33     false    true     
54       catalina-exec-27      main    5        TIMED_WAITING 0        6:30     false    true     
55       catalina-exec-28      main    5        TIMED_WAITING 0        6:31     false    true     
56       catalina-exec-29      main    5        TIMED_WAITING 0        6:33     false    true     
59       catalina-exec-32      main    5        TIMED_WAITING 0        6:31     false    true     
60       catalina-exec-33      main    5        TIMED_WAITING 0        6:31     false    true     
61       catalina-exec-34      main    5        TIMED_WAITING 0        6:35     false    true     
63       catalina-exec-36      main    5        TIMED_WAITING 0        6:32     false    true     
64       catalina-exec-37      main    5        TIMED_WAITING 0        6:35     false    true     
67       catalina-exec-40      main    5        TIMED_WAITING 0        6:34     false    true     
68       catalina-exec-41      main    5        TIMED_WAITING 0        6:32     false    true     
70       catalina-exec-43      main    5        TIMED_WAITING 0        6:32     false    true     
71       catalina-exec-44      main    5        TIMED_WAITING 0        6:33     false    true     
72       catalina-exec-45      main    5        TIMED_WAITING 0        6:32     false    true     
73       catalina-exec-46      main    5        TIMED_WAITING 0        6:31     false    true     
74       catalina-exec-47      main    5        TIMED_WAITING 0        6:34     false    true     
75       catalina-exec-48      main    5        TIMED_WAITING 0        6:33     false    true     
76       catalina-exec-49      main    5        TIMED_WAITING 0        6:31     false    true     
78       catalina-exec-51      main    5        TIMED_WAITING 0        6:31     false    true     
79       catalina-exec-52      main    5        TIMED_WAITING 0        6:34     false    true     
80       catalina-exec-53      main    5        TIMED_WAITING 0        6:31     false    true     
84       catalina-exec-57      main    5        TIMED_WAITING 0        6:30     false    true     
85       catalina-exec-58      main    5        TIMED_WAITING 0        6:34     false    true     
86       catalina-exec-59      main    5        TIMED_WAITING 0        6:29     false    true     
33       catalina-exec-6       main    5        TIMED_WAITING 0        6:31     false    true     
87       catalina-exec-60      main    5        TIMED_WAITING 0        6:30     false    true     
88       catalina-exec-61      main    5        TIMED_WAITING 0        6:33     false    true     
89       catalina-exec-62      main    5        TIMED_WAITING 0        6:33     false    true     
90       catalina-exec-63      main    5        TIMED_WAITING 0        6:30     false    true     
91       catalina-exec-64      main    5        TIMED_WAITING 0        6:34     false    true     
93       catalina-exec-66      main    5        TIMED_WAITING 0        6:31     false    true     
94       catalina-exec-67      main    5        TIMED_WAITING 0        6:34     false    true     
95       catalina-exec-68      main    5        TIMED_WAITING 0        6:32     false    true     
96       catalina-exec-69      main    5        TIMED_WAITING 0        6:33     false    true     
34       catalina-exec-7       main    5        TIMED_WAITING 0        6:33     false    true     
97       catalina-exec-70      main    5        TIMED_WAITING 0        6:32     false    true     
98       catalina-exec-71      main    5        TIMED_WAITING 0        6:31     false    true     
99       catalina-exec-72      main    5        TIMED_WAITING 0        6:31     false    true     
100      catalina-exec-73      main    5        TIMED_WAITING 0        6:31     false    true     
101      catalina-exec-74      main    5        TIMED_WAITING 0        6:35     false    true     
104      catalina-exec-77      main    5        TIMED_WAITING 0        6:32     false    true     
105      catalina-exec-78      main    5        TIMED_WAITING 0        6:32     false    true     
35       catalina-exec-8       main    5        TIMED_WAITING 0        6:33     false    true     
107      catalina-exec-80      main    5        TIMED_WAITING 0        6:35     false    true     
109      catalina-exec-82      main    5        TIMED_WAITING 0        6:30     false    true     
110      catalina-exec-83      main    5        TIMED_WAITING 0        6:28     false    true     
111      catalina-exec-84      main    5        TIMED_WAITING 0        6:32     false    true     
112      catalina-exec-85      main    5        TIMED_WAITING 0        6:31     false    true     
113      catalina-exec-86      main    5        TIMED_WAITING 0        6:29     false    true     
114      catalina-exec-87      main    5        TIMED_WAITING 0        6:32     false    true     
115      catalina-exec-88      main    5        TIMED_WAITING 0        6:31     false    true     
116      catalina-exec-89      main    5        TIMED_WAITING 0        6:32     false    true     
36       catalina-exec-9       main    5        TIMED_WAITING 0        6:33     false    true     
117      catalina-exec-90      main    5        TIMED_WAITING 0        6:33     false    true     
118      catalina-exec-91      main    5        TIMED_WAITING 0        6:32     false    true     
119      catalina-exec-92      main    5        TIMED_WAITING 0        6:31     false    true     
121      catalina-exec-94      main    5        TIMED_WAITING 0        6:30     false    true     
122      catalina-exec-95      main    5        TIMED_WAITING 0        6:35     false    true     
123      catalina-exec-96      main    5        TIMED_WAITING 0        6:32     false    true     
124      catalina-exec-97      main    5        TIMED_WAITING 0        6:33     false    true     
125      catalina-exec-98      main    5        TIMED_WAITING 0        6:31     false    true     
126      catalina-exec-99      main    5        TIMED_WAITING 0        6:32     false    true     
24       http-nio-8001-Acceptor-0   main    5        RUNNABLE      0        0:46     false    true     
22       http-nio-8001-ClientPoller-0         main    5        RUNNABLE      0        6:26     false    true     
23       http-nio-8001-ClientPoller-1         main    5        RUNNABLE      0        6:24     false    true     
143      job-timeout system       9        TIMED_WAITING 0        0:0      false    true     
1        main        main    5        RUNNABLE      0        0:1      false    false    
144      nioEventLoopGroup-2-1      system       10       RUNNABLE      0        0:0      false    false    
148      nioEventLoopGroup-2-2      system       10       RUNNABLE      0        0:2      false    false    
145      nioEventLoopGroup-3-1      system       10       RUNNABLE      0        0:0      false    false    
146      pool-2-thread-1       system       5        TIMED_WAITING 0        0:0      false    false    
147      pool-3-thread-1       system       5        WAITING       0        0:0      false    false    
Affect(row-cnt:0) cost in 120 ms.
```
##### 输入thread -n 3会显示当前最忙的3个线程，可以用来排查线程CPU消耗
```language
$ thread -n 3
"as-command-execute-daemon" Id=157 cpuUsage=100% RUNNABLE
    at sun.management.ThreadImpl.dumpThreads0(Native Method)
    at sun.management.ThreadImpl.getThreadInfo(ThreadImpl.java:440)
    at com.taobao.arthas.core.command.monitor200.ThreadCommand.processTopBusyThreads(ThreadCommand.java:133)
    at com.taobao.arthas.core.command.monitor200.ThreadCommand.process(ThreadCommand.java:79)
    at com.taobao.arthas.core.shell.command.impl.AnnotatedCommandImpl.process(AnnotatedCommandImpl.java:82)
    at com.taobao.arthas.core.shell.command.impl.AnnotatedCommandImpl.access$100(AnnotatedCommandImpl.java:18)
    at com.taobao.arthas.core.shell.command.impl.AnnotatedCommandImpl$ProcessHandler.handle(AnnotatedCommandImpl.java:111)
    at com.taobao.arthas.core.shell.command.impl.AnnotatedCommandImpl$ProcessHandler.handle(AnnotatedCommandImpl.java:108)
    at com.taobao.arthas.core.shell.system.impl.ProcessImpl$CommandProcessTask.run(ProcessImpl.java:370)
    at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1145)
    at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:615)
    at java.lang.Thread.run(Thread.java:745)

    Number of locked synchronizers = 1
    - java.util.concurrent.ThreadPoolExecutor$Worker@21423662


"Reference Handler" Id=2 cpuUsage=0% WAITING on java.lang.ref.Reference$Lock@15cc6d80
    at java.lang.Object.wait(Native Method)
    -  waiting on java.lang.ref.Reference$Lock@15cc6d80
    at java.lang.Object.wait(Object.java:503)
    at java.lang.ref.Reference$ReferenceHandler.run(Reference.java:133)


"Finalizer" Id=3 cpuUsage=0% WAITING on java.lang.ref.ReferenceQueue$Lock@2028f9ae
    at java.lang.Object.wait(Native Method)
    -  waiting on java.lang.ref.ReferenceQueue$Lock@2028f9ae
    at java.lang.ref.ReferenceQueue.remove(ReferenceQueue.java:135)
    at java.lang.ref.ReferenceQueue.remove(ReferenceQueue.java:151)
    at java.lang.ref.Finalizer$FinalizerThread.run(Finalizer.java:209)


Affect(row-cnt:0) cost in 1038 ms
```
##### 输入thread -b 会显示当前处于BLOCKED状态的线程，可以排查线程锁的问题
```language
$ thread -b
No most blocking thread found!
Affect(row-cnt:0) cost in 249 ms.
```

#### 1.3 jvm监控
输入jvm，查看jvm详细的性能数据
```language
$ jvm
 RUNTIME                                                                                                                                                                
------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 MACHINE-NAME                                    411@localhost                                                                                                          
 JVM-START-TIME                                  2019-08-16 15:17:29                                                                                                    
 MANAGEMENT-SPEC-VERSION                         1.2                                                                                                                    
 SPEC-NAME                                       Java Virtual Machine Specification                                                                                     
 SPEC-VENDOR                                     Oracle Corporation                                                                                                     
 SPEC-VERSION                                    1.7                                                                                                                    
 VM-NAME                                         OpenJDK 64-Bit Server VM                                                                                               
 VM-VENDOR                                       Oracle Corporation                                                                                                     
 VM-VERSION                                      24.95-b01                                                                                                              
 INPUT-ARGUMENTS                                 -Djava.util.logging.config.file=/tomcat/xxx/conf/logging.properties                                       
                                                 -Djava.util.logging.manager=org.apache.juli.ClassLoaderLogManager                                                      
                                                 -Xms2048m                                                                                                              
                                                 -Xmx2048m                                                                                                              
                                                 -XX:PermSize=256M                                                                                                      
                                                 -XX:MaxPermSize=256M                                                                                                   
                                                 -Xss256k                                                                                                               
                                                 -XX:+UseConcMarkSweepGC                                                                                                
                                                 -XX:+UseParNewGC                                                                                                       
                                                 -XX:-CMSParallelRemarkEnabled                                                                                          
                                                 -XX:ParallelGCThreads=8                                                                                                
                                                 -XX:MaxTenuringThreshold=10                                                                                            
                                                 -XX:GCTimeRatio=19                                                                                                     
                                                 -XX:+DisableExplicitGC                                                                                                 
                                                 -Djava.awt.headless=true                                                                                               
                                                 -Djava.endorsed.dirs=/tomcat/xxx/endorsed                                                                 
                                                 -Dcatalina.base=/tomcat/xxx                                                                               
                                                 -Dcatalina.home=/tomcat/xxx                                                                               
                                                 -Djava.io.tmpdir=/tomcat/xxx/temp                                                                         
                                                                                                                                                                        
 CLASS-PATH                                      /tomcat:/tomcat/xxx/bin/bootstrap.jar:/tomcat/platform-account/bin/tomcat-juli.jar                        
 BOOT-CLASS-PATH                                 /usr/lib/jvm/java-1.7.0-openjdk-1.7.0.99.x86_64/jre/lib/resources.jar:/usr/lib/jvm/java-1.7.0-openjdk-1.7.0.99.x86_64/ 
                                                 jre/lib/rt.jar:/usr/lib/jvm/java-1.7.0-openjdk-1.7.0.99.x86_64/jre/lib/sunrsasign.jar:/usr/lib/jvm/java-1.7.0-openjdk- 
                                                 1.7.0.99.x86_64/jre/lib/jsse.jar:/usr/lib/jvm/java-1.7.0-openjdk-1.7.0.99.x86_64/jre/lib/jce.jar:/usr/lib/jvm/java-1.7 
                                                 .0-openjdk-1.7.0.99.x86_64/jre/lib/charsets.jar:/usr/lib/jvm/java-1.7.0-openjdk-1.7.0.99.x86_64/jre/lib/rhino.jar:/usr 
                                                 /lib/jvm/java-1.7.0-openjdk-1.7.0.99.x86_64/jre/lib/jfr.jar:/usr/lib/jvm/java-1.7.0-openjdk-1.7.0.99.x86_64/jre/classe 
                                                 s                                                                                                                      
 LIBRARY-PATH                                    /usr/java/packages/lib/amd64:/usr/lib64:/lib64:/lib:/usr/lib                                                           
                                                                                                                                                                        
------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 CLASS-LOADING                                                                                                                                                          
------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 LOADED-CLASS-COUNT                              7238                                                                                                                   
 TOTAL-LOADED-CLASS-COUNT                        7238                                                                                                                   
 UNLOADED-CLASS-COUNT                            0                                                                                                                      
 IS-VERBOSE                                      false                                                                                                                  
                                                                                                                                                                        
------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 COMPILATION                                                                                                                                                            
------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 NAME                                            HotSpot 64-Bit Tiered Compilers                                                                                        
 TOTAL-COMPILE-TIME                              60348(ms)                                                                                                              
                                                                                                                                                                        
------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 GARBAGE-COLLECTORS                                                                                                                                                     
------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 ParNew                                          49548/1059030(ms)                                                                                                      
 [count/time]                                                                                                                                                           
 ConcurrentMarkSweep                             0/0(ms)                                                                                                                
 [count/time]                                                                                                                                                           
                                                                                                                                                                        
------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 MEMORY-MANAGERS                                                                                                                                                        
------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 CodeCacheManager                                Code Cache                                                                                                             
                                                                                                                                                                        
 ParNew                                          Par Eden Space                                                                                                         
                                                 Par Survivor Space                                                                                                     
                                                                                                                                                                        
 ConcurrentMarkSweep                             Par Eden Space                                                                                                         
                                                 Par Survivor Space                                                                                                     
                                                 CMS Old Gen                                                                                                            
                                                 CMS Perm Gen                                                                                                           
                                                                                                                                                                        
                                                                                                                                                                        
------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 MEMORY                                                                                                                                                                 
------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 HEAP-MEMORY-USAGE                               2077753344(1.94 GiB)/2147483648(2.00 GiB)/2077753344(1.94 GiB)/775401976(739.48 MiB)                                   
 [committed/init/max/used]                                                                                                                                              
 NO-HEAP-MEMORY-USAGE                            278462464(265.56 MiB)/270991360(258.44 MiB)/318767104(304.00 MiB)/54271784(51.76 MiB)                                  
 [committed/init/max/used]                                                                                                                                              
 PENDING-FINALIZE-COUNT                          0                                                                                                                      
                                                                                                                                                                        
------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 OPERATING-SYSTEM                                                                                                                                                       
------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 OS                                              Linux                                                                                                                  
 ARCH                                            amd64                                                                                                                  
 PROCESSORS-COUNT                                4                                                                                                                      
 LOAD-AVERAGE                                    0.6                                                                                                                    
 VERSION                                         2.6.32-642.el6.x86_64                                                                                                  
                                                                                                                                                                        
------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 THREAD                                                                                                                                                                 
------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 COUNT                                           127                                                                                                                    
 DAEMON-COUNT                                    121                                                                                                                    
 PEAK-COUNT                                      130                                                                                                                    
 STARTED-COUNT                                   150                                                                                                                    
 DEADLOCK-COUNT                                  0                                                                                                                      
                                                                                                                                                                        
------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 FILE-DESCRIPTOR                                                                                                                                                        
------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 MAX-FILE-DESCRIPTOR-COUNT                       65535                                                                                                                  
 OPEN-FILE-DESCRIPTOR-COUNT                      135                                                                                                                    
Affect(row-cnt:0) cost in 408 ms.
```

#### 1.4 函数耗时监控
- 这一步的使用终于明白了Arthas这神器的强大之外，虽然刚开始用，但真的叹为观止，值得膜拜，秒杀了其他太多工具，效率高了N倍。
1. 压测方法入口为TransferServiceImpl的combineTransfer方法，通过 trace -j com.sfpay.coreplatform.account.service.impl.TransferServiceImpl combineTransfer可发现耗时方法为transfer；
2. 监控transfer方法，trace -j com.sfpay.coreplatform.account.service.impl.TransferServiceImpl transfer可发
trace -j com.sfpay.coreplatform.account.service.impl.TransferServiceImpl doTransfer
```language

`---ts=2019-08-27 16:24:26;thread_name=catalina-exec-8;id=23;is_daemon=true;priority=5;TCCL=org.apache.catalina.loader.WebappClassLoader@eea44aa
    `---[410.313475ms] com.sfpay.coreplatform.account.service.impl.TransferServiceImpl:doTransfer()
        +---[0.001565ms] com.sfpay.coreplatform.account.valueobject.dto.Transfer:getPayerAccount() #100
        +---[1.994626ms] com.sfpay.coreplatform.account.service.inner.IAsyncAccountService:isAsyncAccount() #100
        +---[0.00146ms] com.sfpay.coreplatform.account.valueobject.dto.Transfer:getPayeeAccount() #104
        +---[1.772284ms] com.sfpay.coreplatform.account.service.inner.IAsyncAccountService:isAsyncAccount() #104
        +---[0.001232ms] com.sfpay.coreplatform.account.valueobject.dto.Transfer:getPayerAccount() #107
        +---[0.001044ms] com.sfpay.coreplatform.account.valueobject.dto.Transfer:getPayeeAccount() #108
        +---[401.247182ms] com.sfpay.coreplatform.account.persistence.dao.IAccountDao:selectByAccountNoAndLocked() #135
        +---[0.001748ms] com.sfpay.coreplatform.account.valueobject.dto.Transfer:getAmount() #143
        +---[0.002254ms] com.sfpay.coreplatform.account.valueobject.tmo.AccountVO:getOverdraftAmountLimit() #143
        +---[0.001531ms] com.sfpay.coreplatform.account.valueobject.tmo.AccountVO:getFreezeAmount() #143
        +---[0.004ms] com.sfpay.coreplatform.account.service.impl.TransferServiceImpl:validateBalance() #143
        +---[9.79E-4ms] com.sfpay.coreplatform.account.valueobject.tmo.AccountVO:getCashAmount() #146
        +---[0.001305ms] com.sfpay.coreplatform.account.valueobject.dto.Transfer:getAmount() #147
        +---[0.002095ms] com.sfpay.coreplatform.account.service.impl.TransferServiceImpl:calculateBalance() #147
        +---[0.001115ms] com.sfpay.coreplatform.account.valueobject.tmo.AccountVO:setCashAmount() #148
        +---[0.874777ms] com.sfpay.coreplatform.account.persistence.dao.IAccountDao:updateByPrimaryKey() #149
        +---[1.749246ms] com.sfpay.coreplatform.account.service.inner.IAccountPostingRuleService:generateTallySerials() #165
        +---[0.009297ms] org.slf4j.Logger:info() #169
        +---[0.792239ms] com.sfpay.coreplatform.account.service.inner.IDayChangeService:findDate() #171
        +---[0.001602ms] com.sfpay.coreplatform.account.valueobject.dto.DayChange:getValue() #171
        +---[min=8.93E-4ms,max=0.001594ms,total=0.002487ms,count=2] com.sfpay.coreplatform.account.valueobject.tmo.TallySerial:setTallyDate() #175
        `---[1.678851ms] com.sfpay.coreplatform.account.persistence.dao.ITallySerialDao:addTallySerialList() #178

```
解释：
-j参数可以过滤掉jdk自身的函数
com.***.Trans**ServiceImpl是接口所在的类
combineTransfer是接口的入口函数(即对应方法名）













