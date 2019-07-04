该系列源码可查看：https://github.com/better-yulong/study-code.git
在整合mybatis、spring时，xml配置：
```language
	<bean class="org.mybatis.spring.mapper.MapperScannerConfigurer">
		<property name="basePackage" value="com.sfpay.sypay.**.dao" />
	</bean>
```
而MapperScannerConfigurerTest的单元测试源码：
```language
public final class MapperScannerConfigurerTest {
    private GenericApplicationContext applicationContext;

    @Before
    public void setupContext() {
        applicationContext = new GenericApplicationContext();

        // add the mapper scanner as a bean definition rather than explicitly setting a
        // postProcessor on the context so initialization follows the same code path as reading from
        // an XML config file
        GenericBeanDefinition definition = new GenericBeanDefinition();
        definition.setBeanClass(MapperScannerConfigurer.class);
        definition.getPropertyValues().add("basePackage", "org.mybatis.spring.mapper");
        applicationContext.registerBeanDefinition("mapperScanner", definition);
        Object mapperScanner = applicationContext.getBean("mapperScanner"); //源码没有，另行添加
        setupSqlSessionFactory("sqlSessionFactory");

        // assume support for autowiring fields is added by MapperScannerConfigurer via
       org.springframework.context.annotation.ClassPathBeanDefinitionScanner.includeAnnotationConfig
    }
```
``` 
	applicationContext = new GenericApplicationContext();
       //对应如下构造函数
	public GenericApplicationContext() {
		this.beanFactory = new DefaultListableBeanFactory();
		this.beanFactory.setParameterNameDiscoverer(new LocalVariableTableParameterNameDiscoverer());
		this.beanFactory.setAutowireCandidateResolver(new QualifierAnnotationAutowireCandidateResolver());
	}

```
即类似基于Spirng的应用，根据很久之前分析spring源码的理解，其实可发现在解析xml中的bean标签时，即是每个bean标签生成一个GenericBeanDefinition对象，设置其BeanClass、添加PropertyValue属性集合。为深入理解Spring原理，下载源码分析

### 一.源码下载、导入及demo工程创建
#### 1.下载源码并导入（基于3.1.0.RELEASE）
从https://github.com/spring-projects/spring-framework/releases/tag/v3.1.0.RELEASE 下载源码，解压后导入eclipse。一开始各种报错，无奈决定尝试关闭所有项目逐个处理错误，仅保留spring-core 工程，发现依赖spring-asm工程，遂将该工作也打开，但是spring-core工程仍然报错：找不到许多spring-asm包中的类，开始各种折腾发现并没有什么用；于是决定看下报错类的源码，才发现下载的源码spring-asm并没有java源码，遂关闭spring-asm工程以便spring-core直接从maven仓库下载对应jar；然后将spring-core 的Java Compiler及Project Fact调整为JDK1.6,之后不再报错。（已上传至：https://github.com/better-yulong/StudyNote-Resource/tree/master/StudyNote-Resource/source-zip）
#### 2.创建spirng3-analysis包（webapp）
基于eclipse创建web 的demo应用spring3-analysis，然后在spring3-analysis的pom.xml文件中添加spirng-core的依赖（根据spring-core的pom.xml配置）
```language
	<dependency>
		<groupId>org.springframework</groupId>
		<artifactId>spring-core</artifactId>
		<version>3.1.0.RELEASE</version>
	</dependency>
```
启动后可正常访问默认index.jsp，在tomcat的webapp对应目:spring3-analysis\WEB-INF\lib，可发现spirng-core、spring-asm 的jar 包有正常依赖。
##### 2.1 新建测试类并在application.xml配置bean标签
新建BeanExample类：
```language
public class BeanExample {
	private String id;
}
```
application.xml文件（网上随便找的，对应类不存在）
```language
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE beans PUBLIC "-//SPRING//DTD BEAN 2.0//EN" "http://www.springframework.org/dtd/spring-beans-2.0.dtd">

<beans>

	<bean id="jmxAdapter" class="org.springframework.jmx.export.MBeanExporter">
		<property name="beans">
			<map>
				<entry key="bean:name=testBean1">
					<ref local="testBean"/>
				</entry>
			</map>
		</property>
	</bean>

	<bean id="testBean" class="org.springframework.jmx.JmxTestBean">
		<property name="name">
			<value>TEST</value>
		</property>
		<property name="age">
			<value>100</value>
		</property>
	</bean>

</beans>

```
- 并从spirng 源码包中搜索applicationContext.xml，然后复制到spring3-analysis的resource目录，并将名称修改为application.xml （公司的工程中会使用非默认名称application.xml)；application.xml文件中bean配置对应的class没有对应的类且依赖jar也未配置，如若加载到该xml解析时应报错。重新启动spring3-analysis工程，并未报错且原测试jsp可正常访问。
- 还原为applicationContext.xml效果相同，也不会被加载；并不像网上某些资料说只需将该文件入到对应目录即可；之后名称修改为application.xml继续后面的分析

