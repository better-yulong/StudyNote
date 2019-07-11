Spring MVC 属于Spring Framework web子模块，整个系统均以3.1.0.RELEASE版本；Spring Framework的doc文件(spring-framework-3.2.0.RELEASE-docs)已上传至git 资源仓库。
### 一.源码
参考Spring源码分析，其中已包含spring-webmvc子模块源码（对应：org.springframework.web.servlet）。

### 二.Demo实践
#### 2.1 Demo框架及验证
1. 新建基于maven webapp的springmvc3-analysis工程，pom.xml配置spring-webmvc 依赖jar包（根据之前的分析，配置spring-mvcweb jar基于依赖传递会自动解析关联依赖jar；发现spring-framework jar基本已自动包含，无需单独引入）
```language
  <groupId>com.aoe.demo</groupId>
  <artifactId>springmvc3-analysis</artifactId>
  <packaging>war</packaging>
  <version>0.0.1-SNAPSHOT</version>
  <name>springmvc-analysis Maven Webapp</name>
  <url>http://maven.apache.org</url>
  <dependencies>
    <dependency>
		<groupId>org.springframework</groupId>
			<artifactId>spring-webmvc</artifactId>
			<version>3.1.0.RELEASE</version>
		</dependency>
  </dependencies>
```
注：该示例引入了spring-context-support、spring-webmvc两个工程的源码，但因其内部又依赖了jasperreports相关jar, 而jasperreports不需要，于是简单处理，移除jasperreports相关jar依赖及代码。
2. web.xml配置
```language
      <servlet>
        <servlet-name>example</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
        <load-on-startup>1</load-on-startup>
    </servlet>

    <servlet-mapping>
        <servlet-name>example</servlet-name>
        <url-pattern>/hello/*</url-pattern>
    </servlet-mapping>
```
3. Controller示例代码
```language
@Controller
public class HelloWorldController {

    @RequestMapping("/hello/helloWorld")
    public String helloWorld(Model model) {
        model.addAttribute("message", "Hello World!");
        return "helloWorld";
    }
}
```
然而，项目启动时直接报错：
```language
信息: Refreshing WebApplicationContext for namespace 'example-servlet': startup date [Sat Jul 06 14:17:06 CST 2019]; root of context hierarchy
2019-7-6 14:17:06 org.springframework.beans.factory.xml.XmlBeanDefinitionReader loadBeanDefinitions
信息: Loading XML bean definitions from ServletContext resource [/WEB-INF/example-servlet.xml]
2019-7-6 14:17:06 org.springframework.web.servlet.FrameworkServlet initServletBean
严重: Context initialization failed
org.springframework.beans.factory.BeanDefinitionStoreException: IOException parsing XML document from ServletContext resource [/WEB-INF/example-servlet.xml]; nested exception is java.io.FileNotFoundException: Could not open ServletContext resource [/WEB-INF/example-servlet.xml]
	at org.springframework.beans.factory.xml.XmlBeanDefinitionReader.loadBeanDefinitions(XmlBeanDefinitionReader.java:341)
	at org.springframework.beans.factory.xml.XmlBeanDefinitionReader.loadBeanDefinitions(XmlBeanDefinitionReader.java:302)
	
```
错误比较一目了然，即WEB_INF目录缺少springmvc的配置文件example-servlet.xml，该文件命名规则即web.xml中的servlet-name加上默认-servlet.xml即可；于是乎在源码工程中查找-servlet.xml结尾的文件找到empty-servlet.xml。
4. 新增example-servlet.xml（不做任何配置）：
```language
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE beans PUBLIC "-//SPRING//DTD BEAN 2.0//EN" "http://www.springframework.org/dtd/spring-beans-2.0.dtd">

<beans>

</beans>
```
运行时应用正常启动，从日志可发现已启动并加载example-servlet.xml。访问：http://localhost:8080/springmvc3-analysis 可正常返回默认的index.jsp页面。但访问http://localhost:8080/springmvc3-analysis/hello/helloWorld则为404；从访问后的控制台日志可看出没有Handler、mapping映射关系：
```language
2019-7-6 14:23:31 org.springframework.web.servlet.DispatcherServlet noHandlerFound
警告: No mapping found for HTTP request with URI [/springmvc3-analysis/hello/helloWorld] in DispatcherServlet with name 'example'
```
那具体需要怎么修改呢？与其百度还不如看源码啦...

