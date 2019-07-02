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
- 通过注解方式注入Bean单单给除给类加上注解外，还需在xml中配置以开启注解扫描，具体怎么配置呢？之前更多是基于度娘或者Spring官方文档，但并未了解过。注解实际也对应一个类，那我们就看看源码（注释也很关键）：
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
- Component用于标识一个类为组件但过于笼统，而Controller、Service、Repository则是基于业务特性分层，前期与Component相似，但后期会被添加独有的特性（基于Domain-Driven Design；在同一包里面，注释方面有差异，会说明可与这些注解配合使用的其他注解）。其中有一句：when using annotation-based configuration and classpath scanning，即说明若需使用注解方式注入bean则应在xml中注解配置和classpath扫描。
##### 2.2 分析注解配置
根据上面的注释 ClassPathBeanDefinitionScanner ，于是首先想到的是在applicationContex.xml配置：
 ```
<bean class="org.springframework.context.annotation.ClassPathBeanDefinitionScanner"></bean>
```
但验证之后发现，启动果断报错，提示没有匹配的构造方法（即无参构造方法），查看源码确实没有，而参数中至今包含一个BeanDefinitionRegistry参数，感觉不对。之后算是没有思路，于是决定看看带Scan关键字的类有哪些，无竟间发现ComponentScan注解,通过查看源码及注释：
```language
 Configures component scanning directives for use with @{@link Configuration} classes.
 * Provides support parallel with Spring XML's {@code <context:component-scan>} element.
```
通过注释可发现，配置组件扫描可使用@Configuration 注解，也可通过Spring XML配置：<context:component-scan>，那么可知即在applicationContext.xml配置<context:component-scan>；便根据经验，因xml文件默认使用beans作为根标签，默认命名空间是支持bean标签；而需使用其他命名空间如context的标签则需在xml文件头引入新的命名空间及xsd文件。一开始除了百度或官网也不确定该如何可准确配置，于是呼尝试继承在源码中查找是否要可用的xml文参考，果不其然找到了componentScan.xml，于是修改applicationContext.xml配置为：
```language
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
		xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
		xmlns:context="http://www.springframework.org/schema/context"
		xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-2.5.xsd
				http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context-2.5.xsd">

	<bean id="beanExample" name="beanExample" class="com.aoe.demo.BeanExample"></bean>
	<context:component-scan base-package="com.aoe"/>
</beans>
```
再次运行bean：beanExample,annotationBeanExample注入成功，日志：
```language
==> aoe ContextLoaderListener contextInitialized start。。。
2019-6-28 14:21:27 org.apache.catalina.core.ApplicationContext log
信息: Initializing Spring root WebApplicationContext
2019-6-28 14:21:27 org.springframework.web.context.ContextLoader initWebApplicationContext
信息: Root WebApplicationContext: initialization started
2019-6-28 14:21:27 org.springframework.context.support.AbstractApplicationContext prepareRefresh
信息: Refreshing Root WebApplicationContext: startup date [Fri Jun 28 14:21:27 CST 2019]; root of context hierarchy
2019-6-28 14:21:27 org.springframework.beans.factory.xml.XmlBeanDefinitionReader loadBeanDefinitions
信息: Loading XML bean definitions from class path resource [applicationContext.xml]
2019-6-28 14:21:44 org.springframework.beans.factory.support.DefaultListableBeanFactory preInstantiateSingletons
信息: Pre-instantiating singletons in org.springframework.beans.factory.support.DefaultListableBeanFactory@966905: defining beans [beanExample,annotationBeanExample,org.springframework.context.annotation.internalConfigurationAnnotationProcessor,org.springframework.context.annotation.internalAutowiredAnnotationProcessor,org.springframework.context.annotation.internalRequiredAnnotationProcessor,org.springframework.context.annotation.internalCommonAnnotationProcessor,org.springframework.context.annotation.ConfigurationClassPostProcessor$ImportAwareBeanPostProcessor#0]; root of factory hierarchy
afterPropertiesSet has been created
2019-6-28 14:21:44 org.springframework.web.context.ContextLoader initWebApplicationContext
信息: Root WebApplicationContext: initialization completed in 17536 ms
==> aoe ContextLoaderListener contextInitialized end。。。
2019-6-28 14:21:44 org.apache.coyote.AbstractProtocolHandler start
信息: Starting ProtocolHandler ["http-bio-8080"]
2019-6-28 14:21:45 org.apache.coyote.AbstractProtocolHandler start
信息: Starting ProtocolHandler ["ajp-bio-8009"]
2019-6-28 14:21:45 org.apache.catalina.startup.Catalina start
信息: Server startup in 18622 ms
```
从日志来看afterPropertiesSet运行但init方法并未执行，其实原因很简单，实现implements InitializingBean的bean由spring容器化实例之后会自动调用afterPropertiesSet方法；而init虽然名为init仅只是一个普通的方法，如在xml中可通过bean的init-method来指定在实例初始化之前执行，但注解方式该如何让其执行呢？暂时并没有什么好的思路：1.既然知道是在beean实例化时调用那么可再次梳理下之前分析的Spring bean注入源码，调试确认；2.可通过之前了解的init-method标签解析梳理调用逻辑。但实际分析了并未找到对应的逻辑，初步认为注解实现方式与xml标签实现有差异，借助于度娘:https://blog.csdn.net/glory1234work2115/article/details/51815911；https://blog.csdn.net/topwqp/article/details/8681467
###### 2.2.1 bean初步化、销毁方法多种实现方式
1. @PostConstruct 和 @PreDestroy 方法 或 实现InitializingBean和 DisposableBean接口
```language
import javax.annotation.PostConstruct;
import javax.annotation.PreDestroy;

import org.springframework.beans.factory.DisposableBean;
import org.springframework.beans.factory.InitializingBean;
import org.springframework.stereotype.Component;

@Component
public class AnnotationBeanExample implements InitializingBean,DisposableBean{
	private String name ;

	public String getName() {
		return name;
	}

	public void setName(String name) {
		this.name = name;
	}

	//实现InitializingBean接口
	public void afterPropertiesSet() throws Exception {
		System.out.println("AnnotationBeanExample afterPropertiesSet has been created");
	}

	//实现DisposableBean接口
	public void destroy() throws Exception {
		System.out.println("AnnotationBeanExample destroy  running...");
	}
	
	@PostConstruct
        public void init() {  
           System.out.println("AnnotationBeanExample init-method is called");  
        }  
	
	@PreDestroy 
	public void preDestroy() throws Exception {
		System.out.println("AnnotationBeanExample preDestroy running...");
	}
	
}
```
2. xml中定义init-method 和  destory-method方法
```language
	<bean id="beanExample" name="beanExample" class="com.aoe.demo.BeanExample" init-method="init" destroy-method="destroy"></bean>
```
```language
public class BeanExample {
	
	public void destroy() throws Exception {
		System.out.println("BeanExample destroy  running...");
	}
	
    public void init() {  
        System.out.println("BeanExample init-method is called");  
     }  
	
}
```
三种方式实现有差异，各方法执行的顺序不同，可同时支持多种；示例运行日志为：
```language
==> aoe ContextLoaderListener contextInitialized start。。。
2019-6-28 18:24:09 org.apache.catalina.core.ApplicationContext log
信息: Initializing Spring root WebApplicationContext
2019-6-28 18:24:09 org.springframework.web.context.ContextLoader initWebApplicationContext
信息: Root WebApplicationContext: initialization started
2019-6-28 18:24:10 org.springframework.context.support.AbstractApplicationContext prepareRefresh
信息: Refreshing Root WebApplicationContext: startup date [Fri Jun 28 18:24:10 CST 2019]; root of context hierarchy
2019-6-28 18:24:10 org.springframework.beans.factory.xml.XmlBeanDefinitionReader loadBeanDefinitions
信息: Loading XML bean definitions from class path resource [applicationContext.xml]
2019-6-28 18:24:18 org.springframework.beans.factory.support.DefaultListableBeanFactory preInstantiateSingletons
信息: Pre-instantiating singletons in org.springframework.beans.factory.support.DefaultListableBeanFactory@270fc4: defining beans [beanExample,annotationBeanExample,org.springframework.context.annotation.internalConfigurationAnnotationProcessor,org.springframework.context.annotation.internalAutowiredAnnotationProcessor,org.springframework.context.annotation.internalRequiredAnnotationProcessor,org.springframework.context.annotation.internalCommonAnnotationProcessor,org.springframework.context.annotation.ConfigurationClassPostProcessor$ImportAwareBeanPostProcessor#0]; root of factory hierarchy
BeanExample init-method is called
******************************
AnnotationBeanExample init-method is called
******************************
AnnotationBeanExample afterPropertiesSet has been created
==> aoe ContextLoaderListener contextInitialized end。。。2019-6-28 18:24:24 org.springframework.web.context.ContextLoader initWebApplicationContext
信息: Root WebApplicationContext: initialization completed in 14249 ms
```
- 补充：applicationContext.xml文件中的bean及component-scan没有优先级，默认按照其在xml中的顺序解析、实例化（但如若实例化是依赖其他bean，则可能优先实例化依赖bean）；

