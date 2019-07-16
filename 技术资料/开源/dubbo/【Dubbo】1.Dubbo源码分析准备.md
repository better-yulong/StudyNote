Dubbo2.7由apache维护，因公司使用的仍是com.alibaba的Dubbo版本，故选择bubbo2.5.3 作为此次分析：github:https://github.com/apache/dubbo（选择tag:2.5.3）,doc:http://dubbo.apache.org/zh-cn/docs/dev/release.html

### 一.源码下载
- eclipse基于https://github.com/apache/dubbo，选择2.5.x为初始化版本，创建本地仓库。Eclipse中import git仓库代码并copy至workspace。
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
rpc-skeleton新


Configuration problem: Unable to locate Spring NamespaceHandler for XML schema namespace