### 三.问题错误及定位
基于之前的分析和经验，先从DispatcherServlet 源码入手，而正好其源码也正好有"No mapping found for HTTP"的日志输出，直接在此处断点调试，进入断点时，栈信息：
```language
Daemon Thread ["http-bio-8080"-exec-3] (Suspended (breakpoint at line 1056 in DispatcherServlet))	
	owns: SocketWrapper<E>  (id=54)	
	DispatcherServlet.noHandlerFound(HttpServletRequest, HttpServletResponse) line: 1056	
	DispatcherServlet.doDispatch(HttpServletRequest, HttpServletResponse) line: 865	
	DispatcherServlet.doService(HttpServletRequest, HttpServletResponse) line: 827	
	DispatcherServlet(FrameworkServlet).processRequest(HttpServletRequest, HttpServletResponse) line: 882	
	DispatcherServlet(FrameworkServlet).doGet(HttpServletRequest, HttpServletResponse) line: 778	
	DispatcherServlet(HttpServlet).service(HttpServletRequest, HttpServletResponse) line: 621	
	DispatcherServlet(HttpServlet).service(ServletRequest, ServletResponse) line: 722	
	ApplicationFilterChain.internalDoFilter(ServletRequest, ServletResponse) line: 304	
	ApplicationFilterChain.doFilter(ServletRequest, ServletResponse) line: 210	
	StandardWrapperValve.invoke(Request, Response) line: 240	
	StandardContextValve.invoke(Request, Response) line: 164	
	NonLoginAuthenticator(AuthenticatorBase).invoke(Request, Response) line: 462	
	StandardHostValve.invoke(Request, Response) line: 164	
	ErrorReportValve.invoke(Request, Response) line: 100	
	AccessLogValve.invoke(Request, Response) line: 563	
	StandardEngineValve.invoke(Request, Response) line: 118	
	CoyoteAdapter.service(Request, Response) line: 399	
	Http11Processor.process(SocketWrapper<Socket>) line: 317	
	Http11Protocol$Http11ConnectionHandler.process(SocketWrapper<Socket>, SocketStatus) line: 204	
	Http11Protocol$Http11ConnectionHandler.process(SocketWrapper<Socket>) line: 182	
	JIoEndpoint$SocketProcessor.run() line: 311	
	ThreadPoolExecutor$Worker.runTask(Runnable) line: not available	
	ThreadPoolExecutor$Worker.run() line: not available	
	TaskThread(Thread).run() line: not available	
```
从栈信息来看，主要处理为：1.tomcat启动新的线程，并创建Socket；2.将请求交由tomcat容器（对request对象、reponse对象进行处理，这其中就有基于各种filter对request对象、reponse对象进行处理）并调用servlet进行处理；3.调用DispatcherServlet的service方法。其中前2点与tomcat相关以后单独章节分析tomcat源码，而第3点则正常也当前分析的spring-webmvc有关。

#### 3.1 Servlet请求分析
此外调用DispatcherServlet的service方法，但实际执行的是servlet-api.jar中的javax.servlet.http.HttpServlet(抽象类; DispatcherServlet间接继承HttpServlet)的service方法，而service主要做了啥呢？即获取request的Methods，并根据不两只的method类型调用不同的方法（doGet、doPost），其实这点也就印证最开始学习JavaWeb时所讲的，写一个Servlet需继承HttpServlet并根据需要实现doGet、doPost方法。虽然不同的Servlet容器（tomcat、jetty、weblogic等）实现可能各有差异，但都会遵循该规则。
#### 3.2 SpringMVC Servlet接收请求处理逻辑
无论doGet抑或doPost方法内部，均是直接调用FrameworkServlet（DispatcherServlet的直接父类）的processRequest方法（移除了部分次要代码）：
```language
	protected final void processRequest(HttpServletRequest request, HttpServletResponse response)
			throws ServletException, IOException {

		long startTime = System.currentTimeMillis();
		Throwable failureCause = null;

		//1.获取上一次请求的本地上下文
		LocaleContext previousLocaleContext = LocaleContextHolder.getLocaleContext();
                //2.创建当前请求的本地上下文
		LocaleContextHolder.setLocaleContext(buildLocaleContext(request), this.threadContextInheritable);

		// Expose current RequestAttributes to current thread.
                //3.获取上一次请求的RequestAttributes
		RequestAttributes previousRequestAttributes = RequestContextHolder.getRequestAttributes();
		ServletRequestAttributes requestAttributes = null;
                //4.如若上一次没有RequestAttributes则当前请求创建新的requestAttributes （推断适用常量 应该为服务端的forward模式可传递数据）
		if (previousRequestAttributes == null || previousRequestAttributes.getClass().equals(ServletRequestAttributes.class)) {
			requestAttributes = new ServletRequestAttributes(request);
			RequestContextHolder.setRequestAttributes(requestAttributes, this.threadContextInheritable);
		}

		try {   
                        //处理请求，单独分析
			doService(request, response);
		}
		catch (ServletException ex) {}

		finally {
			// Clear request attributes and reset thread-bound context.
                        //上下文处理（清理、继承）--inheritableLocaleContextHolder、localeContextHolder
			LocaleContextHolder.setLocaleContext(previousLocaleContext, this.threadContextInheritable);
                        //请求属性处理（清理、继承）--inheritableRequestAttributesHolder、requestAttributesHolder
			if (requestAttributes != null) {
				RequestContextHolder.setRequestAttributes(previousRequestAttributes, this.threadContextInheritable);
				requestAttributes.requestCompleted();
			}
			if (this.publishEvents) {
                               //无论处理成功与否，发布事件
				// Whether or not we succeeded, publish an event.
				long processingTime = System.currentTimeMillis() - startTime;
				this.webApplicationContext.publishEvent(
			new ServletRequestHandledEvent(this,
								request.getRequestURI(), request.getRemoteAddr(),
								request.getMethod(), getServletConfig().getServletName(),
								WebUtils.getSessionId(request), getUsernameForRequest(request),
								processingTime, failureCause));
			}
		}
	}
```
#### 3.2.1 DispatcherServlet的doService方法（移除了部分次要代码）
```	
protected void doService(HttpServletRequest request, HttpServletResponse response) throws Exception {
		// Keep a snapshot of the request attributes in case of an include,
		// to be able to restore the original attributes after the include.
		Map<String, Object> attributesSnapshot = null;
                //暂存请求中的Attribute数据，可避免应用中被remove
		if (WebUtils.isIncludeRequest(request)) {
			logger.debug("Taking snapshot of request attributes before include");
			attributesSnapshot = new HashMap<String, Object>();
			Enumeration<?> attrNames = request.getAttributeNames();
			while (attrNames.hasMoreElements()) {
				String attrName = (String) attrNames.nextElement();
				if (this.cleanupAfterInclude || attrName.startsWith("org.springframework.web.servlet")) {
					attributesSnapshot.put(attrName, request.getAttribute(attrName));
				}
			}
		}
                
                //该方法可获取上一次request中的flashMap转存数据（通过源码发现request之前仍然可以实现临时的数据共享，可参考源码，分析主要适用于redirect场景，也可详细了解RedirectAttributes
		this.flashMapManager.requestStarted(request);

		// Make framework objects available to handlers and view objects.
                //初始化设计 request属性
		request.setAttribute(WEB_APPLICATION_CONTEXT_ATTRIBUTE, getWebApplicationContext());
		request.setAttribute(LOCALE_RESOLVER_ATTRIBUTE, this.localeResolver);
		request.setAttribute(THEME_RESOLVER_ATTRIBUTE, this.themeResolver);
		request.setAttribute(THEME_SOURCE_ATTRIBUTE, getThemeSource());

		try {
                        //核心逻辑单独分析
			doDispatch(request, response);
		}
		finally {
                        //保存当前请求需转存供下次请求的数据
			this.flashMapManager.requestCompleted(request);
			
			// Restore the original attribute snapshot, in case of an include.
                        //处理Attributes：移除部分Attributes并重置部分Attributes
			if (attributesSnapshot != null) {
				restoreAttributesAfterInclude(request, attributesSnapshot);
			}
		}
	}

```
#### 3.2.2 DispatcherServlet的doDispatch方法（移除了部分次要代码）
```language
	protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
		HttpServletRequest processedRequest = request;
		HandlerExecutionChain mappedHandler = null;
		int interceptorIndex = -1;

		try {
			ModelAndView mv;
			boolean errorView = false;

			try {   
                                //根据请求信息判断是否为特殊类型Content-Type：multipart文件上传）；如若是则需返回原request的包装类型，如DefaultMultipartHttpServletRequest
				processedRequest = checkMultipart(request);

				// Determine handler for the current request.
                                //根据请求获取Handler（上面的示例正好是未匹配到报错了）
				mappedHandler = getHandler(processedRequest, false);
				if (mappedHandler == null || mappedHandler.getHandler() == null) {
					noHandlerFound(processedRequest, response);
					return;
			}
		//此处实际还有较多重要代码以，目前先行分析当前问题，故省略该部分				
	}
```
而getHandler方法源码为：
```language
	protected HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
		for (HandlerMapping hm : this.handlerMappings) {
			if (logger.isTraceEnabled()) {
				logger.trace(
						"Testing handler map [" + hm + "] in DispatcherServlet with name '" + getServletName() + "'");
			}
			HandlerExecutionChain handler = hm.getHandler(request);
			if (handler != null) {
				return handler;
			}
		}
		return null;
	}
```
handlerMappings涉及到两个：
[org.springframework.web.servlet.handler.BeanNameUrlHandlerMapping@1fb80c9, org.springframework.web.servlet.mvc.annotation.DefaultAnnotationHandlerMapping@c44aaf]。都有共同的父类AbstractUrlHandlerMapping，而获取handler的逻辑相对简单：1.根据url分别从HandlerMapping的handlerMap中匹配Handler（注意BeanNameUrlHandlerMapping、DefaultAnnotationHandlerMapping各有一个实例变量）handlerMap；2.是否为/，若是则返回RootHandler；3.如若未找到Handler则返回DefaultHandler（默认为null）。而上面的示例经过查找均未匹配到Handler，所以页面返回404则日志提示未匹配到。而这些，均需分析SpringMVC的启动逻辑才能明白。

