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
启动应用，中单有个小插曲，启动时始终报ClassNotFoundException，即ExampleController.class始终找不到，去应用的target/classes目录确实找不到class文件，结合eclipse的Problems发现系JavaBuild配置报错，调整后clean





