### 二.源码分析
#### 2.1 web.xml中配置Listeners
```language
<web-app>
  <display-name>spring3-analysis</display-name>
  <listener>
    	<listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
  </listener>
</web-app>

```
ContextLoaderListener类在spring-web工程，open该工程，Java Compiler及Project Fact调整为JDK1.6，移除Java Build Path中IVY引入依赖及Project工程依赖；待处理所有报错后，在pom.xml配置spring-web的依赖：
```language
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
	</dependencies>
```
重启清理启动 ，报错：
```language
2019-6-20 18:14:09 org.springframework.web.context.ContextLoader initWebApplicationContext
严重: Context initialization failed
org.springframework.beans.factory.BeanDefinitionStoreException: IOException parsing XML document from ServletContext resource [/WEB-INF/applicationContext.xml]; nested exception is java.io.FileNotFoundException: Could not open ServletContext resource [/WEB-INF/applicationContext.xml]
	at org.springframework.beans.factory.xml.XmlBeanDefinitionReader.loadBeanDefinitions(XmlBeanDefinitionReader.java:341)
	at org.springframework.beans.factory.xml.XmlBeanDefinitionReader.loadBeanDefinitions(XmlBeanDefinitionReader.java:302)
	at org.springframework.beans.factory.support.AbstractBeanDefinitionReader.loadBeanDefinitions(AbstractBeanDefinitionReader.java:174)
	at org.springframework.beans.factory.support.AbstractBeanDefinitionReader.loadBeanDefinitions(AbstractBeanDefinitionReader.java:209)
	at org.springframework.beans.factory.support.AbstractBeanDefinitionReader.loadBeanDefinitions(AbstractBeanDefinitionReader.java:180)
	at org.springframework.web.context.support.XmlWebApplicationContext.loadBeanDefinitions(XmlWebApplicationContext.java:125)
	at org.springframework.web.context.support.XmlWebApplicationContext.loadBeanDefinitions(XmlWebApplicationContext.java:94)
	at org.springframework.context.support.AbstractRefreshableApplicationContext.refreshBeanFactory(AbstractRefreshableApplicationContext.java:131)
	at org.springframework.context.support.AbstractApplicationContext.obtainFreshBeanFactory(AbstractApplicationContext.java:522)
	at org.springframework.context.support.AbstractApplicationContext.refresh(AbstractApplicationContext.java:436)
	at org.springframework.web.context.ContextLoader.configureAndRefreshWebApplicationContext(ContextLoader.java:384)
	at org.springframework.web.context.ContextLoader.initWebApplicationContext(ContextLoader.java:283)
	at org.springframework.web.context.ContextLoaderListener.contextInitialized(ContextLoaderListener.java:111)
	at org.apache.catalina.core.StandardContext.listenerStart(StandardContext.java:4723)
	at org.apache.catalina.core.StandardContext$1.call(StandardContext.java:5226)
	at org.apache.catalina.core.StandardContext$1.call(StandardContext.java:5221)
	at java.util.concurrent.FutureTask$Sync.innerRun(Unknown Source)
	at java.util.concurrent.FutureTask.run(Unknown Source)
	at java.util.concurrent.ThreadPoolExecutor$Worker.runTask(Unknown Source)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(Unknown Source)
	at java.lang.Thread.run(Unknown Source)
Caused by: java.io.FileNotFoundException: Could not open ServletContext resource [/WEB-INF/applicationContext.xml]
	at org.springframework.web.context.support.ServletContextResource.getInputStream(ServletContextResource.java:118)
	at org.springframework.beans.factory.xml.XmlBeanDefinitionReader.loadBeanDefinitions(XmlBeanDefinitionReader.java:328)
	... 20 more
2019-6-20 18:14:09 org.apache.catalina.core.StandardContext listenerStart
严重: Exception sending context initialized event to listener instance of class org.springframework.web.context.ContextLoaderListener
org.springframework.beans.factory.BeanDefinitionStoreException: IOException parsing XML document from ServletContext resource [/WEB-INF/applicationContext.xml]; nested exception is java.io.FileNotFoundException: Could not open ServletContext resource [/WEB-INF/applicationContext.xml]
	at org.springframework.beans.factory.xml.XmlBeanDefinitionReader.loadBeanDefinitions(XmlBeanDefinitionReader.java:341)
```
从该报错可以明显看出，默认会在classpath路径查找applicationContext.xml文件；而具体核心方法调用链路为：tomcat的StandardContext.listenerStart --> spring的ContextLoaderListener.contextInitialized --> spring的ContextLoader.initWebApplicationContext --> spring的ContextLoader.configureAndRefreshWebApplicationContext --> spirng的AbstractApplicationContext.refresh  -->spirng的XmlBeanDefinitionReader.loadBeanDefinitions (中间部分方法调用省略，详细可参考上面的错误日志)
##### 2.2 tomcat的StandardContext.listenerStart
核心在StandardContext.listenerStart方法
```language
    /**
     * Configure the set of instantiated application event listeners
     * for this Context.  Return <code>true</code> if all listeners wre
     * initialized successfully, or <code>false</code> otherwise.
     */
    public boolean listenerStart() {

        if (log.isDebugEnabled())
            log.debug("Configuring application event listeners");

        // Instantiate the required listeners
        String listeners[] = findApplicationListeners();
        Object results[] = new Object[listeners.length];
        boolean ok = true;
        for (int i = 0; i < results.length; i++) {
            if (getLogger().isDebugEnabled())
                getLogger().debug(" Configuring event listener class '" +
                    listeners[i] + "'");
            try {
                results[i] = instanceManager.newInstance(listeners[i]);
            } catch (Throwable t) {
                ExceptionUtils.handleThrowable(t);
                getLogger().error
                    (sm.getString("standardContext.applicationListener",
                                  listeners[i]), t);
                ok = false;
            }
        }
        if (!ok) {
            getLogger().error(sm.getString("standardContext.applicationSkipped"));
            return (false);
        }

        // Sort listeners in two arrays
        ArrayList<Object> eventListeners = new ArrayList<Object>();
        ArrayList<Object> lifecycleListeners = new ArrayList<Object>();
        for (int i = 0; i < results.length; i++) {
            if ((results[i] instanceof ServletContextAttributeListener)
                || (results[i] instanceof ServletRequestAttributeListener)
                || (results[i] instanceof ServletRequestListener)
                || (results[i] instanceof HttpSessionAttributeListener)) {
                eventListeners.add(results[i]);
            }
            if ((results[i] instanceof ServletContextListener)
                || (results[i] instanceof HttpSessionListener)) {
                lifecycleListeners.add(results[i]);
            }
        }

        //Listeners may have been added by ServletContextInitializers.  Put them after the ones we know about.
        for (Object eventListener: getApplicationEventListeners()) {
            eventListeners.add(eventListener);
        }
        setApplicationEventListeners(eventListeners.toArray());
        for (Object lifecycleListener: getApplicationLifecycleListeners()) {
            lifecycleListeners.add(lifecycleListener);
        }
        setApplicationLifecycleListeners(lifecycleListeners.toArray());

        // Send application start events

        if (getLogger().isDebugEnabled())
            getLogger().debug("Sending application start events");

        // Ensure context is not null
        getServletContext();
        context.setNewServletContextListenerAllowed(false);
        
        Object instances[] = getApplicationLifecycleListeners();
        if (instances == null)
            return (ok);
        ServletContextEvent event =
          new ServletContextEvent(getServletContext());
        for (int i = 0; i < instances.length; i++) {
            if (instances[i] == null)
                continue;
            if (!(instances[i] instanceof ServletContextListener))
                continue;
            ServletContextListener listener =
                (ServletContextListener) instances[i];
            try {
                fireContainerEvent("beforeContextInitialized", listener);
                listener.contextInitialized(event);
                fireContainerEvent("afterContextInitialized", listener);
            } catch (Throwable t) {
                ExceptionUtils.handleThrowable(t);
                fireContainerEvent("afterContextInitialized", listener);
                getLogger().error
                    (sm.getString("standardContext.listenerStart",
                                  instances[i].getClass().getName()), t);
                ok = false;
            }
        }
        return (ok);

    }
```
即Tomcat初始化时会根据web.xml实例化Listeners对象（web.xml中配置的Listeners、Servlet、Filter对象由Servlet容器实例化），同时会调listener对象的contextInitialized方法，详细分析如下：
###### 2.2.1 Listeners 获取及实例化
String listeners[] = findApplicationListeners(); 会获取应用web.xml配置的spirng 默认org.springframework.web.context.ContextLoaderListener；那么此处其实可认为是Servlet容器提供的扩展机制，可配置多个Listen（按顺序实例化及调用）,也可通过继承已有Listener(如ContextLoaderListener)或实现ServletContextListener 接口自定义Linstener（具体实践后面另行讲解）：
```language
//建议配置为第一个Listeners，可查阅:https://www.cnblogs.com/qiankun-site/p/5886673.html
<listener>
        <listener-class>org.springframework.web.util.IntrospectorCleanupListener</listener-class>
</listener>
```
此处获取的是String[] 的listeners，即完整类名格式；而循环中调用                 results[i] = instanceManager.newInstance(listeners[i])；即完成了listener对象的实例化（底层调用class.newInstance())

