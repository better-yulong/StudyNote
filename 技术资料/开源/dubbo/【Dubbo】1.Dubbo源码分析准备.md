Dubbo2.7由apache维护，因公司使用的仍是com.alibaba的Dubbo版本，故选择bubbo2.5.3 作为此次分析：github:https://github.com/apache/dubbo（选择tag:2.5.3）,doc:http://dubbo.apache.org/zh-cn/docs/dev/release.html

### 一.准备
#### 1.1 环境准备
#### 1.1.1 dubbo源码下载
eclipse基于https://github.com/apache/dubbo，选择2.5.3为初始化版本，创建本地仓库。Eclipse中import git仓库代码并copy至workspace。
#### 1.1.2 dubbo intalll至本地仓库
1.  dubbo解压缩后/目录运行mvn clean install -Dmaven.test.skip ，报错提示：opensesame相关报错。https://github.com/alibaba/opensesame 下载opensesame 的zip包（该包为alibaba开源项目的顶级父类项目），解压opensesame的zip包在其/目录运行 mvn clean install -Dmaven.test.skip ，相对比较顺利：opensesame成功编译并被安装至本地；
2. 再次切到dubbo /目录运行mvn clean install -Dmaven.test.skip，但又发现缺少hessian-lite依赖，于是首各种资料（尝试找源码编译、找可用的maven仓库），花了不少时间根据资料（https://github.com/apache/dubbo/issues/21）在mavne的 settings.xml文件添加mirror源（可同时解决opensesame及hessian-lite问题；即第1步的手动编译可忽略）：
```language
    <mirror>  
        <id>cuisongliu</id>  
        <mirrorOf>*</mirrorOf>    
        <name>cubisongliu</name>  
        <url>http://maven.cuisongliu.com/content/groups/public</url>  
    </mirror> 
```
3. 步骤2完成后，再次切到dubbo /目录运行mvn clean install -Dmaven.test.skip，此时运行发现dubbo-common包中的JSONTest.java测试方法报错无法编译通过。同样百度各种尝试如手动编译（却发现无源码）或更换fastjson版本号，各种不爽；无奈之下，因其是Test方法那粗暴点注释该方法，重新编译还真OK。于是乎后面同样的方式处理ClientReconnectTest报错。
4. 步骤3运行一段时间之后再次中断，系统报错原因：PermGen space 溢出；该问题看似比较简单，但在我本地还是花了点时间，常用方法即是修改mvn.bat（windows平台）的MAVEN_OPTS参数（set MAVEN_OPTS=-Xms512m -Xmx2048m -XX:MaxPermSize=256m）。然额，修改后使用命令mvn -v查看提示Error，而仅修改了该参数；之后研究其他方式配置或者调整jdk版本，但并没生效，经过折腾及尝试，mvn.bat调整为：
```language
set MAVEN_OPTS=-XX:MaxPermSize=256m  //同时配置Xms、Xmx就不行，仅配置这一个确实可生效，先解决暂不定位原因
set JAVA_HOME=D:\work\java\JDK6 //考虑到jdk兼容，采用JDK6编译，因为demo工程基本也是使用的JRE6
```
4. 同步骤3解决NettyClientTest编译报错及其他类似报错，耗时13分钟终于OK。
注：alibaba dubbo官方文档（区别于apache dubbo）：https://github.com/alibaba/dubbo-doc-static

#### 1.1.2 dubbo 源码导入至eclipse


#### 1.1.3  zookeeper
windows版本（zookeeper-3.3.6.tar；已上传至StudyNote-Resource），解压缩之后运行bin目录下的zkServer即可启动zookeeper；另外也可运行同目录下zkCli进行客户端命令行模式（本地运行IP：127.0.0.1；默认端口2181)

### 二.dubbo配置及示例验证
dubbo系统示例工程沿用Spring&SpirngMVC分析时hessian示例使用的三个工程:rpc-server、rpc-client、rpc-skeleton.
#### 2.1 配置方式
根据dubbo的官方文档（http://dubbo.apache.org/zh-cn/docs/user/configuration/xml.html）， dubbo支持多种配置：XML配置、API配置、注解配置，当前示例先选择xml配置方式
##### 2.1.1 dubbo 服务接口（rpc-skeleton)
```language
package com.aoe.demo.rpc.dubbo;

import java.util.List;

public interface DubboExampleInterf1 {
	
	public List serviceProvider(List params);

}
```
##### 2.1.2 dubbo服务实现(rpc-server)
######  2.1.2.1 DubboExampleService1服务实现
```language
public class DubboExampleService1 implements DubboExampleInterf1 {

	public List serviceProvider(List params) {
		System.out.println("param0:" + params.get(0));
		List serviceNames = new ArrayList<String>();
		serviceNames.add("DubboExampleService1");
		return serviceNames ;
	}
}
```
######  2.1.2.2 xml配置
参考官网添加xml头(xmlns、xsd），之后基于dubbo 标签完成bean及dubbo相关配置，然而启动后始终报错：
org.springframework.beans.factory.parsing.BeanDefinitionParsingException: Configuration problem: Unable to locate Spring NamespaceHandler for XML schema namespace [http://dubbo.apache.org/schema/dubbo]。
- 最初基于官网配置，并未意识到未引入dubbo 的jar包；
开始没太明白，初步怀疑是xml头文件修改时出现错误，然而多次确认发现并无明显错误；于是基于之前自定义NamespaceHanlder的经验，于是全文搜索"dubbo.apache.org/schema/dubbo"确认对应hanlder配置发现并没有找到（步骤一下载的dubbo源码并无匹配项）。本来还有些许疑虑,于是换个思路分析 *.halder文件中关于Namespace配置，至此才恍然大悟，官网基于apache dubbo配置，而当前dubbo版本为alibaba版本：
```language
http\://code.alibabatech.com/schema/dubbo=com.alibaba.dubbo.config.spring.schema.DubboNamespaceHandler
```


Configuration problem: Unable to locate Spring NamespaceHandler for XML schema namespace