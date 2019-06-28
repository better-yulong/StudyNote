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
```
  package com.aoe.demo;

  public class BeanExample {
	private String id;
  }
```
```language
  <bean id="beanExample" name="beanExample" class="com.aoe.demo.BeanExample"></bean>
```
运行OK，tomat以debug模式启动日志：
```language
019-6-27 17:28:49 org.apache.catalina.core.StandardEngine startInternal
信息: Starting Servlet Engine: Apache Tomcat/7.0.14
==> aoe ContextLoaderListener contextInitialized start。。。
2019-6-27 17:28:51 org.apache.catalina.core.ApplicationContext log
信息: Initializing Spring root WebApplicationContext
2019-6-27 17:28:51 org.springframework.web.context.ContextLoader initWebApplicationContext
信息: Root WebApplicationContext: initialization started
2019-6-27 17:28:51 org.springframework.context.support.AbstractApplicationContext prepareRefresh
信息: Refreshing Root WebApplicationContext: startup date [Thu Jun 27 17:28:51 CST 2019]; root of context hierarchy
2019-6-27 17:28:51 org.springframework.beans.factory.xml.XmlBeanDefinitionReader loadBeanDefinitions
信息: Loading XML bean definitions from class path resource [applicationContext.xml]
2019-6-27 17:28:52 org.springframework.beans.factory.support.DefaultListableBeanFactory preInstantiateSingletons
信息: Pre-instantiating singletons in org.springframework.beans.factory.support.DefaultListableBeanFactory@10cc9b4: defining beans [beanExample]; root of factory hierarchy
2019-6-27 17:28:52 org.springframework.web.context.ContextLoader initWebApplicationContext
信息: Root WebApplicationContext: initialization completed in 1263 ms
==> aoe ContextLoaderListener contextInitialized end。。。
2019-6-27 17:28:52 org.apache.coyote.AbstractProtocolHandler start
信息: Starting ProtocolHandler ["http-bio-8080"]
2019-6-27 17:28:52 org.apache.coyote.AbstractProtocolHandler start
信息: Starting ProtocolHandler ["ajp-bio-8009"]
2019-6-27 17:28:52 org.apache.catalina.startup.Catalina start
信息: Server startup in 3018 ms
```

### 二. 基于注解注入Bean
目前仍未引入SpirngMVC，主要用于分析spring bean的初始化原理，为便于分析，结合之前分析源码有关于其他扩展方法（init-method方、afterPropertiesSet、BeanPostProcessor），故同步实践相关方法
#### 1.init-method方、afterPropertiesSet、BeanPostProcessor介绍
##### 1.1 init-method方法
init-method方法，初始化bean的时候执行，可以针对某个具体的bean进行配置。init-method需要在applicationContext.xml配置文档中bean的定义里头写明。例如：<bean id="TestBean" class="nju.software.xkxt.util.TestBean" init-method="init"></bean>
这样，当TestBean在初始化的时候会执行TestBean中定义的init方法。
##### 1.2 afterPropertiesSet方法
afterPropertiesSet方法，初始化bean的时候执行，可以针对某个具体的bean进行配置。afterPropertiesSet 必须实现 InitializingBean接口。实现 InitializingBean接口必须实现afterPropertiesSet方法。
##### 1.3 BeanPostProcessor
BeanPostProcessor针对所有Spring上下文中所有的bean，可以在配置文档applicationContext.xml中配置一个BeanPostProcessor，然后对所有的bean进行一个初始化之前和之后的代理。BeanPostProcessor接口中有两个方法： postProcessBeforeInitialization和postProcessAfterInitialization。 postProcessBeforeInitialization方法在bean初始化之前执行， postProcessAfterInitialization方法在bean初始化之后执行。
- 总之，afterPropertiesSet 和init-method之间的执行顺序是afterPropertiesSet 先执行，init-method 后执行。从BeanPostProcessor的作用，可以看出最先执行的是postProcessBeforeInitialization，然后是afterPropertiesSet，然后是init-method，然后是postProcessAfterInitialization。

#### 2 基于注解方式自动注入分析
之前基于xml中通过bean标签注入，但后续实际更多的是基于xml配置扫描、java源文件使用注解标签方式注入
##### 2.1 示例类AnnotationBeanExample（暂未配置注解扫描）
```language
package com.aoe.demo;

import org.springframework.beans.factory.InitializingBean;
import org.springframework.stereotype.Component;

@Component
public class AnnotationBeanExample implements InitializingBean{
	private String name ;

    public void init() {  
        System.out.println("init-method is called");  
        System.out.println("******************************");  
    }  
    
	public String getName() {
		return name;
	}

	public void setName(String name) {
		this.name = name;
	}

	public void afterPropertiesSet() throws Exception {
		System.out.println("afterPropertiesSet has been created");
	}
}
```
因未配置注解，运行时并未自动完成该bean的注入，如若实例化成功，会有如上运行日志片段：
> 信息: Pre-instantiating singletons in org.springframework.beans.factory.support.DefaultListableBeanFactory@10cc9b4: defining beans [beanExample]; root of factory hierarchy
- 通过注解方式注入Bean单单给除给类加上注解外，还需在xml中配置以开启注解扫描，具体怎么配置呢？之前更多是基于度娘或者Spring官方文档，但并未了解过。注解实际也对应一个类，那我们就看看源码（注解也很关键）：
```language
/**
 * Indicates that an annotated class is a "component".
 * Such classes are considered as candidates for auto-detection
 * when using annotation-based configuration and classpath scanning.
 *
 * <p>Other class-level annotations may be considered as identifying
 * a component as well, typically a special kind of component:
 * e.g. the {@link Repository @Repository} annotation or AspectJ's
 * {@link org.aspectj.lang.annotation.Aspect @Aspect} annotation.
 *
 * @author Mark Fisher
 * @since 2.5
 * @see Repository
 * @see Service
 * @see Controller
 * @see org.springframework.context.annotation.ClassPathBeanDefinitionScanner
 */
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Component {

	/**
	 * The value may indicate a suggestion for a logical component name,
	 * to be turned into a Spring bean in case of an autodetected component.
	 * @return the suggested component name, if any
	 */
	String value() default "";

}
```
- Component用于标识一个类为组件但过于笼统，而Controller、Service、Repository则是基于业务特性分层，前期与Component相似，但后期会被添加独有的特性（基于Domain-Driven Design；在同一包里面，注释方面由有差异，会说明可与这些注解配合使用的其他注解）。其中有一句：when using annotation-based configuration and classpath scanning，即说明若需使用注解方式注入bean则应在xml中注解配置和classpath扫描。
##### 2.2 分析注解配置
根据上面的注释 ClassPathBeanDefinitionScanner ，于是首首先想到的是在applicationContex.xml配置：
> <bean class="org.springframework.context.annotation.ClassPathBeanDefinitionScanner"></bean>
但验证之后发现，启动果断报错，提示没有匹配的构造方法（即无参构造方法），查看源码确实没有，而参数中至今包含一个BeanDefinitionRegistry参数，感觉不对。之后算是没有思路，于是决定看看带Scan关键字的类有哪些，无竟间发现ComponentScan注解,通过查看源码及注释：
```language
 Configures component scanning directives for use with @{@link Configuration} classes.
 * Provides support parallel with Spring XML's {@code <context:component-scan>} element.
```
通过注释可发现，配置组件扫描可使用@Configuration 注解，也可通过Spring XML配置：<context:component-scan>，那么可知即在applicationContext.xml配置<context:component-scan>；便根据经验，因xml文件默认使用beans作为根标签，默认