###### 2.2.2 基于注解的bean实例化分析
之前分析applicationContext时有讲过，applicationContext默认的命名空间是beans（其支持的常用子标签是bean）；但在applicationContext会判断当前标签是否为默认命名空间，如若不是调用自定义的元素解析：
```language
	/**
	 * Parse the elements at the root level in the document:
	 * "import", "alias", "bean".
	 * @param root the DOM root element of the document
	 */
	protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
		if (delegate.isDefaultNamespace(root)) {
			NodeList nl = root.getChildNodes();
			for (int i = 0; i < nl.getLength(); i++) {
				Node node = nl.item(i);
				if (node instanceof Element) {
					Element ele = (Element) node;
					if (delegate.isDefaultNamespace(ele)) {
 						//默认命名空间beans
						parseDefaultElement(ele, delegate);
					}
					else {
                                                //自定义命名空间
						delegate.parseCustomElement(ele);
					}
				}
			}
		}
		else {
			delegate.parseCustomElement(root);
		}
	}

```
自定义命名解析流程：
1. String namespaceUri = getNamespaceURI(node); 根据节点获取命命名空间URI。如根据context获取URI为http://www.springframework.org/schema/context（对应xml的配置：xmlns:context="http://www.springframework.org/schema/context"，其中ns即为namespace简写）
2. 通过xmlns值在spirng-beans的META_INF的spring.handlers及spring.schemas找到配置：
- > http\://www.springframework.org/schema/context=org.springframework.context.config.ContextNamespaceHandler  (命名空间处理器配置)
- > http\://www.springframework.org/schema/context/spring-context-2.5.xsd=org/springframework/context/config/spring-context-2.5.xsd （命名空间xml文件描述符xsd文件）
```language
public class ContextNamespaceHandler extends NamespaceHandlerSupport {

	public void init() {
		registerBeanDefinitionParser("property-placeholder", new PropertyPlaceholderBeanDefinitionParser());
		registerBeanDefinitionParser("property-override", new PropertyOverrideBeanDefinitionParser());
		registerBeanDefinitionParser("annotation-config", new AnnotationConfigBeanDefinitionParser());
		registerBeanDefinitionParser("component-scan", new ComponentScanBeanDefinitionParser());
		registerBeanDefinitionParser("load-time-weaver", new LoadTimeWeaverBeanDefinitionParser());
		registerBeanDefinitionParser("spring-configured", new SpringConfiguredBeanDefinitionParser());
		registerBeanDefinitionParser("mbean-export", new MBeanExportBeanDefinitionParser());
		registerBeanDefinitionParser("mbean-server", new MBeanServerBeanDefinitionParser());
	}

}
```
即针对不同的elementName指定不同的解析器Parser（默认的命名空间beans则是硬编码方式根据elementName指定不同的解析方法），如ComponentScanBeanDefinitionParser（除了base-package还支持很多配置参数，具体可查看源码）：
```language
	public BeanDefinition parse(Element element, ParserContext parserContext) {
                //获取配置的basePackages数组，支持的分隔符为，；及空格，如base-package="com.aoe;1.c d,f"
		String[] basePackages = StringUtils.tokenizeToStringArray(element.getAttribute(BASE_PACKAGE_ATTRIBUTE),
				ConfigurableApplicationContext.CONFIG_LOCATION_DELIMITERS);

		// Actually scan for bean definitions and register them.
	        //根据标签配置实例化ClassPathBeanDefinitionScanner，并设置ResourceLoade、Environment、BeanDefinitionDefaults、AutowireCandidatePatterns
		ClassPathBeanDefinitionScanner scanner = configureScanner(parserContext, element);
                //即扫描所有class完成BeanDefinition注册至registry，返回BeanDefinitionHolder集合
		Set<BeanDefinitionHolder> beanDefinitions = scanner.doScan(basePackages);
		registerComponents(parserContext.getReaderContext(), beanDefinitions, element);

		return null;
	}
```
```language
	protected Set<BeanDefinitionHolder> doScan(String... basePackages) {
		Assert.notEmpty(basePackages, "At least one base package must be specified");
		Set<BeanDefinitionHolder> beanDefinitions = new LinkedHashSet<BeanDefinitionHolder>();
		for (String basePackage : basePackages) {
                        //1.生成资源路径：classpath*:com/aoe/**/*.class ；2.获取所有class文件的Resources列表；3.每个class定义一个MetadataReader，并基于MetadataReader生成ScannedGenericBeanDefinition实例sbd；4.判断是
			Set<BeanDefinition> candidates = findCandidateComponents(basePackage);
			for (BeanDefinition candidate : candidates) {
                                //创建ScopeMetadata并candidate获取value值、获取proxyMode或设置proxyMode默认为NO（默认为SCOPE_SINGLETON）
				ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(candidate);
				candidate.setScope(scopeMetadata.getScopeName());
                                //如若通过value指定beanName则直接返回；否则根据根据className生成beanName
				String beanName = this.beanNameGenerator.generateBeanName(candidate, this.registry);
				if (candidate instanceof AbstractBeanDefinition) {
					postProcessBeanDefinition((AbstractBeanDefinition) candidate, beanName);
				}
				if (candidate instanceof AnnotatedBeanDefinition) {
//设置candidate的Primary（优先注入）、Lazy、DependsOn、Role（bean角色定义）
AnnotationConfigUtils.processCommonDefinitionAnnotations((AnnotatedBeanDefinition) candidate);
				}
				//1.检查registry是否已包含beanName对应的BeanDefinition若无则直接返回true；2.获取BeanDefinition及OriginatingBeanDefinition，检查传入的candidate与从registry获取的是否可匹配
				if (checkCandidate(beanName, candidate)) {
                                        //根据candidate、beanName实例化definitionHolder 
					BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(candidate, beanName);
                                        //因为上面设置proxyMode默认为NO，此处实际直接返回definitionHolder自身
					definitionHolder = AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
					beanDefinitions.add(definitionHolder);
                                        //1.将beanName及其对应的BeanDefinition注册到registry；2.处理beanName与alias映射关系
					registerBeanDefinition(definitionHolder, this.registry);
				}
			}						
		}
		return beanDefinitions;
	}

```
```language
	/**
	 * Scan the class path for candidate components.
	 * @param basePackage the package to check for annotated classes
	 * @return a corresponding Set of autodetected bean definitions
	 */
	public Set<BeanDefinition> findCandidateComponents(String basePackage) {
		Set<BeanDefinition> candidates = new LinkedHashSet<BeanDefinition>();
		try {
                        1.生成资源路径：classpath*:com/aoe/**/*.class
			String packageSearchPath = ResourcePatternResolver.CLASSPATH_ALL_URL_PREFIX +
					resolveBasePackage(basePackage) + "/" + this.resourcePattern;
			2.获取所有class文件的Resources列表
                        Resource[] resources = this.resourcePatternResolver.getResources(packageSearchPath);
			boolean traceEnabled = logger.isTraceEnabled();
			boolean debugEnabled = logger.isDebugEnabled();
			for (Resource resource : resources) {
				if (traceEnabled) {
					logger.trace("Scanning " + resource);
				}
				if (resource.isReadable()) {
					try {
						MetadataReader metadataReader = this.metadataReaderFactory.getMetadataReader(resource);
                                                //会根据class类的metadat判断是否有Component注解(及@Controller，@Service，@Repository注解)
						if (isCandidateComponent(metadataReader)) {
							ScannedGenericBeanDefinition sbd = new ScannedGenericBeanDefinition(metadataReader);
							sbd.setResource(resource);
							sbd.setSource(resource);
							if (isCandidateComponent(sbd)) {
								if (debugEnabled) {
									logger.debug("Identified candidate component class: " + resource);
								}
								candidates.add(sbd);
							}
                ...........省略多行
		return candidates;
	}

```
补充知识点1：
1. Component注解可用于标识某个Class是Spring的一个组件；而通过Service源码（Controller、Repository类似）可发现其也被定义为Spring的一个组件，而context:component-scan则正好是扫描符合规则的组件class
```language
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Component
public @interface Service {

	/**
	 * The value may indicate a suggestion for a logical component name,
	 * to be turned into a Spring bean in case of an autodetected component.
	 * @return the suggested component name, if any
	 */
	String value() default "";

}
```
2.同一个类可实例化多个对象（xml配置多个bean方式或者@Service标签配置@Service(value="BeanExample,BeanExample1")：
```language
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
		xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
		xmlns:context="http://www.springframework.org/schema/context"
		xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-2.5.xsd
				http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context-2.5.xsd">

	<bean id="beanExample" name="beanExample" class="com.aoe.demo.BeanExample" init-method="init" destroy-method="destroy"></bean>
	<bean id="beanExample1"  class="com.aoe.demo.BeanExample" init-method="init" destroy-method="destroy"></bean>
	<bean name="beanExample2" class="com.aoe.demo.BeanExample" init-method="init" destroy-method="destroy"></bean>
	<context:component-scan base-package="com.aoe;1.c d,f"/>
</beans>
```
spring容器初始化日志：
> Pre-instantiating singletons in org.springframework.beans.factory.support.DefaultListableBeanFactory@15fc606: defining beans [beanExample,beanExample1,beanExample2,annotationBean,annotationBeanExample,BeanExample,BeanExample1,org.springframework.context.annotation.internalConfigurationAnnotationProcessor,org.springframework.context.annotation.internalAutowiredAnnotationProcessor,org.springframework.context.annotation.internalRequiredAnnotationProcessor,org.springframework.context.annotation.internalCommonAnnotationProcessor,org.springframework.context.annotation.ConfigurationClassPostProcessor$ImportAwareBeanPostProcessor#0]; 
###### 注解的解析
 在Spring上下文初始化时会调用registerBeanPostProcessors完成这3个Processor的注册及实例化
([org.springframework.context.annotation.internalAutowiredAnnotationProcessor, org.springframework.context.annotation.internalRequiredAnnotationProcessor, org.springframework.context.annotation.internalCommonAnnotationProcessor, org.springframework.context.annotation.ConfigurationClassPostProcessor$ImportAwareBeanPostProcessor#0])
- 后续分析主要涉及两个：CommonAnnotationBeanPostProcessor、AutowiredAnnotationBeanPostProcessor。在调用DefaultListableBeanFactory类（实际AbstractAutowireCapableBeanFactory类）的doCreateBean方法获取bean对象时，doCreateBean方法内会在获取instanceWrapper（bean实例）后调用applyMergedBeanDefinitionPostProcessors（即完成bean实例化后的）


1. CommonAnnotationBeanPostProcessor-->postProcessMergedBeanDefinition-->findResourceMetadata 会查找使用Resource的Field、Methods并封装ResourceElement使用当前所属class存储至CommonAnnotationBeanPostProcessor的injectionMetadataCache以便后续使用；
2.AutowiredAnnotationBeanPostProcessor；之后在AbstractAutowireCapableBeanFactory类的doCreateBean方法获取bean对象时，createBean方法内会在获取instanceWrapper（bean实例）后调用applyMergedBeanDefinitionPostProcessors（即完成bean实例化后的），即会调用AutowiredAnnotationBeanPostProcessor的postProcessMergedBeanDefinition方法：
```language

	public void postProcessMergedBeanDefinition(RootBeanDefinition beanDefinition, Class<?> beanType, String beanName) {
		if (beanType != null) {
                        //遍历class的Field、Method并获取注解信息，基于class及注解元素、信息，调用injectionMetadataCache.put(clazz, metadata)保存并返回（metadata中包含多个AutowiredFieldElement）
			InjectionMetadata metadata = findAutowiringMetadata(beanType);
			//检查将依赖的bean信息绑定到beanDefinition
			metadata.checkConfigMembers(beanDefinition);
		}
	}
```
```language
	/**
	 * Populate the bean instance in the given BeanWrapper with the property values
	 * from the bean definition.
	 * @param beanName the name of the bean
	 * @param mbd the bean definition for the bean
	 * @param bw BeanWrapper with bean instance
	 */
	protected void populateBean(String beanName, AbstractBeanDefinition mbd, BeanWrapper bw) {
		PropertyValues pvs = mbd.getPropertyValues();
                //省略N行
		if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_NAME ||
				mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_TYPE) {
			MutablePropertyValues newPvs = new MutablePropertyValues(pvs);

			// Add property values based on autowire by name if applicable.
			if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_NAME) {
				autowireByName(beanName, mbd, bw, newPvs);
			}

			// Add property values based on autowire by type if applicable.
			if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_TYPE) {
				autowireByType(beanName, mbd, bw, newPvs);
			}

			pvs = newPvs;
		}

		boolean hasInstAwareBpps = hasInstantiationAwareBeanPostProcessors();
		boolean needsDepCheck = (mbd.getDependencyCheck() != RootBeanDefinition.DEPENDENCY_CHECK_NONE);

		if (hasInstAwareBpps || needsDepCheck) {
			PropertyDescriptor[] filteredPds = filterPropertyDescriptorsForDependencyCheck(bw);
			if (hasInstAwareBpps) {
				for (BeanPostProcessor bp : getBeanPostProcessors()) {
					if (bp instanceof InstantiationAwareBeanPostProcessor) {
						InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
                                                //最核心的是findResourceMetadata，获取Resources注解并封装为ResourceElement并添加至metadata
						pvs = ibp.postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName);
						if (pvs == null) {
							return;
						}
					}
				}
			}
			if (needsDepCheck) {
				checkDependencies(beanName, mbd, filteredPds, pvs);
			}
		}

		applyPropertyValues(beanName, mbd, bw, pvs);
	}
```

###### 对象的注入
同上面的注解解析，AbstractAutowireCapableBeanFactory类的doCreateBean方法获取bean对象时，会在调用applyMergedBeanDefinitionPostProcessors之后调用其内部 populateBean方法

###### BeanPostProcessor接口作用：
如果我们想在Spring容器中完成bean实例化、配置以及其他初始化方法前后要添加一些自己逻辑处理。我们需要定义一个或多个BeanPostProcessor接口实现类，然后注册到Spring IoC容器中。