###### 2.2.2 Listeners 分组
从源码可看出会将所有listener分成两个List存储；其中ServletContextAttributeListener、ServletRequestAttributeListener、ServletRequestListener、HttpSessionAttributeListener 四个接口的实现类放入eventListeners列表；而其他所有listener则放入lifecycleListeners列表
###### 2.2.3 Listenersg整合
Listeners may have been added by ServletContextInitializers.  Put them after the ones we know about.即将通过其他方式添加的Listeners获取后添加至eventListeners、lifecycleListeners列表；并通过setApplicationEventListeners、setApplicationLifecycleListeners赋值给当前StandardContext实例
###### 2.2.4 ServletContext上下文非空
通过 getServletContext();获取ServletContext并设置 context.setNewServletContextListenerAllowed(false);避免被设置新的ServletContext。其中getServletContext()会判断当前context为null，调用context = new ApplicationContext(this)完成context的实例化。---特别注意：此处是Servlet容器的context
###### 2.2.5 Listener上下文初始化
listener.contextInitialized(event);则调用到具体Listeners对象的contextInitialized方法（而event则为new ServletContextEvent(getServletContext())）。另外contextInitialized调用前后fireContainerEvent("beforeContextInitialized", listener)、fireContainerEvent("afterContextInitialized", listener)可触发容器事件（listeners定义为：The container event listeners for this Container）
*** 
##### 2.3 listener.contextInitialized(event)（即ContextLoaderListener）
```language
	/**
	 * Initialize the root web application context.
	 */
	public void contextInitialized(ServletContextEvent event) {
		this.contextLoader = createContextLoader();
		if (this.contextLoader == null) {
			this.contextLoader = this;
		}
		this.contextLoader.initWebApplicationContext(event.getServletContext());
	}
```
createContextLoader()默认直接返回null，this.contextLoader.initWebApplicationContext(event.getServletContext());最终执行的是ContextLoaderListener的父类ContextLoader的 initWebApplicationContext方法：
```language
   //ContextLoader的 initWebApplicationContext核心代码
			// Store context in local instance variable, to guarantee that
			// it is available on ServletContext shutdown.
			if (this.context == null) {
				this.context = createWebApplicationContext(servletContext);
			}
			if (this.context instanceof ConfigurableWebApplicationContext) {
				configureAndRefreshWebApplicationContext((ConfigurableWebApplicationContext)this.context, servletContext);
			}
}
```
- this.context = createWebApplicationContext(servletContext);
```language
	protected WebApplicationContext createWebApplicationContext(ServletContext sc) {
		Class<?> contextClass = determineContextClass(sc);
		if (!ConfigurableWebApplicationContext.class.isAssignableFrom(contextClass)) {
			throw new ApplicationContextException("Custom context class [" + contextClass.getName() +
					"] is not of type [" + ConfigurableWebApplicationContext.class.getName() + "]");
		}
		ConfigurableWebApplicationContext wac =
				(ConfigurableWebApplicationContext) BeanUtils.instantiateClass(contextClass);
		return wac;
	}
```
而其中determineContextClass方法中，会读取spring-web工程的ContextLoader.properties文件来获取spring容器对应的contextClass 
```language
determineContextClass方法代码：
contextClassName = defaultStrategies.getProperty(WebApplicationContext.class.getName());
//ContextLoader.properties文件
org.springframework.web.context.WebApplicationContext=org.springframework.web.context.support.XmlWebApplicationContext
```
通过如上方法即已实例化Spring的上下文context对象，然后调用configureAndRefreshWebApplicationContext方法
```language
//ContextLoader的 configureAndRefreshWebApplicationContext
	protected void configureAndRefreshWebApplicationContext(ConfigurableWebApplicationContext wac, ServletContext sc) {
		if (ObjectUtils.identityToString(wac).equals(wac.getId())) {
			// The application context id is still set to its original default value
			// -> assign a more useful id based on available information
			String idParam = sc.getInitParameter(CONTEXT_ID_PARAM);
			if (idParam != null) {
				wac.setId(idParam);
			}
			else {
				// Generate default id...
				if (sc.getMajorVersion() == 2 && sc.getMinorVersion() < 5) {
					// Servlet <= 2.4: resort to name specified in web.xml, if any.
					wac.setId(ConfigurableWebApplicationContext.APPLICATION_CONTEXT_ID_PREFIX +
							ObjectUtils.getDisplayString(sc.getServletContextName()));
				}
				else {
					wac.setId(ConfigurableWebApplicationContext.APPLICATION_CONTEXT_ID_PREFIX +
							ObjectUtils.getDisplayString(sc.getContextPath()));
				}
			}
		}

		// Determine parent for root web application context, if any.
		ApplicationContext parent = loadParentContext(sc);

		wac.setParent(parent);
		wac.setServletContext(sc);
		String initParameter = sc.getInitParameter(CONFIG_LOCATION_PARAM);
		if (initParameter != null) {
			wac.setConfigLocation(initParameter);
		}
		customizeContext(sc, wac);
		wac.refresh();
	}
```
为兼容 Servlet <= 2.4 版本，wac(XmlWebApplicationContext实例）id会做差异处理；loadParentContext(sc)获取父上下文此处默认为null，而wac.setParent(parent)、wac.setServletContext(sc)则用于指定spring容器上下文（XmlWebApplicationContext实例）的父上下文、所属Servlet上下文。
- sc.getInitParameter(CONFIG_LOCATION_PARAM)其中CONFIG_LOCATION_PARAM参数为常量值：contextConfigLocation，即获取web.xml中的初始化参数contextConfigLocation的值（如若未设置返回""）并设置wac.setConfigLocation(initParameter)，该配置可支持多种配置方式如：
```language
    <context-param>
	<param-name>contextConfigLocation</param-name>
	<param-value>classpath:application.xml
	</param-value>
    </context-param>
    <context-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>/WEB-INF/classes/spring/rest-provider.xml</param-value>
    </context-param>
```
- customizeContext(sc, wac)该方法用于获取ServletContext的contextInitializerClasses配置并实例化对应对象，如（具体后续另行分析）：
```language
<context-param>  
        <param-name>contextInitializerClasses</param-name>  
        <param-value>com.zyr.web.spring.SpringApplicationContextInitializer</param-value>  
 </context-param>  
```
在执行完上面方法好，最后一行 wac.refresh() ；看似简单实际才是重点，下一小节单元讲解
##### 2.2.4 Spring容器初始化 
wac.refresh()，war为XmlWebApplicationContext的实例，基于的XmlWebApplicationContext多级继承关系分析，最终调用的是AbstractApplicationContext的refresh()
```language
  //AbstractApplicationContext类
	public void refresh() throws BeansException, IllegalStateException {
		synchronized (this.startupShutdownMonitor) {
			// Prepare this context for refreshing.
			prepareRefresh();

			// Tell the subclass to refresh the internal bean factory.
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

			// Prepare the bean factory for use in this context.
			prepareBeanFactory(beanFactory);

			try {
				// Allows post-processing of the bean factory in context subclasses.
				postProcessBeanFactory(beanFactory);

				// Invoke factory processors registered as beans in the context.
				invokeBeanFactoryPostProcessors(beanFactory);

				// Register bean processors that intercept bean creation.
				registerBeanPostProcessors(beanFactory);

				// Initialize message source for this context.
				initMessageSource();

				// Initialize event multicaster for this context.
				initApplicationEventMulticaster();

				// Initialize other special beans in specific context subclasses.
				onRefresh();

				// Check for listener beans and register them.
				registerListeners();

				// Instantiate all remaining (non-lazy-init) singletons.
				finishBeanFactoryInitialization(beanFactory);

				// Last step: publish corresponding event.
				finishRefresh();
			}

			catch (BeansException ex) {
				// Destroy already created singletons to avoid dangling resources.
				destroyBeans();

				// Reset 'active' flag.
				cancelRefresh(ex);

				// Propagate exception to caller.
				throw ex;
			}
		}
	}
```
###### 2.2.4.1 AbstractApplicationContext类prepareRefresh()
```language
	protected void prepareRefresh() {
		this.startupDate = System.currentTimeMillis();

		synchronized (this.activeMonitor) {
			this.active = true;
		}

		if (logger.isInfoEnabled()) {
			logger.info("Refreshing " + this);
		}

		// Initialize any placeholder property sources in the context environment
		initPropertySources();

		// Validate that all properties marked as required are resolvable
		// see ConfigurablePropertyResolver#setRequiredProperties
		this.environment.validateRequiredProperties();
	}
```
 initPropertySources();该类在顶级父类为空方法，留给子类覆盖；validateRequiredProperties用于验证必须的属性（可参考https://www.cnblogs.com/wade-luffy/p/6072460.html https://blog.csdn.net/boling_cavalry/article/details/81474340）做属性扩展及必须参数的校验
```language
    //AbstractRefreshableWebApplicationContext类
	@Override
	protected void initPropertySources() {
		super.initPropertySources();
		WebApplicationContextUtils.initServletPropertySources(
				this.getEnvironment().getPropertySources(), this.servletContext,
				this.servletConfig);
	}
```
###### 2.2.4.2 AbstractApplicationContext类obtainFreshBeanFactory();
```language
  //AbstractApplicationContext类
	protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
		refreshBeanFactory();
		ConfigurableListableBeanFactory beanFactory = getBeanFactory();
		if (logger.isDebugEnabled()) {
			logger.debug("Bean factory for " + getDisplayName() + ": " + beanFactory);
		}
		return beanFactory;
	}
```
基于类继承关系,实际调用的是AbstractRefreshableApplicationContext类的refreshBeanFactory()、：销毁并关闭已有beanFactory，重新初始化beanFactory为DefaultListableBeanFactory并完成自定义的配置；getBeanFactory()方法检查beanFactory是否已实例化。其中refreshBeanFactory()很重要的一点是，其中会调用loadBeanDefinitions方法，而之前讲过当前ApplicationContext为其子类XmlWebApplicationContext，而XmlWebApplicationContext通过类名就可看出是基于xml来实现应用上下文（个人理解后面spring-boot无xml配置模式则会有新的ApplicationContext子类来实，如AnnotationConfigWebApplicationContext），正好可与applicationContext.xml文件对应，xml中每一个bean元素在解析时会被实例化一个BeanDefinition。
- 针对XmlWebApplicationContext的loadBeanDefinitions方法做下细化说明：
  1. getConfigLocations 用于获取spring上下文匹配文件：优先获取根据该方法可用于获取之前参数初始化的configLocations（根据contextConfigLocation配置赋值）；如若没有手动指定，则默认/WEB-INF/applicationContext.xml或者根据getNamespace()获取类似spring-servlet.xml配置文件路径（此处也正好可协助用一起理解springmvc子上下文与spring父上下文；到此可明白如若要自定义实现默认的applicationContext.xml命名为application.xml则需考虑自定义实现getConfigLocations方法，具体还包含其他类）
  - 从这儿对于spring全局配置文件路径及命名认识就非常清晰了；由于我将applicationContext.xml放在resource目录（编译后会进入classes目录）且在web.xml中未指定所以getConfigLocations时未读取并解析到该文件（为了验证，决定将源码XmlWebApplicationContext类的DEFAULT_CONFIG_LOCATION修改为"/WEB-INF/classes/applicationContext.xml"来验证分析是否正确。
  - ServletContext实例对象sc提供获取context-param参数的值，但仅是获取而不做处理；获取该值后，如上获取contextConfigLocation会根据;切分成locations数组，然后逐个根据路径格式（如/开头绝对路径；抑或classpath:开头路径；file开头等）基于PropertySourcesPropertyResolver成解析占位符获取路径生成configLocations数组供后续使用（前缀classpath:此处会保留）
  2. 遍历getConfigLocations结果（此） ,使用XmlBeanDefinitionReader实例的loadBeanDefinitions方法来解析xml文件及加载BeanDefinitions(实际对应其父类AbstractBeanDefinitionReader的loadBeanDefinitions)
```language
//父类AbstractBeanDefinitionReader的loadBeanDefinitions
public int loadBeanDefinitions(String location, Set<Resource> actualResources) throws BeanDefinitionStoreException {
		ResourceLoader resourceLoader = getResourceLoader();
		if (resourceLoader == null) {
			throw new BeanDefinitionStoreException(
					"Cannot import bean definitions from location [" + location + "]: no ResourceLoader available");
		}

		if (resourceLoader instanceof ResourcePatternResolver) {
			// Resource pattern matching available.
			try {
				Resource[] resources = ((ResourcePatternResolver) resourceLoader).getResources(location);
				int loadCount = loadBeanDefinitions(resources);
				if (actualResources != null) {
					for (Resource resource : resources) {
						actualResources.add(resource);
					}
				}
				if (logger.isDebugEnabled()) {
					logger.debug("Loaded " + loadCount + " bean definitions from location pattern [" + location + "]");
				}
				return loadCount;
			}
			catch (IOException ex) {
				throw new BeanDefinitionStoreException(
						"Could not resolve bean definition resource pattern [" + location + "]", ex);
			}
		}
		else {
			// Can only load single resources by absolute URL.
			Resource resource = resourceLoader.getResource(location);
			int loadCount = loadBeanDefinitions(resource);
			if (actualResources != null) {
				actualResources.add(resource);
			}
			if (logger.isDebugEnabled()) {
				logger.debug("Loaded " + loadCount + " bean definitions from location [" + location + "]");
			}
			return loadCount;
		}
	}
```
 - Resource[] resources = ((ResourcePatternResolver) resourceLoader).getResources(location)该行根据location字符串值（如classpath:applicationContext.xml）获取对应的ClassPathResource；
 - XmlBeanDefinitionReader类loadBeanDefinitions、doLoadBeanDefinitions、registerBeanDefinitions --> DefaultBeanDefinitionDocumentReader类registerBeanDefinitions、doRegisterBeanDefinitions--> DefaultBeanDefinitionDocumentReader类doRegisterBeanDefinitions、parseBeanDefinitions、parseDefaultElement、processBeanDefinition，大致步骤：使用InputStream读取xml文件；根据文件xsd及xml解析工具生成doc对象；获取beans节点及属性；获取beans所有子节点对应的element对象并获取属性值（为部分未明确指定的属性设置默认值,BeanDefinitionParserDelegate类的populateDefaults）
 - DefaultBeanDefinitionDocumentReader类processBeanDefinition方法
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
						parseDefaultElement(ele, delegate);
					}
					else {
						delegate.parseCustomElement(ele);
					}
				}
			}
		}
		else {
			delegate.parseCustomElement(root);
		}
	}

        private void parseDefaultElement(Element ele, BeanDefinitionParserDelegate delegate) {
		if (delegate.nodeNameEquals(ele, IMPORT_ELEMENT)) {
			importBeanDefinitionResource(ele);
		}
		else if (delegate.nodeNameEquals(ele, ALIAS_ELEMENT)) {
			processAliasRegistration(ele);
		}
		else if (delegate.nodeNameEquals(ele, BEAN_ELEMENT)) {
			processBeanDefinition(ele, delegate);
		}
		else if (delegate.nodeNameEquals(ele, NESTED_BEANS_ELEMENT)) {
			// recurse
			doRegisterBeanDefinitions(ele);
		}
	}
	/**
	 * Process the given bean element, parsing the bean definition
	 * and registering it with the registry.
	 */
	protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
		BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
		if (bdHolder != null) {
			bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);
			try {
				// Register the final decorated instance.
				BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry());
			}
			catch (BeanDefinitionStoreException ex) {
				getReaderContext().error("Failed to register bean definition with name '" +
						bdHolder.getBeanName() + "'", ele, ex);
			}
			// Send registration event.
			getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder));
		}
	}
