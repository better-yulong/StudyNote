### 一.mvn命令行启动web应用
- 运行模式（默认tomcat端口8080
mvn tomcat:run 或者 mvn tomcat7:run  或 mvn tomcat7:run -Dmaven.tomcat.port=8081 （指定tomcat端口）
- 调试模式
mvnDebug tomcat7:run(远程调试模式启动，默认远程端口8000)
- 另可通过-X参数（mvn -X tomcat:run ）开启debug日志输出，对于分析定位问题相当有用

### 一.mvn命令分析工程依赖
命令无需合约
