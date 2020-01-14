尝试下载netty源码在Win10环境下导入eclipse学习，但未成功编译生成 io.netty.util.collection 对应的java类及class文件，于是切换至CentOS8，问题解决通过。修改及编译后的源码 github：https://github.com/better-yulong/netty.git 。其中common的target目录下有编译生成 io.netty.util.collection 对应的java类及class文件。

- 问题1：
非常多的工程java文件提示报错，诸如找不到如下类：
```
	import io.netty.util.collection.IntObjectHashMap;
	import io.netty.util.collection.IntObjectMap;
	import io.netty.util.collection.IntObjectMap.PrimitiveEntry;
```
经发现，整个netty源码工程还真找不到collection包及相关java类，后续了解到实际该类由 netty-common编译时基于groovy-maven-plugin动态生成。于是先行尝试编译构造mvn install common工程，但发现common工程源码报错。此处为netty的一种技巧性操作，即基于模版类KObjectHashMap.template、KObjectMap.template、KCollections.template（其实就是3个java源码文件），通过codegen.groovy脚本，结合groovy-maven-plugin插件，实现编译时通过模版类如KObjectHashMap.template 动态生成
- 问题2：common 工程报错：Plugin execution not covered by lifecycle configuration
查询研究许久，最终发现并不需处理，mvn install common工程成功。编译成功后target目录的classes目录下会生成io.netty.util.collection 包缺失各类的class文件。

- 问题3：
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











《Netty官方文档》设置开发环境（请注意：这个指南并不是用户指南，它是开发 Netty 本身的指南，而不是使用Netty 开发其他程序的指南）：https://yq.aliyun.com/articles/85340


Netty 分享之动态生成重复性的代码: https://www.jianshu.com/p/9160684f134b