```
###### 解析并获取BeanDefinitionHolder 
上面可看出parseDefaultElement用于解析默认元素；beans节点默认只支持import、alias、bean、beans四类子节点，因重点分析bean标签解析故上面只贴了processBeanDefinition方法源码；而parseCustomElement则用于支持解析自定义标签（如aop、context、tx等自定义标签）。spring4个标签示例如： 
 ```language
    <import resource="user-appalicationContext.xml"/>
    <bean id="user" class="cn.com.infcn.test.User"></bean>
    <alias name="user" alias="myUser" />
    <beans>
        //嵌套 beans标签
    </beans>
```
 - 其中代码：BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele) 对应BeanDefinitionParserDelegate类
```language
	public BeanDefinitionHolder parseBeanDefinitionElement(Element ele, BeanDefinition containingBean) {
                // 首先获取id和name属性
		String id = ele.getAttribute(ID_ATTRIBUTE); //id
		String nameAttr = ele.getAttribute(NAME_ATTRIBUTE);
                //// name属性同时作为bean的别名保存起来
		List<String> aliases = new ArrayList<String>();
		if (StringUtils.hasLength(nameAttr)) {
			String[] nameArr = StringUtils.tokenizeToStringArray(nameAttr, MULTI_VALUE_ATTRIBUTE_DELIMITERS);
			aliases.addAll(Arrays.asList(nameArr));
		}
                //如果只配置了name，没有配置id，则从aliases移除并获取第一个name作为beanName 
		String beanName = id;
		if (!StringUtils.hasText(beanName) && !aliases.isEmpty()) {
			beanName = aliases.remove(0);
			if (logger.isDebugEnabled()) {
				logger.debug("No XML 'id' specified - using '" + beanName +
						"' as bean name and " + aliases + " as aliases");
			}
		}
                //containingBean默认传入null
		if (containingBean == null) {
                        //检验beanName、aliases是否已被使用，若未使用将beanName、aliases添加至usedNames对应的Map
			checkNameUniqueness(beanName, aliases, ele);
		}

		AbstractBeanDefinition beanDefinition = parseBeanDefinitionElement(ele, beanName, containingBean);
		if (beanDefinition != null) {
			if (!StringUtils.hasText(beanName)) {
				try {
					if (containingBean != null) {
						beanName = BeanDefinitionReaderUtils.generateBeanName(
								beanDefinition, this.readerContext.getRegistry(), true);
					}
					else {
						beanName = this.readerContext.generateBeanName(beanDefinition);
						// Register an alias for the plain bean class name, if still possible,
						// if the generator returned the class name plus a suffix.
						// This is expected for Spring 1.2/2.0 backwards compatibility.
						String beanClassName = beanDefinition.getBeanClassName();
						if (beanClassName != null &&
								beanName.startsWith(beanClassName) && beanName.length() > beanClassName.length() &&
								!this.readerContext.getRegistry().isBeanNameInUse(beanClassName)) {
							aliases.add(beanClassName);
						}
					}
					if (logger.isDebugEnabled()) {
						logger.debug("Neither XML 'id' nor 'name' specified - " +
								"using generated bean name [" + beanName + "]");
					}
				}
				catch (Exception ex) {
					error(ex.getMessage(), ele);
					return null;
				}
			}
			String[] aliasesArray = StringUtils.toStringArray(aliases);
			return new BeanDefinitionHolder(beanDefinition, beanName, aliasesArray);
		}

		return null;
	}
