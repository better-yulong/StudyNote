之前，只简单的理解eclipse是基于Java语言开发的集成开发工具，了解到可通过修改eclipse.ini文件来指定eclipse启动使用的jdk、堆内存Xmx等来优化。但具体并未深入发解，而近期在研究Mybatis源码时发现在windows7平台使用jps -l 命令无法查看到主类名：
```language
C:\>jps -l
	7800
	6976
	6716 sun.tools.jps.Jps
	5920 com.zyl.base.io.SocketServer
```
查找度娘，网上主要有两类问题：
1. jps命令无法查看到 pid（Java进程ID）：jps工具原理是基于操作系统临时目录的pid文件来返回java进程信息，故没有权限读写临时目录或者pid已生成但没有权限读取均有可能出现该问题；---java启动时提供了参数(-Djava.io.tmpdir)可调整临时目录
2. jps可查看到pid，但未显示主类名（即上面示例）：暂未明确有解决办法

于是抱着试试看的态度，决定先调整临时目录，遂eclipse.ini文件添加 -Djava.io.tmpdir:D:/tmp（期望将临时目录调整至D盘），重新eclipse后jps 命令无效，而D:/tmp并没有效果。初步怀疑是配置错误，那具体该怎么配置呢？继续度娘但无有明确结果。但发现一新有用