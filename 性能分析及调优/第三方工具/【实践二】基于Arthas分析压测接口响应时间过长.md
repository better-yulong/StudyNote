近期基于业务需要，正好在对交易进行性能测试，其中交易研发同学根据服务调用日志反馈记账接口偶尔处理时长超过200ms，于是此基于此，决定使用Arthas进行定位。

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
输入thread会显示所有线程的状态信息

输入thread -n 3会显示当前最忙的3个线程，可以用来排查线程CPU消耗

输入thread -b 会显示当前处于BLOCKED状态的线程，可以排查线程锁的问题




















