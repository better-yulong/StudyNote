### 一.mvn命令行启动web应用
- 运行模式（默认tomcat端口8080
mvn tomcat:run 或者 mvn tomcat7:run  或 mvn tomcat7:run -Dmaven.tomcat.port=8081 （指定tomcat端口）
- 调试模式
mvnDebug tomcat7:run(远程调试模式启动，默认远程端口8000)
- 另可通过-X参数（mvn -X tomcat:run ）开启debug日志输出，对于分析定位问题相当有用

### 一.mvn命令分析工程依赖
命令无需死记，命令行输入 mvn dependency 会运行分析，但会报错（因mvn -h 命令并不会输出需要的命令提示信息）：
```language
plugin-group-id>:<plugin-artifact-id>[:<plugin-version>]:<goal>. Available lifecycle phases are: validate, initialize, generate-sources, p
rocess-sources, generate-resources, process-resources, compile, process-classes, generate-test-sources, process-test-sources, generate-tes
t-resources, process-test-resources, test-compile, process-test-classes, test, prepare-package, package, pre-integration-test, integration
-test, post-integration-test, verify, install, deploy, pre-clean, clean, post-clean, pre-site, site, post-site, site-deploy. -> [Help 1]
```
通过报错即可知其支持的参数，t

