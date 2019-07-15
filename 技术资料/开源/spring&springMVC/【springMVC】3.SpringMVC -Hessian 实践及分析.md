结合之前的分析，根据spring 文档（章节：3 Using Hessian or Burlap to remotely call services via HTTP），搭建hessian示例工程并运行。

### 一. demo示例编写
#### 1.1 创建示例工程
创建示例web 工程rpc-client、rpc-server,及java工程 rpc-skeleton，分别对于hessian客户端、hessian服务端、rpc骨架包
##### 1.1.1 rpc-skeleton 接口
```language
	package com.aoe.demo.rpc.hessian;

	import java.util.List;

	public interface HessianExampleInterf1 {

		List getServiceNames(List paramList);

	}
```
之后需在rpc-client、rpc-server 应用的pom.xml中配置对rpc-skeleton的依赖