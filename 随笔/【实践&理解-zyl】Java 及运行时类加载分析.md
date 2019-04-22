1. java -verbose  (java -verbose:gc或java -verbose:class) 用于获取JVM详细
- 而执行java -verbose是启动一个JVM虚拟机，而默认直接执行java -verbose:class 即为启动一个JVM虚拟机并打印类加载信息，所以打印出来只会有jre依赖的所有class类加载信息。
- 另外java -verbose 命令并不支持指定java 进程PID来获取已运行时的JVM类加载信息，该命令为启动一个新的JVM虚拟机。
2. 如若要查看基于tomcat的应用运行时class信息，则在tomcat启动脚本中添加 -verbose:class参数即可将class加载信息输出到catalina.out文件，如： [Loaded org.mybatis.spring.SqlSessionUtils$SqlSessionSynchronization from file:/tomcat/tomcat/sypay-passport/webapps/***/WEB-INF/lib/mybatis-spring-1.0.2.jar]