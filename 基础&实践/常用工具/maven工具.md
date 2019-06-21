### 一.mvn命令行启动web应用
运行模式（默认omcat端口8080
mvn tomcat:run 或者 mvn tomcat7:run  或 mvn tomcat7:run -Dmaven.tomcat.port=8081 （指定tomcat端口）
或 mvnDebug tomcat7:run(远程调试模式启动)
其中可 mvn -X tomcat7:run 开启debug日志输出