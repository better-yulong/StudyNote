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
之后需在rpc-client、rpc-server 应用的pom.xml中配置对rpc-skeleton的依赖.
##### 1.1.2 rpc-server 服务
HessianExampleService1服务实现类
```language
package com.aoe.demo.rpc.hessian;

import java.util.ArrayList;
import java.util.List;

public class HessianExampleService1 implements HessianExampleInterf1 {
	
	/* (non-Javadoc)
	 * @see com.aoe.demo.rpc.hessian.HessianExampleInterf1#getServiceNames(java.util.List)
	 */
	public List getServiceNames(List paramList){
		System.out.println("param0:" + paramList.get(0));
		List serviceNames = new ArrayList<String>();
		serviceNames.add("service hessianExampleService1");
		return serviceNames ;
	}

}

```
spring-servlet.xml配置
```language
<bean name="hessianExampleService1"  class="com.aoe.demo.rpc.hessian.HessianExampleService1"></bean>
	
	<bean name="/hessianExampleService1" class="org.springframework.remoting.caucho.HessianServiceExporter">
	    <property name="service" ref="hessianExampleService1"/>
	    <property name="serviceInterface" value="com.aoe.demo.rpc.hessian.HessianExampleInterf1"/>
	</bean>
```
pom.xml中配置对hessian的jar依赖
```language
     <dependency>
	<groupId>com.caucho</groupId>
	<artifactId>com.springsource.com.caucho</artifactId>
	<version>3.2.1</version>
    </dependency>  
```
##### 1.1.3 rpc-client 客户端
外部请求Controller（用于发起hessian客户端请求）
```language
package com.aoe.demo.rpc.controller;

import java.util.ArrayList;
import java.util.List;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.ResponseBody;

import com.aoe.demo.rpc.hessian.HessianExampleInterf1;

@Controller
public class EntryController {
	
	@Autowired
	private HessianExampleInterf1 hessianExampleService;
	
	@RequestMapping("/entry")
	@ResponseBody
	public Object entry(){
		List<String> params =  new ArrayList<String>();
		params.add("parm");
		System.out.println("result0:" + hessianExampleService.getServiceNames(params).get(0));
		return "entry";
	}

}
```
spring-servlet.xml配置
```language
 <bean id="accountService" class="org.springframework.remoting.caucho.HessianProxyFactoryBean">
    	<property name="serviceUrl" value="http://localhost:8080/rpc-server/rpc/hessianExampleService1"/>
    	<property name="serviceInterface" value="com.aoe.demo.rpc.hessian.HessianExampleInterf1"/>
	</bean> 
```
同rpc-server在其pom.xml中配置对hessian的jar依赖
##### 1.1.4 验证
访问http://localhost:8080/rpc-client/rpc/entry，如预期完成hessian调用验证。

### 二. hessian 服务端（HessianServiceExporter）
HessianServiceExporter源码
```language
public class HessianServiceExporter extends HessianExporter implements HttpRequestHandler {

	/**
	 * Processes the incoming Hessian request and creates a Hessian response.
	 */
	public void handleRequest(HttpServletRequest request, HttpServletResponse response)
			throws ServletException, IOException {

		if (!"POST".equals(request.getMethod())) {
			throw new HttpRequestMethodNotSupportedException(request.getMethod(),
					new String[] {"POST"}, "HessianServiceExporter only supports POST requests");
		}

		response.setContentType(CONTENT_TYPE_HESSIAN);
		try {
		  invoke(request.getInputStream(), response.getOutputStream());
		}
		catch (Throwable ex) {
		  throw new NestedServletException("Hessian skeleton invocation failed", ex);
		}
	}

}
```
HessianServiceExporter 类本身的源码仅包含handleRequest方法；分析之前先回顾下之前分析SpringMVC关于获取Hanlder、HanlderAdapter及调用原理:
1. hanlder注册
SpringMVC容器启动初始化上下文时，会检测所有的Hanlder(注解或者beanName）；而当前示例hessian服务发布时beanName为"/hessianExampleService1"，符合该规则即在rpc-server启动时，会调用this.handlerMap.put(urlPath, resolvedHandler) 完成urlPath("/hessianExampleService1")与resolvedHandler（HessianServiceExporter实例）的映射；
2. 根据rpc-client的配置客户端发起hessian服务调用时（<property name="serviceUrl" value="http://localhost:8080/rpc-server/rpc/hessianExampleService1"/>），获取到步骤1注册时对应的Hanlder（HessianServiceExporter实例）
3. 