```
 parseBeanDefinitionElement方法：
```language
        BeanDefinitionParserDelegate类
	/**
	 * Parse the bean definition itself, without regard to name or aliases. May return
	 * <code>null</code> if problems occured during the parse of the bean definition.
	 */
	public AbstractBeanDefinition parseBeanDefinitionElement(
			Element ele, String beanName, BeanDefinition containingBean) {

		this.parseState.push(new BeanEntry(beanName));

		String className = null;
		if (ele.hasAttribute(CLASS_ATTRIBUTE)) {
			className = ele.getAttribute(CLASS_ATTRIBUTE).trim();
		}

		try {
			String parent = null;
			if (ele.hasAttribute(PARENT_ATTRIBUTE)) {
				parent = ele.getAttribute(PARENT_ATTRIBUTE);
			}
	                //创建GenericBeanDefinition并指定ParentName、其BeanClass或者BeanClassName（ <bean id="child" class="com.timo.domain.Child" parent="parent">）
			AbstractBeanDefinition bd = createBeanDefinition(className, parent);
                        //解析scope/singleton/abstract/lazy-init/autowire/dependency-check/depends-on/autowire-candidate/primary/init-method/destroy-method/factory-method/factory-bean属性值及根据默认值判断处理更新bd对象
			parseBeanDefinitionAttributes(ele, beanName, containingBean, bd);
                        //description设置
			bd.setDescription(DomUtils.getChildElementValueByTagName(ele, DESCRIPTION_ELEMENT));
                        //解析meta子节点（meta不体现在bean本身；而是一个额外的声明，当需要使用里面的信息的时候可以通过BeanDefinition的getAttribute(key)方法进行获取）
			parseMetaElements(ele, bd);
                        //获取器注入，可根据需要在配置文件指定需返回的bean，参考：https://blog.csdn.net/jishuizhipan/article/details/79391688
			parseLookupOverrideSubElements(ele, bd.getMethodOverrides());
                        //替换bean中方法：https://blog.csdn.net/qq_22912803/article/details/52503914
			parseReplacedMethodSubElements(ele, bd.getMethodOverrides());
                        //解析constructor-arg（index、type、value-type）
			parseConstructorArgElements(ele, bd);
		        //解析property（name、ref、value，以及property标签可包含子节点meta解析)
			parsePropertyElements(ele, bd);
                        //解析qualifier,通过Qualifier指定注入bean的名称（极少使用)
			parseQualifierElements(ele, bd);

			bd.setResource(this.readerContext.getResource());
			bd.setSource(extractSource(ele));

			return bd;
		}
		catch (ClassNotFoundException ex) {
			error("Bean class [" + className + "] not found", ele, ex);
		}
		catch (NoClassDefFoundError err) {
			error("Class that bean class [" + className + "] depends on not found", ele, err);
		}
		catch (Throwable ex) {
			error("Unexpected failure during bean definition parsing", ele, ex);
		}
		finally {
			this.parseState.pop();
		}

		return null;
	}
```
BeanDefinitionReaderUtils.generateBeanName及beanName = this.readerContext.generateBeanName(beanDefinition)主要针对没有beanName时根据parent、factory或者beanClassName来生成beanName并添加至aliases（如<bean class="org.springframework.jmx.export.MBeanExporter">）
以上方法执行完parseBeanDefinitionElement方法会返回new  BeanDefinitionHolder(beanDefinition, beanName, aliasesArray); 
###### BeanDefinitionParserDelegate类decorateBeanDefinitionIfRequired 
用于解析内嵌的自定义标签
###### BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry());
完成beanName注册.beanDefinitionMap.put(beanName, beanDefinition)，并绑定beanName与所有alias
至此obtainFreshBeanFactory()完成，返回DefaultListableBeanFactory并完成application.xml文件解析生成对应的BeanDefinitionHolder
###### 2.2.4.3 AbstractApplicationContext类prepareBeanFactory(beanFactory);
该方法主要用于调用beanFactory的setBeanClassLoader、addPropertyEditorRegistrar、addBeanPostProcessor、ignoreDependencyInterface、registerResolvableDependency等方法，并调用beanFactory的registerSingleton将  beanName为environment、systemPropertiessystemEnvironment及对应实例对象注册到beanFactory的registeredSingletons。（这里有个特殊处理，即对应beanName如若singletonObject为null会先使用NULL_OBJECT作为占位符）
```language
	/**
	 * Add the given singleton object to the singleton cache of this factory.
	 * <p>To be called for eager registration of singletons.
	 * @param beanName the name of the bean
	 * @param singletonObject the singleton object
	 */
	protected void addSingleton(String beanName, Object singletonObject) {
		synchronized (this.singletonObjects) {
			this.singletonObjects.put(beanName, (singletonObject != null ? singletonObject : NULL_OBJECT));
			this.singletonFactories.remove(beanName);
			this.earlySingletonObjects.remove(beanName);
			this.registeredSingletons.add(beanName);
		}
	}

