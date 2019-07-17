Dubbo2.7由apache维护，因公司使用的仍是com.alibaba的Dubbo版本，故选择bubbo2.5.3 作为此次分析：github:https://github.com/apache/dubbo（选择tag:2.5.3）,doc:http://dubbo.apache.org/zh-cn/docs/dev/release.html

### 一.准备
#### 1.1 环境
- eclipse基于https://github.com/apache/dubbo，选择2.5.3为初始化版本，创建本地仓库。Eclipse中import git仓库代码并copy至workspace。
- zookeeper：windows版本（zookeeper-3.3.6.tar；已上传至StudyNote-Resource），解压缩之后运行bin目录下的zkServer即可启动zookeeper；另外也可运行同目录下zkCli进行客户端命令行模式（本地运行IP：127.0.0.1；默认端口2181)

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