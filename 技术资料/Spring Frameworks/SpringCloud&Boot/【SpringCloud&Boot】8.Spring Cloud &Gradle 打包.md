参考：https://blog.csdn.net/stupid56862/article/details/86363292

### 三级标题jar 包 和 war 包
- 普通 jar 包 : 会将源码编译后以工具包（即将class打成jar包）的形式对外提供，此时，你的 jar 包不一定要是可执行的，只要能通过编译，可以被别的项目以 import 的方式调用。
- 可执行 jar 包 : 能通过 java -jar 的命令运行。
- 普通 war 包 : war 是一个 web 模块，其中包括 WEB-INF，是可以直接运行的 WEB 模块。做好一个 web 应用后，打成包部署到容器中。
- 可执行 war 包 : 普通 war 包 + 内嵌容器 。

### 实践
#### 编译打可执行jar包
项目源码根目录执行 gradle build ，报错（找不到依赖的jar包，麻屁，竟然还会出现错误提示中文汉字重复，先忽略）：
```
D:\workspace\work-git\aoe\src\main\java\com\aoe\Application.java:3: 错错误误: 程程序序包包org.mybatis.spring.annotation不存在
import org.mybatis.spring.annotation.MapperScan;
........
```