### 四、SpringMVC的启动初始化分析
针对web.xml配置
```language
     <servlet>
        <servlet-name>example</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
        <load-on-startup>1</load-on-startup>
    </servlet>

    <servlet-mapping>
        <servlet-name>example</servlet-name>
        <url-pattern>/hello/*</url-pattern>
    </servlet-mapping>
```
- web.xml的解析涉及到tomcat源码，此处简单带过，之后会有单独系列分析。简单说，即tomcat加载应用时，会调用WebXml解析web.xml文件，其中如上个servlet会由一个ServletDef对象表示（同时会创建一个默认的JspServlet用于处理jsp页面请求）（configureContext方法即解析web.xml,涉及到contextParams、filters、errorPages、listeners、servlets等解析，具体可查看源码；另也会初始化默认DefaultServlet 。另外也可发现Linster实例化先于Servlet实例化）
- StandardContext类loadOnStartup方法过滤出启动即需加载的Servlet并实例化（注释：Load the collected "load on startup" servlets）；由StandardWrapper.loadServlet-->DefaultInstanceManager.newInstance--(通过反射)-->调用DispatcherServlet的无参构造方法
#### 4.1 DispatcherServlet实例化分析
##### 4.1.1 DispatcherServlet类加载初始化
静态代码段，主要用于初始化SpringMVC默认配置文件DispatcherServlet.properties：
```language
static {
		// Load default strategy implementations from properties file.
		// This is currently strictly internal and not meant to be customized
		// by application developers.
		try {
			ClassPathResource resource = new ClassPathResource(DEFAULT_STRATEGIES_PATH, DispatcherServlet.class);
			defaultStrategies = PropertiesLoaderUtils.loadProperties(resource);
		}
		catch (IOException ex) {
			throw new IllegalStateException("Could not load 'DispatcherServlet.properties': " + ex.getMessage());
		}
	}
```
```language
#DispatcherServlet.properties文件
org.springframework.web.servlet.LocaleResolver=org.springframework.web.servlet.i18n.AcceptHeaderLocaleResolver
org.springframework.web.servlet.ThemeResolver=org.springframework.web.servlet.theme.FixedThemeResolver
org.springframework.web.servlet.HandlerMapping=org.springframework.web.servlet.handler.BeanNameUrlHandlerMapping,org.springframework.web.servlet.mvc.annotation.DefaultAnnotationHandlerMapping
org.springframework.web.servlet.HandlerAdapter=org.springframework.web.servlet.mvc.HttpRequestHandlerAdapter,org.springframework.web.servlet.mvc.SimpleControllerHandlerAdapter,org.springframework.web.servlet.mvc.annotation.AnnotationMethodHandlerAdapter
org.springframework.web.servlet.HandlerExceptionResolver=org.springframework.web.servlet.mvc.annotation.AnnotationMethodHandlerExceptionResolver,\
	org.springframework.web.servlet.mvc.annotation.ResponseStatusExceptionResolver,\
	org.springframework.web.servlet.mvc.support.DefaultHandlerExceptionResolver
org.springframework.web.servlet.RequestToViewNameTranslator=org.springframework.web.servlet.view.DefaultRequestToViewNameTranslator
org.springframework.web.servlet.ViewResolver=org.springframework.web.servlet.view.InternalResourceViewResolver
org.springframework.web.servlet.FlashMapManager=org.springframework.web.servlet.support.DefaultFlashMapManager
```
其实从这儿也可以看出来，之前上面分析url映射的HandlerMapping是哪儿配置的了。
##### 4.1.2 DispatcherServlet实例初始化
如此创建DispatcherServlet之后 ，tomcat的StandardWrapper的initServlet方法调用servlet.init(facade) -->GenericServlet.init(ServletConfig）-->HttpServletBean.init()-->FrameworkServlet.initServletBean(); 这个方法才是关键，其中两个核心方法：
```language
    this.webApplicationContext = initWebApplicationContext();
    initFrameworkServlet();
```
###### 4.1.2.1 初始化上下文 FrameworkServlet.initWebApplicationContext()
```language
	protected WebApplicationContext initWebApplicationContext() {
                //获取Spring ROOT Conext，即对应Spring的ApplicationContext.ROOT(个人理解，即对应之前分析 Spring源码启动时根据Listener、applicationContext.xml创建的ApplicationContext。其实之前很多人会有疑问，为什么我不配置Listener、applicationContext.xml而仅配置DispatcherServlet,其实正是为了此处分析及父子上下文的理解
		WebApplicationContext rootContext =
			WebApplicationContextUtils.getWebApplicationContext(getServletContext());
		WebApplicationContext wac = null;

		if (this.webApplicationContext != null) {
			// A context instance was injected at construction time -> use it
                        //从注释来看即已有上下文实例，即若未激活则类似于之前的Spring分析时的过程(当前示例this.webApplicationContext为null，不会运行至此处；即获取Servlet关联的webApplicationContext
			wac = this.webApplicationContext;
			if (wac instanceof ConfigurableWebApplicationContext) {
				ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext) wac;
				if (!cwac.isActive()) {
					// The context has not yet been refreshed -> provide services such as
					// setting the parent context, setting the application context id, etc
					if (cwac.getParent() == null) {
						// The context instance was injected without an explicit parent -> set
						// the root application context (if any; may be null) as the parent
						cwac.setParent(rootContext);
					}
                                        //配置并刷新已有的WebApplicationContext
					configureAndRefreshWebApplicationContext(cwac);
				}
			}
		}
		if (wac == null) {
                        //当前示例wac为null，不会运行至此处;即尝试获取注册到ServleContext中context实例
			// No context instance was injected at construction time -> see if one
			// has been registered in the servlet context. If one exists, it is assumed
			// that the parent context (if any) has already been set and that the
			// user has performed any initialization such as setting the context id
			wac = findWebApplicationContext();
		}
		if (wac == null) {
			// No context instance is defined for this servlet -> create a local one
                        //此处是创建新的context ，新的context的id的Parents为null,其id是org.springframework.web.context.WebApplicationContext:/springmvc3-analysis/example，namespace为example-servlet；最后的操作则与之前分析Spring一样，解析其对应的example-servlet.xml文件完成子上下文（容器）的初始化（包括添加该容器的Linstener：SourceFilteringListener、ContextRefreshListener（而关于如Mapping等处理均在Linster，稍后单独分析
			wac = createWebApplicationContext(rootContext);
		}

		if (!this.refreshEventReceived) {
			// Either the context is not a ConfigurableApplicationContext with refresh
			// support or the context injected at construction time had already been
			// refreshed -> trigger initial onRefresh manually here.
                         //用于处理非ConfigurableApplicationContext的容器初始化
			onRefresh(wac);
		}

		if (this.publishContext) {
			// Publish the context as a servlet context attribute.
			String attrName = getServletContextAttributeName();
			getServletContext().setAttribute(attrName, wac);
		}

		return wac;
	}
```
###### 4.1.2.2 SpringMVC容器Listener运行分析 
SpringMVC初始化时，容器初始化结束发布事件会触发SourceFilteringListener.onApplicationEventInternal-->GenericApplicationListenerAdapter.onApplicationEvent-->FrameworkServlet$ContextRefreshListener.onApplicationEvent-->DispatcherServlet.onRefresh-->DispatcherServlet.initStrategies方法
```language
        //DispatcherServlet类，context对应WebApplicationContext for namespace 'example-servlet'
	protected void initStrategies(ApplicationContext context) {
                //即如文件上传类型的的Resolver，默认为null,可为null
		initMultipartResolver(context);
                //普通的LocaleResolver，为null会创建默认的AcceptHeaderLocaleResolver实例,可用于处理http头、cookie等，如CookieLocaleResolver
		initLocaleResolver(context);
                //普通的ThemeResolver，为null会创建默认的FixedThemeResolver；加载主题资源，国际化时会使用到
		initThemeResolver(context);
                //初始化HandlerMapping，未配置会默认实例化BeanNameUrlHandlerMapping、DefaultAnnotationHandlerMapping（根据上面讲到的DispatcherServlet类加载初始化的配置文件）
		initHandlerMappings(context);
                //初始化HandlerAdapter，未配置会默认实例化HttpRequestHandlerAdapter、SimpleControllerHandlerAdapter、AnnotationMethodHandlerAdapter（根据上面讲到的DispatcherServlet类加载初始化的配置文件）
		initHandlerAdapters(context);
                //初始化HandlerExceptionResolver，未配置会默认实例化AnnotationMethodHandlerExceptionResolver、ResponseStatusExceptionResolver、DefaultHandlerExceptionResolver（根据上面讲到的DispatcherServlet类加载初始化的配置文件）
		initHandlerExceptionResolvers(context);
               //初始化RequestToViewNameTranslator，未配置会默认实例化DefaultRequestToViewNameTranslator（根据上面讲到的DispatcherServlet类加载初始化的配置文件）
		initRequestToViewNameTranslator(context);
                //初始化ViewResolver，未配置会默认实例化InternalResourceViewResolver（根据上面讲到的DispatcherServlet类加载初始化的配置文件）
		initViewResolvers(context);
                //初始化FlashMapManager，未配置会默认实例化DefaultFlashMapManager（根据上面讲到的DispatcherServlet类加载初始化的配置文件）
		initFlashMapManager(context);
	}
```
分析到这儿,其实可以认为SpringMVC容器已经初始化结束了,而之前想分析的RequestMapping注解解析究竟是哪儿处理的呢?同样,之前分析过Spring的DI(即基于@Resources、@Autoware注解）的处理逻辑就应该可以猜到其处理逻辑了。
###### 4.1.2.3 RequestMapping解析分析
同样，根据之前分析Spring源码时初步推断，SpringMVC相关注解的解析仍然需要开启注解自动扫描，但是具体如何配置呢？是需配置SpringMVC独有的注解扫描？然而，看了下Controller、@RequestMapping源码及注释发现，并没有特别对特别说明。那么结合之前的bean注入有xml配置与注解配置两种方式，那么此处就采用原始的xml配置方式来试试看（为便于验证，对原Controller类名和web.xml做了简单调整，代码为：
```language
//Controller类
@Controller
@RequestMapping("/example")
public class ExampleController {

    @RequestMapping("/helloWorld")
    @ResponseBody
    public String helloWorld(Model model) {
        model.addAttribute("message", "Hello World!");
        return "helloWorld";
    }
}
```
web.xml配置：
```language
      <servlet>
        <servlet-name>example</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
        <load-on-startup>1</load-on-startup>
    </servlet>

    <servlet-mapping>
        <servlet-name>example</servlet-name>
        <url-pattern>/example/*</url-pattern>
    </servlet-mapping>
```
example-servlet.xml：
```language
   <bean id="servletContextAwareBean" class="com.aoe.demo.springmvc.ExampleController"/>
```
启动应用，中单有个小插曲，启动时始终报ClassNotFoundException，即ExampleController.class始终找不到，去应用的target/classes目录确实找不到class文件，结合eclipse的Problems发现系JavaBuild配置报错，调整后clean应用class正常编译。启动应用日志为：
```language
信息: Refreshing WebApplicationContext for namespace 'example-servlet': startup date [Tue Jul 09 18:58:51 CST 2019]; root of context hierarchy
2019-7-9 18:58:51 org.springframework.beans.factory.xml.XmlBeanDefinitionReader loadBeanDefinitions
信息: Loading XML bean definitions from ServletContext resource [/WEB-INF/example-servlet.xml]
2019-7-9 18:58:52 org.springframework.beans.factory.support.DefaultListableBeanFactory preInstantiateSingletons
信息: Pre-instantiating singletons in org.springframework.beans.factory.support.DefaultListableBeanFactory@98062f: defining beans [servletContextAwareBean]; root of factory hierarchy
2019-7-9 18:58:52 org.springframework.web.servlet.handler.AbstractUrlHandlerMapping registerHandler
信息: Mapped URL path [/example/helloWorld] onto handler 'servletContextAwareBean'
2019-7-9 18:58:52 org.springframework.web.servlet.handler.AbstractUrlHandlerMapping registerHandler
信息: Mapped URL path [/example/helloWorld.*] onto handler 'servletContextAwareBean'
2019-7-9 18:58:52 org.springframework.web.servlet.handler.AbstractUrlHandlerMapping registerHandler
信息: Mapped URL path [/example/helloWorld/] onto handler 'servletContextAwareBean'
2019-7-9 18:58:53 org.springframework.web.servlet.FrameworkServlet initServletBean
```
从日志来看，这次确实已经正确实例化bean及识别到MappedURL信息了，但当访问：http://localhost:8080/springmvc3-analysis/example/helloWorld时仍然响应404，而控制台日志为：No mapping found for HTTP request with URI [/springmvc3-analysis/example/helloWorld] in DispatcherServlet with name 'example'
###### 调试分析
根据上面的日志，可发现AbstractUrlHandlerMapping类的registerHandler方法，通过方法调用链分析，核心位于方法DispatcherServlet.initStrategies-->DispatcherServlet.initHandlerMappings-->DispatcherServlet.getDefaultStrategies(context, HandlerMapping.class)
```language
@SuppressWarnings("unchecked")
	protected <T> List<T> getDefaultStrategies(ApplicationContext context, Class<T> strategyInterface) {
                //根据strategyInterface的值HandlerMapping.class获取完整类名org.springframework.web.servlet.HandlerMapping
		String key = strategyInterface.getName();
                //根据key取DispatcherServlet.properties文件对应的Properties对象获取配置项：org.springframework.web.servlet.handler.BeanNameUrlHandlerMapping,org.springframework.web.servlet.mvc.annotation.DefaultAnnotationHandlerMapping
		String value = defaultStrategies.getProperty(key);
		if (value != null) {
                        //类名转数组
			String[] classNames = StringUtils.commaDelimitedListToStringArray(value);
			List<T> strategies = new ArrayList<T>(classNames.length);
			for (String className : classNames) {
				try {
                                        //加载class获取clazz 对象
					Class<?> clazz = ClassUtils.forName(className, DispatcherServlet.class.getClassLoader());
                                        // context.getAutowireCapableBeanFactory().createBean(clazz)，即获取BeanFactory并实例化clazz 对应的实例对象（即Spring源码已分析的过程）
					Object strategy = createDefaultStrategy(context, clazz);
					strategies.add((T) strategy);
				}
				catch (ClassNotFoundException ex) {
                                  //省略异常处理
				}
			}
			return strategies;
		}
		else {
			return new LinkedList<T>();
		}
	}
```
- BeanNameUrlHandlerMapping、DefaultAnnotationHandlerMapping原始对象实例化就不做分析，重点分析原始对象实例化之后的处理过程。之前有分析过BeanPostProcessor接口有两个核心接口：postProcessBeforeInitialization、postProcessAfterInitialization用于实例对象初始化前后的处理，此处就涉及到ApplicationContextAwareProcessor，即判断当前实例是否有实现Aware接口若有则反射调用对应接口。
- BeanNameUrlHandlerMapping、DefaultAnnotationHandlerMapping拥有共同直接父类AbstractDetectingUrlHandlerMapping、间接共同接口ApplicationContextAware，那么根据分析BeanNameUrlHandlerMapping原始对象创建之后即被反射调用其setApplicationContext(ApplicationContext context):
```language
        //BeanNameUrlHandlerMapping(WebApplicationObjectSupport)类setApplicationContext方法
	public final void setApplicationContext(ApplicationContext context) throws BeansException {
		if (context == null && !isContextRequired()) {
			// Reset internal context state.
			this.applicationContext = null;
			this.messageSourceAccessor = null;
		}
		else if (this.applicationContext == null) {
			//省略N行
                        //设置当前HandlerMapping实例对应的context
			this.applicationContext = context;
			this.messageSourceAccessor = new MessageSourceAccessor(context);
                        //通过WebApplicationObjectSupport.initApplicationContext-->AbstractDetectingUrlHandlerMapping.initApplicationContext()-->AbstractDetectingUrlHandlerMapping.detectHandlers()（重点）
			initApplicationContext(context);
		}
		else {
			//省略N行
		}
	}
```
AbstractDetectingUrlHandlerMapping.detectHandlers()方法从名称即可判断是检测Hanlder：
```language

	protected void detectHandlers() throws BeansException {
		//此处detectHandlersInAncestorContexts为false，从代码来看是获取所有的beanNames集合供遍历         
		String[] beanNames = (this.detectHandlersInAncestorContexts ?
				BeanFactoryUtils.beanNamesForTypeIncludingAncestors(getApplicationContext(), Object.class) :
				getApplicationContext().getBeanNamesForType(Object.class));

		// Take any bean name that we can determine URLs for.
		for (String beanName : beanNames) {
                        //此处AbstractDetectingUrlHandlerMapping的determineUrlsForHandler为abstract需要子类自行实现，即根据beanName获取可处理请求的urls列表(BeanNameUrlHandlerMapping、DefaultAnnotationHandlerMapping有不同的实现)
			String[] urls = determineUrlsForHandler(beanName);
			if (!ObjectUtils.isEmpty(urls)) {
				// URL paths found: Let's consider it a handler.
                                /此处AbstractDetectingUrlHandlerMapping的registerHandler，会调用父类该方法，即类似以/example/helloWorld.*为key，beanName对应的class为value存储至HandlerMapping实例对应的handlerMap（BeanNameUrlHandlerMapping、DefaultAnnotationHandlerMapping各有自己的实例变量：handlerMap）
				registerHandler(urls, beanName);
			}
			else {}
		}
	}
```
BeanNameUrlHandlerMapping类determineUrlsForHandler方法（从源码来看beanName或者alias需以/开头）
```language
	@Override
	protected String[] determineUrlsForHandler(String beanName) {
		List<String> urls = new ArrayList<String>();
		if (beanName.startsWith("/")) {
			urls.add(beanName);
		}
		String[] aliases = getApplicationContext().getAliases(beanName);
		for (String alias : aliases) {
			if (alias.startsWith("/")) {
				urls.add(alias);
			}
		}
		return StringUtils.toStringArray(urls);
	}
```
DefaultAnnotationHandlerMapping类determineUrlsForHandler方法（beanName为servletContextAwareBean）：
```language
	protected String[] determineUrlsForHandler(String beanName) {
		ApplicationContext context = getApplicationContext();
                //通过beanName为servletContextAwareBean即可获得ExampleController（原因？稍后分析，哈哈，是自己傻了），handlerType 对应class com.aoe.demo.springmvc.ExampleController
		Class<?> handlerType = context.getType(beanName);
                //获取bean（即class）对应RequestMapping的数据封装为RequestMapping
		RequestMapping mapping = context.findAnnotationOnBean(beanName, RequestMapping.class);
		if (mapping != null) {
			// @RequestMapping found at type level
			this.cachedMappings.put(handlerType, mapping);
			Set<String> urls = new LinkedHashSet<String>();
                        //结果为：[/example]
			String[] typeLevelPatterns = mapping.value();
			if (typeLevelPatterns.length > 0) {
				// @RequestMapping specifies paths at type level
                                //判断是方法级别的RequestMapping配置信息，根据配置组装urls，即会根据（如若配置的url不是./斜线结尾且开启默认后续则会自动生成类似3个URL地址（[/helloWorld, /helloWorld.*, /helloWorld/]））
				String[] methodLevelPatterns = determineUrlsForHandlerMethods(handlerType, true);
				for (String typeLevelPattern : typeLevelPatterns) {
					if (!typeLevelPattern.startsWith("/")) {
						typeLevelPattern = "/" + typeLevelPattern;
					}
					boolean hasEmptyMethodLevelMappings = false;
					for (String methodLevelPattern : methodLevelPatterns) {
						if (methodLevelPattern == null) {
							hasEmptyMethodLevelMappings = true;
						}
						else {
                                                        //封装class类的RequestMapping及Methods的RequestMapping成完整的url列表（[/example/helloWorld, /example/helloWorld.*, /example/helloWorld/]）
							String combinedPattern = getPathMatcher().combine(typeLevelPattern, methodLevelPattern);
                                                         addUrlsForPath(urls, combinedPattern);
						}
					}
  					if (hasEmptyMethodLevelMappings ||
							org.springframework.web.servlet.mvc.Controller.class.isAssignableFrom(handlerType)) {
						addUrlsForPath(urls, typeLevelPattern);
					}
				}
				return StringUtils.toStringArray(urls);
			}
			else {
				// actual paths specified by @RequestMapping at method level
				return determineUrlsForHandlerMethods(handlerType, false);
			}
		}
		else if (AnnotationUtils.findAnnotation(handlerType, Controller.class) != null) {
			/class无RequestMapping仅方法上包含RequestMapping注解 
			return determineUrlsForHandlerMethods(handlerType, false);
		}
		else {
			return null;
		}
	}
```
因由于是示例是基于@RequestMapping注解方式实现的url映射，故仅DefaultAnnotationHandlerMapping有获取到url配置并添加到其handlerMap。在分析请求是为何无法匹配到方法返回404之前，还有两个问题待解答。
###### ExampleControler与servletContextAwareBean关系
- 在源码传入servletContextAwareBean调用Class<?> handlerType = context.getType(beanName)时，却可获取到- ExampleControler的class对象。
- 经过层层分析，为啥呢？想呵呵，很简单因为xml配置ExampleControler时复制的忘了修改指定的id就是rvletContextAwareBean；导致xml解析bean标签注册BeanDefinition时ExampleControler对应的beanName就是这个。哈哈，傻了...
###### BeanNameUrlHandlerMapping是如何解析生成handlerMap？
上面已经分析了基于注解方式的时，DefaultAnnotationHandlerMapping会依次解析class、method方法上的@RequestMapping注解，生成url并将url与ExampleControler.class映射保存至其handlerMap；而BeanNameUrlHandlerMapping呢？如其如应该就是beanName名称url处理，直接盾BeanNameUrlHandlerMapping源码注释：
```language
 The mapping is from URL to bean name. Thus an incoming URL "/foo" would map
 * to a handler named "/foo", or to "/foo /foo2" in case of multiple mappings to
 * a single handler. Note: In XML definitions, you'll need to use an alias
 * name="/foo" in the bean definition, as the XML id may not contain slashes
```
相对就比较好理解，即在example-servelt.xml修改为（注意此处为手动注入bean,暂不要配置context:component-scan）：
```language
	<bean id="exampleController"  name="/exampleController"  class="com.aoe.demo.springmvc.ExampleController"/>
```
日志为：
```language
2019-7-10 18:17:29 org.springframework.web.servlet.handler.AbstractUrlHandlerMapping registerHandler
信息: Mapped URL path [/exampleController] onto handler 'exampleController'
2019-7-10 18:17:29 org.springframework.web.servlet.handler.AbstractUrlHandlerMapping registerHandler
信息: Mapped URL path [/example/helloWorld] onto handler 'exampleController'
2019-7-10 18:17:29 org.springframework.web.servlet.handler.AbstractUrlHandlerMapping registerHandler
信息: Mapped URL path [/example/helloWorld.*] onto handler 'exampleController'
2019-7-10 18:17:29 org.springframework.web.servlet.handler.AbstractUrlHandlerMapping registerHandler
信息: Mapped URL path [/example/helloWorld/] onto handler 'exampleController'
```
###### 补充知识点
1. 当spring的xml配置文件中，手动注入bean（xml的bean标签： name="/exampleController"）与context:component-scan(且class需有@Controller(value="/exampleController2")同时配置时会xml的顺序优先取前面对应的name值作为其容器中bean的beanNam
2. 同1如若同时使用注解扫描及手动bean注入时，因beanName不同则会实例化两个bean对象至spring容器（但如若xml与@Controller两个地方均配置指定的beanName，则会使用后面解析的BeanDefinition替换掉先解析生成的BeanDefinition，即只会实例化该beanName最终对应的BeanDefinition对应的实例）；而AbstractUrlHandlerMapping分析url配置时基于bean识别，由于同一个class两个bean则会识别两次。若代码如：
```language
@Controller(value="/exampleController2")
@RequestMapping("/example")
public class ExampleController {

    @RequestMapping("/helloWorld")
    @ResponseBody
    public String helloWorld(Model model) {
        model.addAttribute("message", "Hello World!");
        return "helloWorld";
    }
}
```
```language
	<context:component-scan base-package="com.aoe.demo.springmvc"/>
	<bean id="exampleController1"  name="/exampleController"  class="com.aoe.demo.springmvc.ExampleController"/>
```
若如该示例运行时会异常（摘取）：
```language
信息: Pre-instantiating singletons in org.springframework.beans.factory.support.DefaultListableBeanFactory@172290f: defining beans [/exampleController2,org.springframework.context.annotation.internalConfigurationAnnotationProcessor,org.springframework.context.annotation.internalAutowiredAnnotationProcessor,org.springframework.context.annotation.internalRequiredAnnotationProcessor,org.springframework.context.annotation.internalCommonAnnotationProcessor,exampleController1,org.springframework.context.annotation.ConfigurationClassPostProcessor$ImportAwareBeanPostProcessor#0]; root of factory hierarchy
2019-7-11 9:22:28 org.springframework.web.servlet.handler.AbstractUrlHandlerMapping registerHandler
信息: Mapped URL path [/exampleController2] onto handler '/exampleController2'
2019-7-11 9:22:28 org.springframework.web.servlet.handler.AbstractUrlHandlerMapping registerHandler
信息: Mapped URL path [/exampleController] onto handler 'exampleController1'
2019-7-11 9:22:28 org.springframework.web.servlet.handler.AbstractUrlHandlerMapping registerHandler
信息: Mapped URL path [/example/helloWorld] onto handler '/exampleController2'
2019-7-11 9:22:28 org.springframework.web.servlet.handler.AbstractUrlHandlerMapping registerHandler
信息: Mapped URL path [/example/helloWorld.*] onto handler '/exampleController2'
2019-7-11 9:22:28 org.springframework.web.servlet.handler.AbstractUrlHandlerMapping registerHandler
信息: Mapped URL path [/example/helloWorld/] onto handler '/exampleController2'
Error creating bean with name 'org.springframework.web.servlet.mvc.annotation.DefaultAnnotationHandlerMapping': Initialization of bean failed; nested exception is java.lang.IllegalStateException: Cannot map handler 'exampleController1' to URL path [/example/helloWorld]: There is already handler of type [class com.aoe.demo.springmvc.ExampleController] mapped.
```
原因呢？做如下几点解答：
1. 之前说的xml中标签是按顺序解析，这个地方按顺序的只是解析生成BeanDefinition是按明的当两该顺序；所以就会有上面说的出现beanName重复进，会用后解析生成的BeanDefinition替换掉先解析生成的BeanDefitinion(如日志：Overriding bean definition for bean 'exampleController1': replacing [Generic bean: class [com.aoe.demo.springmvc.ExampleController]）。而bean的实例化则是beanDefinitionNames List的顺序（即按解析到beanName的先后顺序）
2. 如若xml中bean标签与@Controller同时配置，而beanName不一样，会分别在Spring容器中实例化两个不同的实例与不同的beanName对应。
3. xml中bean标签与@Controller同时且beanName不一致时，为何会提示handler 已经被映射（只是在解析RequestMapping标签时才会报错） ，解析时会获取Spring容器中所有的beanName表现（顺序同上面的beanDefinitionNames List）：
   1. BeanNameUrlHandlerMapping逻辑相对简单，即判断beanName是否/开头，若是则以beanName为key，ExampleController.class为value存入至BeanNameUrlHandlerMapping实例对应的hanlderMap。存入hanlderMap之前会先使用key检查是否已存在，而beanName不两只故检查通过；
   2. DefaultAnnotationHandlerMapping则是解析class、method对应的RequestMapping注解生成url，然后以url为keyExampleController.class为value存入至DefaultAnnotationHandlerMapping实例对应hanlderMap。同样，存入hanlderMap之前会先使用key检查是否已存在，而由于上面讲的同一class因beanName不同容器中存在两个对象，所以解析到第2个对象的RequestMapping注解时，生成的url之前存入到hanlderMap中了，此次存入之前的检查不通过，故会报上面的错误。
- 完成BeanNameUrlHandlerMapping、DefaultAnnotationHandlerMapping基于beanName或者RequestMapping解析生成hanlder之后，会将BeanNameUrlHandlerMapping、DefaultAnnotationHandlerMapping对应的两个实例对象赋值给DispatcherServlet实例，而后续http请求则会分别调用这两个实例进行url匹配与分析（先BeanNameUrlHandlerMapping匹配，后匹配DefaultAnnotationHandlerMapping）

##### 4.1.3 http请求404问题
1. 既然上面已经明确了xml中bean标签与@Controller同时存在问题，注释xml的bean的手动注入使用注解方式来分析该问题。之前有分析过，http请求分发的入口源码在DispatcherServlet(FrameworkServlet).processRequest(HttpServletRequest, HttpServletResponse) 。
- 基于http://localhost:8080/springmvc3-analysis/example/helloWorld 访问时始终报404，经调试分析在匹配url之分几点：1.判断fullPath是否包含应用名（有可能是/）并处理；2.使用url匹配servlet配置（即对应wb.xml的servlet配置（<url-pattern>/example/*</url-pattern>）并生成servletName（会从url中把web.xml中可匹配的前缀去除但保留前面的/）即为/helloWorld；3.使用servletName即/helloWorld去hanlderMap获取获取Hanlder实例（DefaultAnnotationHandlerMapping）...额，根据之前的分析/helloWorld确实匹配不到。那就知道怎么确认分析是否正确了，访问：http://localhost:8080/springmvc3-analysis/example/example/helloWorld 就OK了，既然这样就知道怎么处理了。
2. 由于示例有配置@Controller(value="/exampleController")，故也顺便看看http://localhost:8080/springmvc3-analysis/exampleController 结果怎样。页面仍是404，而后台有日志：
```language
2019-7-11 18:14:15 org.springframework.web.servlet.mvc.support.DefaultHandlerExceptionResolver handleNoSuchRequestHandlingMethod
警告: No matching handler method found for servlet request: path '/exampleController', method 'GET', parameters map[[empty]]
```
- 从日志来看，即未找到RequestHandlingMethod。通过调试发现，基于BeanNameUrlHandlerMapping会先根据url前面部分（/exampleController）获取Hanlder（实现与上面分析相似），但在获取到Hanlder之后，则会再次获取HandlerAdapter实例ha，最终调用 ha.handle(processedRequest, response, mappedHandler.getHandler())完成实际调用并返回ModelAndView实例（ha这点儿点即使上面的基于DefaultAnnotationHandlerMapping流程也类似）。补充一点哈，调用 ha.handle之前也会根据调用Interceptor）



##### 补充知识点
- 


