```
###### 2.2.4.4 AbstractRefreshableWebApplicationContext类postProcessBeanFactory
spring中并没有具体去实现postProcessBeanFactory方法，是提供给想要实现BeanPostProcessor的三方框架使用的，主要承接前文中的prepareBeanFactory()方法。谁要使用谁就去实现，作用是在BeanFactory准备工作完成后做一些定制化的处理，一般结合BeanPostProcessor接口的实现类一起使用，注入一些重要资源（类似Application的属性和ServletContext的属性）。最后需要设置忽略这类BeanPostProcessor子接口的自动装配
```language
    //https://www.cnblogs.com/question-sky/p/6760811.html
    /**
     *注册request/session环境
     * Register request/session scopes, a {@link ServletContextAwareProcessor}, etc.
     */
    @Override
    protected void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
        //注册ServletContextAwareProcessor
        beanFactory.addBeanPostProcessor(new ServletContextAwareProcessor(this.servletContext, this.servletConfig));
        beanFactory.ignoreDependencyInterface(ServletContextAware.class);
        beanFactory.ignoreDependencyInterface(ServletConfigAware.class);
        //注册web环境，包括request、session、golableSession、application
        WebApplicationContextUtils.registerWebApplicationScopes(beanFactory, this.servletContext);
        //注册servletContext、contextParamters、contextAttributes  、servletConfig单例bean，即实际调用singletonObjects.put、registeredSingletons.add(beanName)
        WebApplicationContextUtils.registerEnvironmentBeans(beanFactory, this.servletContext, this.servletConfig);
    }
```
###### 2.2.4.5 AbstractApplicationContext类invokeBeanFactoryPostProcessors（BeanFactoryPostProcessor）
执行BeanFactoryPostProcessors对应的postProcessBeanFactory方法。BeanDefinitionRegistryPostProcessor可用于将在xml解析完成BeanDefinition之后将自定义实现BeanDefinition并注册到spring环境，以便于可通过spring管理BeanDefinition对应的对象（https://blog.csdn.net/boling_cavalry/article/details/82193692），更多提供给第三方框架使用，如Mybatis的MapperScannerConfigurer（https://www.cnblogs.com/fangjian0423/p/spring-mybatis-MapperScannerConfigurer-analysis.html）
###### 2.2.4.6 AbstractApplicationContext类registerBeanPostProcessors（BeanPostProcessor）
将处定义的addBeanPostProcessor添加至beanFactory，以便实现对bean拦截的自定义创建；如AOP，最终放进Spring容器的，必须是代理对象，而不是原先的对象 ，这样别的对象在注入时，才能获得带有切面逻辑的代理对象；BeanPostProcessor是连接IOC和AOP的桥梁。
https://www.cnblogs.com/yuxiang1/archive/2018/06/19/9199730.html
###### 2.2.4.7 AbstractApplicationContext类initMessageSource
初始化MessageSource组件（做国际化功能；消息绑定，消息解析）；
###### 2.2.4.8 AbstractApplicationContext类initApplicationEventMulticaster
初始化自定义或者默认事件监听多路广播器（ https://blog.csdn.net/yu_kang/article/details/883897000；可用于事件发布，参考：https://blog.csdn.net/weixin_39035120/article/details/86225377、https://www.cnblogs.com/takumicx/p/9972461.html
###### 2.2.4.9 AbstractRefreshableWebApplicationContext类onRefresh
执行Spring 主题的刷新 UiApplicationContextUtils.initThemeSource()
###### 2.2.4.10 AbstractRefreshableWebApplicationContext类registerListeners
注册监听器
###### 2.2.4.11 AbstractApplicationContext类finishBeanFactoryInitialization
- ConversionService：Environment主要是负责解析properties和profile，最终通过过PropertySourcesPropertyResolver这个类来处理；如Environment的<T> T getProperty(String key, Class<T> targetType)为泛型参数，properties初始值得到的是String,需要ConversionService以及其相关类从String转换成T类型。可参考：https://www.cnblogs.com/abcwt112/p/7447435.html
```language
        //http://www.imooc.com/article/details/id/253222
	/**
	 * Finish the initialization of this context's bean factory,
	 * initializing all remaining singleton beans.
	 */
	protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
		// Initialize conversion service for this context.
		// 初始化上下文的转换服务，ConversionService是一个类型转换接口
		if (beanFactory.containsBean(CONVERSION_SERVICE_BEAN_NAME) &&
				beanFactory.isTypeMatch(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class)) {
			beanFactory.setConversionService(
					beanFactory.getBean(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class));
		}

		// Stop using the temporary ClassLoader for type matching.
                //prepareBeanFactory时设置TempClassLoader；如postProcessBeanFactory时需根据beanName、class校验是否匹配
		beanFactory.setTempClassLoader(null);

		// Allow for caching all bean definition metadata, not expecting further changes.
		//设置beanFactory的configurationFrozen为true,并将所有beanDefinitionNames赋值给frozenBeanDefinitionNames
		beanFactory.freezeConfiguration();

		// Instantiate all remaining (non-lazy-init) singletons.
		beanFactory.preInstantiateSingletons();
	}
```
beanFactory.preInstantiateSingletons()初始化所有非延迟加载的单例bean
```language

	public void preInstantiateSingletons() throws BeansException {
		if (this.logger.isInfoEnabled()) {
			this.logger.info("Pre-instantiating singletons in " + this);
		}
		synchronized (this.beanDefinitionMap) {
			// Iterate over a copy to allow for init methods which in turn register new bean definitions.
			// While this may not be part of the regular factory bootstrap, it does otherwise work fine.
			List<String> beanNames = new ArrayList<String>(this.beanDefinitionNames);
			for (String beanName : beanNames) {
				RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
				if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
					if (isFactoryBean(beanName)) {
						final FactoryBean factory = (FactoryBean) getBean(FACTORY_BEAN_PREFIX + beanName);
						boolean isEagerInit;
						if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
							isEagerInit = AccessController.doPrivileged(new PrivilegedAction<Boolean>() {
								public Boolean run() {
									return ((SmartFactoryBean) factory).isEagerInit();
								}
							}, getAccessControlContext());
						}
						else {
							isEagerInit = (factory instanceof SmartFactoryBean &&
									((SmartFactoryBean) factory).isEagerInit());
						}
						if (isEagerInit) {
							getBean(beanName);
						}
					}
					else {
						getBean(beanName);
					}
				}
			}
		}
	}

