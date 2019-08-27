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
3. 运行直接成功（因为服务器并不像我本地有多个JDK版本，故无需