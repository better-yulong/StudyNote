### 一.mvn命令行启动web应用
- 运行模式（默认tomcat端口8080
mvn tomcat:run 或者 mvn tomcat7:run  或 mvn tomcat7:run -Dmaven.tomcat.port=8081 （指定tomcat端口）
- 调试模式
mvnDebug tomcat7:run(远程调试模式启动，默认远程端口8000)
- 另可通过-X参数（mvn -X tomcat:run ）开启debug日志输出，对于分析定位问题相当有用

### 一.mvn命令分析工程依赖
命令无需死记，命令行输入 mvn dependency: 会运行分析(注意后面的冒号)，但会报错（因mvn -h 命令并不会输出需要的命令提示信息）：
```language
[ERROR] Could not find goal '' in plugin org.apache.maven.plugins:maven-dependency-plugin:2.1 among available goals unpack-dependencies, g
o-offline, copy-dependencies, analyze-dep-mgt, list, purge-local-repository, help, get, build-classpath, sources, analyze-report, analyze,
 tree, unpack, resolve, copy, analyze-only, resolve-plugins 
```
通过报错即可知其支持的参数，在工程根目录（与pom.xml同级）运行 mvn dependency:tree ：
```language
[INFO] com.zyl.demo.web:spring3-analysis:war:0.0.1-SNAPSHOT
[INFO] +- org.springframework:spring-core:jar:3.1.0.RELEASE:compile
[INFO] |  +- org.springframework:spring-asm:jar:3.1.0.RELEASE:compile
[INFO] |  \- commons-logging:commons-logging:jar:1.1.1:compile
[INFO] \- org.springframework:spring-web:jar:3.1.0.RELEASE:compile
[INFO]    +- aopalliance:aopalliance:jar:1.0:compile
[INFO]    +- org.springframework:spring-beans:jar:3.1.0.RELEASE:compile
[INFO]    \- org.springframework:spring-context:jar:3.1.0.RELEASE:compile
[INFO]       +- org.springframework:spring-aop:jar:3.1.0.RELEASE:compile
[INFO]       \- org.springframework:spring-expression:jar:3.1.0.RELEASE:compile
```
mvn dependency:tree 可用于分析当前应用的依赖语法树（类似于Eclipse中Dependency Hierarchy的语法树层级显示；但悲催的是Eclipse该依赖分析无法复制）层级信息；其他如mvn dependency:tree