```
- RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName)（实际为getMergedBeanDefinition(beanName, getBeanDefinition(beanName), null)）用于合并BeanDefinition并将其put至mergedBeanDefinitions，首先会从mergedBeanDefinitions根据beanName获取如若获取到直接返回；如若未获取到流程为：
  1. BeanDefinition bd 的parent为null，若bd类型为RootBeanDefinition，则直接cloneBeanDefinitionbd克隆返回mbd即可；如bd类型不为RootBeanDefinition则使用bd实例化RootBeanDefinition返回mbd;
  2. BeanDefinition bd 的parent不为null且是普通bean（符合!beanName.equals(parentBeanName)）则递归调用getMergedBeanDefinition完成parent的BeanDefinition实例化；若bd 的parent不为null且不是普通bean（parentBeanFactory为ConfigurableBeanFactory子类，即该 bean为ConfigurableBeanFactory的实现类，这个地方理解后续springmvc涉及到了子上下文容器会出现该场景）则从ParentBeanFactory获取parentBeanName对应的BeanDefinition实例。之后基于parent的BeanDefinition实例化新的RootBeanDefinition并mbd.overrideFrom(bd)，完合并。
  3. 在上面1、2完成之后即获取到RootBeanDefinition实例mbd，则是设置mbd的scope（这里有一点：如若containingBd不为null且其是非单例bean，那么mdb本身也不可以是单例；在使用自定义的namespace时，mbd会对应一个containingBd来关联）
- 获取RootBeanDefinition 之后最重要的方法莫过于getBean()方法（普通bean和FactoryBean差异后面补充，先分析getBean方法),对应AbstractBeanFactory的getBean()方法
```language
   //return doGetBean(name, null, null, false)
   	@SuppressWarnings("unchecked")
	protected <T> T doGetBean(
			final String name, final Class<T> requiredType, final Object[] args, boolean typeCheckOnly)
			throws BeansException {
                //处理beanName,工厂类bean的beanName前面添加&符号以与普通bean的beanName区分,具体可查看DefaultListableBeanFactory类的getBeanNamesForType方法; transformedBeanName方法用于获取不带&的beanName
		final String beanName = transformedBeanName(name);
		Object bean;

		// Eagerly check singleton cache for manually registered singletons.
               //获取Singleton实例
		Object sharedInstance = getSingleton(beanName);
		if (sharedInstance != null && args == null) {
                        //该分分主要处理：1.普通bean或者FactoryBean自身直接返回sharedInstance；2.FactoryBean但是是用来创建bean实例
			if (logger.isDebugEnabled()) {
				if (isSingletonCurrentlyInCreation(beanName)) {
					logger.debug("Returning eagerly cached instance of singleton bean '" + beanName +
							"' that is not fully initialized yet - a consequence of a circular reference");
				}
				else {
					logger.debug("Returning cached instance of singleton bean '" + beanName + "'");
				}
			}
                        //会做如下判断：1.若name是&开头但beanInstance不是FactoryBean的实例则抛出异常；2.若name是&开头或者beanInstance不是FactoryBean的实例则直接返回beanInstance；3.从cache获取实例：getCachedObjectForFactoryBean(beanName)；factoryBean实例化后传以存入factoryBeanObjectCache；4.如若cache没有，则该beanInstance为FactoryBean，然后调用getObjectFromFactoryBean(factory, beanName, !synthetic);返回bean
			bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
		}

		else {
 		       //该分支用于beanName首次
			// Fail if we're already creating this bean instance:
			// We're assumably within a circular reference.
                        //该beanName对应为原型模式且正在创建中；进行循环依赖检测，如果a中持有b，b中又持有c会报错
			if (isPrototypeCurrentlyInCreation(beanName)) {
				throw new BeanCurrentlyInCreationException(beanName);
			}

			// Check if bean definition exists in this factory.
			BeanFactory parentBeanFactory = getParentBeanFactory();
		        //如果在所有已经加载的类中没有beanName则会尝试从parentBeanFactory中检测--即使用lookup 模式处理prototype模式的bean生成：WebApplicationContext中的bean可以注入到MVC Context的bean中，反向不可以---判断工厂中是否含有当前 Bean 的定义，如果当前beanFactory不包含bean的定义且父工厂不为空，则查询父工厂
			if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
				// Not found -> check parent.
				String nameToLookup = originalBeanName(name);
				if (args != null) {
					// Delegation to parent with explicit args.
					return (T) parentBeanFactory.getBean(nameToLookup, args);
				}
				else {
					// No args -> delegate to standard getBean method.
					return parentBeanFactory.getBean(nameToLookup, requiredType);
				}
			}

			if (!typeCheckOnly) {
                               //不是类型检查而是创建bean，即alreadyCreated.add(beanName)
				markBeanAsCreated(beanName);
			}
			//获取RootBeanDefinition 并检查其isAbstract、isPrototype（抛异常）
			final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
			checkMergedBeanDefinition(mbd, beanName, args);

			// Guarantee initialization of beans that the current bean depends on.
                        //如若显示通过depends-on明确先行初始化依赖的bean，则完成依赖bean的实例化
			String[] dependsOn = mbd.getDependsOn();
			if (dependsOn != null) {
				for (String dependsOnBean : dependsOn) {
					getBean(dependsOnBean);
					registerDependentBean(dependsOnBean, beanName);
				}
			}

			// Create bean instance.
			if (mbd.isSingleton()) {
                                //getSingletonw会判断若beanName未实例化则最终调用beanName对应的ObjectFactory的getObject方法，即createBean(beanName, mbd, args)；beanName已实例化则返回该实例化对象	
				sharedInstance = getSingleton(beanName, new ObjectFactory<Object>() {
					public Object getObject() throws BeansException {
						try { 
                                                        //实例化bean
							return createBean(beanName, mbd, args);
						}
						catch (BeansException ex) {
							// Explicitly remove instance from singleton cache: It might have been put there
							// eagerly by the creation process, to allow for circular reference resolution.
							// Also remove any beans that received a temporary reference to the bean.
							destroySingleton(beanName);
							throw ex;
						}
					}
				});
				//同上，若是调用对应的getBean方法则调用getObjectFromFactoryBean；否则则直接返回bean实例
				bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
			}

			else if (mbd.isPrototype()) {
				// It's a prototype -> create a new instance.
				Object prototypeInstance = null;
				try {
                                        //将beanName添加至prototypesCurrentlyInCreation；多次调用会被添加多次（类似于引用计数）
					beforePrototypeCreation(beanName);
                                        //创建新的实例
					prototypeInstance = createBean(beanName, mbd, args);
				}
				finally {
                                         //将beanName从prototypesCurrentlyInCreation移除
					afterPrototypeCreation(beanName);
				}                            
		                //同上，若是调用对应的getBean方法则调用getObjectFromFactoryBean；否则则直接返回bean实例
				bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
			}

			else {
				String scopeName = mbd.getScope();
				final Scope scope = this.scopes.get(scopeName);
				if (scope == null) {
					throw new IllegalStateException("No Scope registered for scope '" + scopeName + "'");
				}
                                //除了标准的singleton、prototype外，还可扩展支持session、request、globalSession等；但看源码此处发现没有什么区别
				try {
					Object scopedInstance = scope.get(beanName, new ObjectFactory<Object>() {
						public Object getObject() throws BeansException {
							beforePrototypeCreation(beanName);
							try {
								return createBean(beanName, mbd, args);
							}
							finally {
								afterPrototypeCreation(beanName);
							}
						}
					});
					bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
				}
				catch (IllegalStateException ex) {
					throw new BeanCreationException(beanName,
							"Scope '" + scopeName + "' is not active for the current thread; " +
							"consider defining a scoped proxy for this bean if you intend to refer to it from a singleton",
							ex);
				}
			}
		}

		// Check if required type matches the type of the actual bean instance.
                //如若指定返回对象的类型，则对对象做强制类型转换
		if (requiredType != null && bean != null && !requiredType.isAssignableFrom(bean.getClass())) {
			try {
				return getTypeConverter().convertIfNecessary(bean, requiredType);
			}
			catch (TypeMismatchException ex) {
				if (logger.isDebugEnabled()) {
					logger.debug("Failed to convert bean '" + name + "' to required type [" +
							ClassUtils.getQualifiedName(requiredType) + "]", ex);
				}
				throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
			}
		}
		return (T) bean;
	}

