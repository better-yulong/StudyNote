MongoDB客户端 Studio 3T破解方法（方法一注册表方式验证OK): https://www.jianshu.com/p/7257f15e2620


问题1：
非常多的工程java文件提示报错，诸如找不到如下类：
```
	import io.netty.util.collection.IntObjectHashMap;
	import io.netty.util.collection.IntObjectMap;
	import io.netty.util.collection.IntObjectMap.PrimitiveEntry;
```
经发现，整个netty源码工程还真找不到collection包及相关java类，后续了解到实际该类由 netty-common编译时基于groovy-maven-plugin动态生成。于是先行尝试编译构造mvn install common工程，但common工程源码报错
- 问题2：
```  
<plugin>
        <groupId>com.github.veithen.alta</groupId>
        <artifactId>alta-maven-plugin</artifactId>
        <version>0.6.2</version>
        <executions>
          <execution> <!--此处报错：unpack should be executed after packaging: see MDEP-98.-->
            <goals>
              <goal>generate-test-resources</goal>
            </goals>
            <configuration>
              <name>%bundle.symbolicName%.link</name>
              <value>%url%</value>
              <dependencySet>
                <scope>test</scope>
              </dependencySet>
            </configuration>
          </execution>
        </executions>
      </plugin>
      <plugin>

```
解决办法：查询了许久资料，结果是在execution 之后输入一个空格，保存错误即自动消失。见Dubbo报错处理：https://my.oschina.net/z201/blog/745405?utm_source=debugrun&utm_medium=referral
