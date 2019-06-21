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
即类似基于Spirng的应用，容初始化也是默认，根据很久之前分析spring源码的理解，其实可发现在解析xml中的bean标签时，即是每个bean标签生成一个GenericBeanDefinition对象，设置其BeanClass、添加PropertyValue属性集合。为深入理解，下载源码分析

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
application.xml文件
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
##### 2.2 web.xml中配置Listeners
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
###### 2.2.1 tomcat的StandardContext.listenerStart
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
1. Listeners 获取及实例化
String listeners[] = findApplicationListeners(); 会获取应用web.xml配置的spirng 默认org.springframework.web.context.ContextLoaderListener；那么此处其实可认为是Servlet容器提供的扩展机制，可配置多个Listen（按顺序实例化及调用）,也可通过继承已有Listener(如ContextLoaderListener)或实现ServletContextListener 接口自定义Linstener（具体实践后面另行讲解）：
```language
--建议配置为第一个Listeners，可查阅:https://www.cnblogs.com/qiankun-site/p/5886673.html
<listener>
        <listener-class>org.springframework.web.util.IntrospectorCleanupListener</listener-class>
</listener>
```
此处获取的是String[] 的listeners，即完整类名格式；而循环中调用                 results[i] = instanceManager.newInstance(listeners[i])；即完成了listener对象的实例化（底层调用class.newInstance()).
2. Listeners 分组
从源码可看出会将所有listener分成两个List存储；其中ServletContextAttributeListener、ServletRequestAttributeListener、ServletRequestListener、HttpSessionAttributeListener 四个接口的实现类放入eventListeners列表；而其他所有listener则放入lifecycleListeners列表
3. Listenersg整合
Listeners may have been added by ServletContextInitializers.  Put them after the ones we know about.即将通过其他方式添加的Listeners获取后添加至eventListeners、lifecycleListeners列表；并通过setApplicationEventListeners、setApplicationLifecycleListeners赋值给当前StandardContext实例
4. ServletContext上下文非空
通过 getServletContext();获取ServletContext并设置 context.setNewServletContextListenerAllowed(false);





















```language
 public class ContextLoaderListener extends ContextLoader implements ServletContextListener 
```

新建framework-aoe-web 工程（类似spirng-web），新建







Eclipse中Dependency Hierarchy的语法树层级显示及spring3-analysis工程根目录（与pom.xml同级）运行 mvn dependency:tree 分析，只需配置spirng-meb即可自动依赖所需jar：
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














