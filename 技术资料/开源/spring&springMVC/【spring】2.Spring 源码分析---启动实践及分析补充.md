### 一. 自定义ServletContextListener
#### 1.1 自定义framework工程framework-aoe-web工程（类似spirng-web），新建ContextLoaderListener继承org.springframework.web.context.ContextLoaderListener，源码
```language
 package com.framework.aoe.web;

import javax.servlet.ServletContextEvent;

public class ContextLoaderListener extends org.springframework.web.context.ContextLoaderListener{
	
	public void contextInitialized(ServletContextEvent event) {
		System.out.println("==> aoe ContextLoaderListener contextInitialized start。。。");
		super.contextInitialized(event);
		System.out.println("==> aoe ContextLoaderListener contextInitialized end。。。");
	}
}
```
```language
  //framework-aoe-web pom.xml
  <dependencies>
		<dependency>
			<groupId>org.springframework</groupId>
			<artifactId>spring-web</artifactId>
			<version>3.1.0.RELEASE</version>
		</dependency>
  </dependencies>
```

#### 1.2示例工程spring3-analysis 依赖jar引入及配置 
```language
        //spring3-analysis pom.xml
	<dependencies>
		<dependency>
			<groupId>org.springframework</groupId>
			<artifactId>spring-core</artifactId>
			<version>3.1.0.RELEASE</version>
		</dependency>
		<dependency>
			<groupId>org.springframework</groupId>
			<artifactId>spring-web</artifactId>
			<version>3.1.0.RELEASE</version>
		</dependency>
		<dependency>
			<groupId>com.aoe.framework</groupId>
		    <artifactId>framework-aoe-web</artifactId>
		    <version>0.0.1-SNAPSHOT</version>
		</dependency>
		
	</dependencies>
```
```language
  //web.xml
  <context-param>
  	<param-name>contextConfigLocation</param-name>
  	<param-value>classpath:applicationContext.xml</param-value>
  </context-param>
  <listener>
    	<listener-class>com.framework.aoe.web.ContextLoaderListener</listener-class>
  </listener>
```

#### 1.3示例工程spring3-analysis pom.xml配置优化
之前在基于spring、spirngMVC搭建新工程时会比较困惑，究竟需要配置哪几个jar依赖才可以?
Eclipse中Dependency Hierarchy的语法树层级显示jar依赖及传递依赖关系或者spring3-analysis工程根目录（与pom.xml同级）运行 mvn dependency:tree 分析，类似如下好只需配置spirng-web即可自动依赖其他所需jar：
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

根据如上方法，分析spring3-analysis工程依赖，只需按配置如下一个依赖即可通过传递依赖实现所有依赖jar。
```language
        //spring3-analysis pom.xml
	<dependencies>
		<dependency>
			<groupId>com.aoe.framework</groupId>
		    <artifactId>framework-aoe-web</artifactId>
		    <version>0.0.1-SNAPSHOT</version>
		</dependency>
		
	</dependencies>
```
#### 1.4 示例bean及xml配置
```package com.aoe.demo;

public class BeanExample {
	private String id;
}
```
