```
- getBean()方法是有可能触发Bean实例化阶段的活动，是因为只有当对应某个bean定义的getBean()方法第一次被调用时，不管是显式的还是隐式的，Bean实例化阶段的活动才会被触发，第二次被调用则会直接返回容器缓存的第一次实例化完的对象实例（prototype类型bean除外）。当getBean()方法内部发现该bean定义之前还没有被实例化之后，会通过createBean()方法来进行具体的对象实例化。
- 原型Bean:原型的Bean指的就是每次请求Bean实例的时候，返回的都是新实例的Bean对象。也就是说，每次注入到另外的Bean或者通过调用getBean()来获得的Bean都将是全新的实例。 

###### 2.2.4.11 @Lazy或xml中 lazy-init="true"分析
1. BeanDefinition定义：xml解析生成BeanDefinition会判断lazy-init会签；或者scan扫描到BeanDefinition之后会判断是否有@Lazy，之后会调用BeanDefinition的setLazyInit方法设置值；
2. 前面分支的容器启动时会调用代码完成单例bean的实例化finishBeanFactoryInitialization(beanFactory)中有明确判断不实例化 lazy-init 对象
3. @Lazy的Bean实例化分析
```language
@Repository
//@Lazy
public class LazyInitBeanExample {
	
	@PostConstruct
	public void init(){
		System.out.println("LazyInitBeanExample init...");
	}
}
```
通过启用、及注释分别运行应用，从启动日志可可看出添加@Lazy该bean不会有启动时实例化（注释@Lazy后则可在应用启动时实例化），为了验证该需确认在容器启动之后调用该bean来触发其实例化，通过查看spring容器的启动源码，可考虑在AbstractApplicationContext在finishRefresh()方法来处理（因其在容器初始化结束后会触发事件）：
```language
@Component
public class LazyInitTestLinstener implements ApplicationListener {
	
	@Autowired
	private LazyInitBeanExample lazyInitBeanExample;

	public void onApplicationEvent(ApplicationEvent event) {
		if(event instanceof ContextRefreshedEvent){
			System.out.println("lazyInitBeanExample:" + lazyInitBeanExample.hashCode());
		}
		
	}
	
	@PostConstruct
	public void init(){
		System.out.println("LazyInitTestLinstener ... ");
	}

}
```
- 本来预期是在onApplicationEvent方法接受到对应事件才会实例化lazyInitBeanExample，但是却发现在此之前已经实例化了，那空间是哪儿实例化的呢？分析源码调试终于发现，原来是因为@Autowired注解处理时就会实例化的：当延迟初始化的bean被注入到了一个非延迟初始化singleton bean时，也会触发其初始化。
- 通过上面的分析其实对singleton、prototype、延迟初始化bean具体在何是实例化有了比较清晰的认识：
  1. singleton（非延迟初始化bean）在容器初始化时会主动初始化；
  2. 而prototype、延迟初始化bean则是在singleton（非延迟初始化bean）实例化之后解析注解的实例对象才会在Processor中被动初始化。
- 既然有了上面的理解，那就知道上面的Linstener是无法实例在使用时实例化，而是基于注解



###### 2.2.4.12 AbstractApplicationContext类finishRefresh()
正常流程的最后 一个方法，实例化DefaultLifecycleProcessor，并调用其onRefresh()，标识spring容器为running状态；最后publishEvent(new ContextRefreshedEvent(this))。至于excepiton后的destroyBeans()、cancelRefresh(ex)等流程就不分析了。

###### 2.2.4.13 额外说明
上面源码分析涉及很多类，除了spring-web jar包外，还主要涉及spirng-context（ApplicationContext及其各子接口或子类ConfigurableApplicationContext、AbstractApplicationContext、AbstractRefreshableApplicationContext及XmlWebApplicationContext），还涉及spring-beans（ListableBeanFactory、HierarchicalBeanFactory、AbstractBeanFactory、BeanFactoryUtils）

### 补充知识点
#### depends-on与ref
- depends-on是bean标签的属性之一，表示一个bean对其他bean的依赖关系。乍一想，不是有ref吗？其实还是有区别的，<ref/>标签是一个bean对其他bean的引用，而depends-on属性只是表明依赖关系（不一定会引用），这个依赖关系决定了被依赖的bean必定会在依赖bean之前被实例化，反过来，容器关闭时，依赖bean会在被依赖的bean之前被销毁；manager和accoutDao会先于beanOne被实例化，会慢于beanOne被销毁，而beanOne不引用accountDao（或者说beanOne不会将accountDao注入到自己的属性中）。这就是depends-on的主要作用。
```
<bean id="beanOne" class="ExampleBean" depends-on="manager,accountDao">
    <property name="manager" ref="manager" />
</bean>

<bean id="manager" class="ManagerBean" />
<bean id="accountDao" class="x.y.jdbc.JdbcAccountDao" />
```
#### 父子上下文及bean依赖
- WebApplicationContext 是MVC Context的父上下文，是否可以互相注入对方的bean？（https://www.jianshu.com/p/6f9204b812da）
  - WebApplicationContext中的bean可以注入到MVC Context的bean中，反向不可以（亲测）。参见一下代码：来自AbstractBeanFactory
```
// Check if bean definition exists in this factory.
BeanFactory parentBeanFactory = getParentBeanFactory();
if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
    // Not found -> check parent.
    String nameToLookup = originalBeanName(name);
    if (args != null) {
        // Delegation to parent with explicit args.
        return (T) parentBeanFactory.getBean(nameToLookup, args);
    }
    else {
        // No args -> delegate to standard getBean method.
        return parentBeanFactory.getBean(nameToLookup, requiredType);
    }
}

```
  - 如果同一个bean被两个上下文扫描到，会怎么样？是不是有两个实例？
是的，会在两个上下文中生成两个独立的bean（亲测）
  - web.xml为什么有时候需要ContextLoaderListener，有时候又不需要？
其实是需要的，有时候不需要是因为不小心把其他的bean全部扫描进DispatchServlet的MVC Context里面了，所以不需要再加载WebApplicationContext（亲测）
  - 是不是可以只配置MVC Context而不使用WebApplicationContext?
亲测可以
#### 单例模式与原型模式互相依赖
- https://blog.csdn.net/u011120159/article/details/82218472
  1. 单例模式中注入原型bean，prototypeBean其实只实例化了一次，原型模式的作用完全没有发挥; 
  2. lookup方法注入:cglib动态生成一个的子类，该子类会自动override有lookup注解的方法，代码干净整洁。
  - Spring提供了EarlyBeanReference功能，首先Spring里面有个名字为singletonObjects的并发map用来存放所有实例化并且初始化好的bean，singletonFactories则用来存放需要解决循环依赖的bean信息（beanName,和一个回调工厂）。当实例化beanA时候会触发getBean(“beanA”)。普通Bean循环依赖-与注入顺序无关，可正常注入；工厂Bean与普通Bean循环依赖-与注入顺序有关，可能注入失败（http://ifeve.com/%E8%AE%BAspring%E4%B8%AD%E5%BE%AA%E7%8E%AF%E4%BE%9D%E8%B5%96%E7%9A%84%E6%AD%A3%E7%A1%AE%E6%80%A7%E4%B8%8Ebean%E6%B3%A8%E5%85%A5%E7%9A%84%E9%A1%BA%E5%BA%8F%E5%85%B3%E7%B3%BB/）

