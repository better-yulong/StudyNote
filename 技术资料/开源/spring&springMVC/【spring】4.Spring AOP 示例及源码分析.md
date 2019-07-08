说到AOP，其实自然会与<aop:config>或@Aspect有关，首先理解下AOP的概念及AOP的示例。
### 一.AOP术语
Spring AOP的标准术语，其实是比较晦涩难懂的，这是网上摘的相对比较容易理解一种定义。
#### 通知、增强处理（Advice）
就是你想要的功能，也就是上说的安全、事物、日子等。你给先定义好，然后再想用的地方用一下。包含Aspect的一段处理代码
#### 连接点（JoinPoint）
这个就更好解释了，就是spring允许你是通知（Advice）的地方，那可就真多了，基本每个方法的钱、后（两者都有也行），或抛出异常是时都可以是连接点，spring只支持方法连接点。其他如AspectJ还可以让你在构造器或属性注入时都行，不过那不是咱们关注的，只要记住，和方法有关的前前后后都是连接点。
#### 切入点（Pointcut）
上面说的连接点的基础上，来定义切入点，你的一个类里，有15个方法，那就有十几个连接点了对吧，但是你并不想在所有方法附件都使用通知（使用叫织入，下面再说），你只是想让其中几个，在调用这几个方法之前、之后或者抛出异常时干点什么，那么就用切入点来定义这几个方法，让切点来筛选连接点，选中那几个你想要的方法。
#### 切面（Aspect）
切面是通知和切入点的结合。现在发现了吧，没连接点什么事，链接点就是为了让你好理解切点搞出来的，明白这个概念就行了。通知说明了干什么和什么时候干（什么时候通过方法名中的befor，after，around等就能知道），二切入点说明了在哪干（指定到底是哪个方法），这就是一个完整的切面定义。
#### 引入（introduction）
允许我们向现有的类添加新方法属性。这不就是把切面（也就是新方法属性：通知定义的）用到目标类中吗
#### 目标（target）
引入中所提到的目标类，也就是要被通知的对象，也就是真正的业务逻辑，他可以在毫不知情的情况下，被咋们织入切面。二自己专注于业务本身的逻辑。
#### 代理（proxy）
怎么实现整套AOP机制的，都是通过代理，这个一会儿给细说（可参考：https://www.cnblogs.com/lcngu/p/5339555.html、https://www.jianshu.com/p/5004b0b48511）
#### 织入（weaving）
把切面应用到目标对象来创建新的代理对象的过程。有三种方式，spring采用的是运行时，为什么是运行时，在上一文《Spring AOP开发漫谈之初探AOP及AspectJ的用法》中第二个标提到。
#### 目标对象
项目原始的Java组件。
#### AOP代理
由AOP框架生成java对象。
#### AOP代理方法 
advice +　目标对象的方法。

### 二.AOP示例
AOP的配置类似于bean，可xml配置亦可通过注解方式配置
#### 2.1 xml配置 
近期较少使用，只记得需要在xml中使用aop相关的会签，于是乎在spring源码中根据aop关键字查找xml文件AopNamespaceHandlerEventTests-context.xml并及关联java类：
AopNamespaceHandlerTests，结合示例代码，进行示例编码
1. spring相关的jar之前已完成pom.xml的配置，但仍需增加：
```language
		<dependency>
			<groupId>org.aspectj</groupId>
			<artifactId>aspectjweaver</artifactId>
			<version>1.6.8</version>
		</dependency>
```
2. 新增业务类及切换拦截类
```language
//之后会针对init方法做AOP验证
@Repository
public class BizAspectjExample {
	
	@PostConstruct
	public void init(){
		System.out.println("BizExample init...");
	}
}
```
```language
//参考AopNamespaceHandlerTests类
//非注解的切面类，采用xml配置
public class AspectjExample {
	
	public void before(){
		System.out.println("AspectjExample before...");
	}
	
	public void after(){
		System.out.println("AspectjExample after...");
	}
        //这个地方比较关键，因为是around方式增强
	public void around(ProceedingJoinPoint pjp) throws Throwable {
		System.out.println("AspectjExample around before...");
		pjp.proceed();
		System.out.println("AspectjExample around after...");
	}

}
```
3. xml配置
需在applicationContext.xml文件头配置命名空间及xsd文件路径，参考AopNamespaceHandlerEventTests-context.xml
```language
	<aop:config>
		<aop:aspect id="bizAspectjExampleAop" ref="aspectjExample">
			<aop:pointcut id="pc" expression="execution(* *..BizAspectjExample.*(..))"/>
			<aop:before pointcut-ref="pc" method="before" />
			<aop:after pointcut-ref="pc" method="after" />
			<!-- <aop:after-returning pointcut-ref="pc" method="myAfterReturningAdvice" returning="age"/>
			<aop:after-throwing pointcut-ref="pc" method="myAfterThrowingAdvice" throwing="ex"/> -->
			<aop:around pointcut-ref="pc" method="around"/>
		</aop:aspect>
	</aop:config>

	<bean name="aspectjExample" class="com.aoe.demo.aop.AspectjExample"></bean>
```
然而启动后报错，如下：
```严重: Context initialization failed
org.springframework.beans.factory.BeanCreationException: Error creating bean with name 'bizAspectjExample' defined in file [D:\work\webcontainer\tomcat7\webapps\spring3-analysis\WEB-INF\classes\com\aoe\demo\aop\BizAspectjExample.class]: Initialization of bean failed; nested exception is org.springframework.aop.framework.AopConfigException: Cannot proxy target class because CGLIB2 is not available. Add CGLIB to the class path or specify proxy interfaces.
	at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.doCreateBean(AbstractAutowireCapableBeanFactory.java:527)
	at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.createBean(AbstractAutowireCapableBeanFactory.java:456)
	at org.springframework.beans.factory.support.AbstractBeanFactory$1.getObject(AbstractBeanFactory.java:294)

```
既然报错，那就分析下原因，看看如何解决

### 三.AOP解析及初始化分析
有了之前分析context:component-scan的经验，就知道如何分析了（如若不明白可查看该系列笔记2）。根据xml找到AOP标签处理对应的自定义命名空间处理器为AopNamespaceHandler，而config子标签对应的解析器为ConfigBeanDefinitionParser，根据之前的经验核心在parse方法：
```language
	public BeanDefinition parse(Element element, ParserContext parserContext) {
		CompositeComponentDefinition compositeDef =
				new CompositeComponentDefinition(element.getTagName(), parserContext.extractSource(element));
		parserContext.pushContainingComponent(compositeDef);

		configureAutoProxyCreator(parserContext, element);

		List<Element> childElts = DomUtils.getChildElements(element);
		for (Element elt: childElts) {
			String localName = parserContext.getDelegate().getLocalName(elt);
			if (POINTCUT.equals(localName)) {
				parsePointcut(elt, parserContext);
			}
			else if (ADVISOR.equals(localName)) {
				parseAdvisor(elt, parserContext);
			}
			else if (ASPECT.equals(localName)) {
				parseAspect(elt, parserContext);
			}
		}

		parserContext.popAndRegisterContainingComponent();
		return null;
	}
```
#### 3.1 解析前准备configureAutoProxyCreator(parserContext, element)
主要有3点：
1. AopConfigUtils.registerOrEscalateApcAsRequired(AspectJAwareAdvisorAutoProxyCreator.class, registry, source),主要是将创建AspectJAwareAdvisorAutoProxyCreator的BeanDefinition并注册到registry（有点眼熟哈，后面会讲）
2. AopConfigUtils.forceAutoProxyCreatorToUseClassProxying(registry)、AopConfigUtils.forceAutoProxyCreatorToExposeProxy(registry)即解析是否有配置proxy-target-class、expose-proxy，有则根据值并将其作为新属性添加到步骤1对应的AspectJAwareAdvisorAutoProxyCreator的BeanDefinition（definition.getPropertyValues().add("proxyTargetClass", Boolean.TRUE);）
3. 封装AspectJAwareAdvisorAutoProxyCreator的BeanDefinition为BeanComponentDefinition，并设置到parserContext
#### 3.2 解析aop:aspect及其子标签
根据aop:pointcut、aop:before、aop:after、aop:around创建BeanDefintion并注册至register
```	
     private Class getAdviceClass(Element adviceElement, ParserContext parserContext) {
		String elementName = parserContext.getDelegate().getLocalName(adviceElement);
		if (BEFORE.equals(elementName)) {
			return AspectJMethodBeforeAdvice.class;
		}
		else if (AFTER.equals(elementName)) {
			return AspectJAfterAdvice.class;
		}
		else if (AFTER_RETURNING_ELEMENT.equals(elementName)) {
			return AspectJAfterReturningAdvice.class;
		}
		else if (AFTER_THROWING_ELEMENT.equals(elementName)) {
			return AspectJAfterThrowingAdvice.class;
		}
		else if (AROUND.equals(elementName)) {
			return AspectJAroundAdvice.class;
		}
		else {
			throw new IllegalArgumentException("Unknown advice kind [" + elementName + "].");
		}
	}
```
即根据标签，生成不同class的BeanDefinition（还包括AspectJExpressionPointcut、AspectJPointcutAdvisor）等。
#### 3.3 AOP代理bean生成
结合之前分析的经验及AspectJAwareAdvisorAutoProxyCreator（其有父接口：BeanPostProcessor），即在初始bean原始对象的创建后，会调用到AspectJAwareAdvisorAutoProxyCreator（AbstractAutoProxyCreator）的postProcessAfterInitialization方法，根据日志及调试，异常由createAopProxy方法抛出，从代码来看此处有两种生成代理对象的方式：JdkDynamicAopProxy、CglibProxyFactory.createCglibProxy；而报错来看则是并没有在class apth中配置CGLIB的支持，而此处也希望走JdkDynamic；通过调试分析，之所以走到该分支，是因为hasNoUserSuppliedProxyInterfaces获取到的interfaces列表为null.

```language
	public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
		if (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config)) {
			Class targetClass = config.getTargetClass();
			if (targetClass == null) {
				throw new AopConfigException("TargetSource cannot determine target class: " +
						"Either an interface or a target is required for proxy creation.");
			}
			if (targetClass.isInterface()) {
				return new JdkDynamicAopProxy(config);
			}
			if (!cglibAvailable) {
				throw new AopConfigException(
						"Cannot proxy target class because CGLIB2 is not available. " +
						"Add CGLIB to the class path or specify proxy interfaces.");
			}
			return CglibProxyFactory.createCglibProxy(config);
		}
		else {
			return new JdkDynamicAopProxy(config);
		}
	}
```
```language
    private boolean hasNoUserSuppliedProxyInterfaces(AdvisedSupport config) {
		Class[] interfaces = config.getProxiedInterfaces();
		return (interfaces.length == 0 || (interfaces.length == 1 && 
    SpringProxy.class.equals(interfaces[0])));
	}
```
从源码来看，而config.getProxiedInterfaces实际获取的就是proxyFactory对应的Interface列表
```language
		if (!shouldProxyTargetClass(beanClass, beanName)) {
			// Must allow for introductions; can't just set interfaces to
			// the target's interfaces only.
			Class<?>[] targetInterfaces = ClassUtils.getAllInterfacesForClass(beanClass, this.proxyClassLoader);
			for (Class<?> targetInterface : targetInterfaces) {
				proxyFactory.addInterface(targetInterface);
			}
		}
```
经调试,实际上在的代码就是获取被代理类的接口列表并添加到proxyFactory,开始不太理解原因；不过根据之前静态代理、动态的分析来看，动态代理需基于接口而之前的报错日志，翻译应该包含两个信息：即class path缺少CGLIB或者无特定接口，初步推断是需要基于接口来实现AOP。
##### 3.3.0 AOP bean实例化
1. JdkDynamicAopProxy底层仍是使用熟悉的方式（Proxy.newProxyInstance(classLoader, proxiedInterfaces, this)）来实现动态，其实this即之前new的JdkDynamicAopProxy实例（而JdkDynamicAopProxy则InvocationHandler的实现类）。
2. Processor的postProcessAfterInitialization方法是在创建bean的原始对象之后调用各Processor进行封装（或生成代理），而返回给容器的则是封装之后的对象，即基于AOP时，Spring容器中beanName对应的实际存储对象为生成的动态代理实例proxy。
既然上面的分析到了，那下面就验证下分析结果哈
##### 3.3.1 基于接口实现AOP 
###### 3.3.1.1 被代理类业务接口
```language
    public interface BizAspectjExampleInterf {
	void init();
    }
```
###### 3.3.1.2 被代理类业务实现类
```
@Repository
public class BizAspectjExample implements BizAspectjExampleInterf {
	
	/* (non-Javadoc)
	 * @see com.aoe.demo.aop.BizAspectjExampleInterf#init()
	 */
	@PostConstruct
	public void init(){
		System.out.println("BizAspectjExample init...");
	}
}

```
###### 3.3.1.3 applicationContext.xml配置
```language

<!-- 未基于接口
	<aop:config>
		<aop:aspect id="bizAspectjExampleAop" ref="aspectjExample">
			<aop:pointcut id="pc" expression="execution(* *..BizAspectjExample.*(..))"/>
			<aop:before pointcut-ref="pc" method="before" />
			<aop:after pointcut-ref="pc" method="after" />
			<aop:around pointcut-ref="pc" method="around"/>
		</aop:aspect>
	</aop:config>
	 -->
	
	<aop:config>
		<aop:aspect id="bizAspectjExampleAop" ref="aspectjExample">
		    <!-- 将基于实现类BizAspectjExample修改为接口BizAspectjExampleInterf ; BizAspectjExample是BizAspectjExampleInterf的实现类  -->
			<aop:pointcut id="pc" expression="execution(* *..BizAspectjExampleInterf.*(..))"/>
			<aop:before pointcut-ref="pc" method="before" />
			<aop:after pointcut-ref="pc" method="after" />
			<aop:around pointcut-ref="pc" method="around"/>
		</aop:aspect>
	</aop:config>

	<bean name="aspectjExample" class="com.aoe.demo.aop.AspectjExample"></bean>
```
启动应用成功，但日志仅打印了BizAspectjExample的init方法自身的日志，而并未打印AspectjExample拦截期望打印的日志，之后继续分析发现：AOP是基于Processor生成原始实例的动态代理bean，而原始实例创建时会调用init方法，即niit方法已经在之前执行了。那就相对简单了，定义新方法，结合之前LazyInitTestLinstener示例在容器初始化结束之后获取bean并调用对应方法。
```language
public interface BizAspectjExampleInterf {

	void biz();

}
```
```language
@Repository
public class BizAspectjExample implements BizAspectjExampleInterf {
	
	/* (non-Javadoc)
	 * @see com.aoe.demo.aop.BizAspectjExampleInterf#init()
	 */
	@PostConstruct
	public void init(){
		System.out.println("BizAspectjExample init...");
	}
	
	public void biz(){
		System.out.println("BizAspectjExample biz...");
	}
}
```
```language
//LazyInitTestLinstener
@Component
public class LazyInitTestLinstener implements ApplicationListener,ApplicationContextAware {
	
	@Autowired
	private ApplicationContext applicationContext;

	public void onApplicationEvent(ApplicationEvent event) {
		if(event instanceof ContextRefreshedEvent){
			System.out.println("LazyInitTestLinstener --> ContextRefreshedEvent");
			//LazyInitBeanExample 基于@Lazy注解使得在前面的bean初始化时并不会实例化，而里面待首次使用时才会实例化（但如果通过@Autowired、@Rresource也会在前面在注解处理时实例化。
			LazyInitBeanExample lazyInitBeanExample  = applicationContext.getBean(LazyInitBeanExample.class);
			System.out.println("lazyInitBeanExample:" + lazyInitBeanExample.hashCode());
			
			//基于类获取bean会无法获取
			//BizAspectjExample bizAspectjExample  = (BizAspectjExample) applicationContext.getBean(BizAspectjExample.class);
			//通过接口、beanName可以正常获取到bean
			BizAspectjExampleInterf bizAspectjExample  = (BizAspectjExampleInterf) applicationContext.getBean(BizAspectjExampleInterf.class);
			//BizAspectjExampleInterf bizAspectjExample  = (BizAspectjExampleInterf) applicationContext.getBean("bizAspectjExample");
			bizAspectjExample.biz();
		}
		
	}
```
启动成功，通过日志可发现已正常的实现AOP拦截：
```language
AspectjExample before...
AspectjExample around before...
BizAspectjExample biz...
AspectjExample after...
AspectjExample around after...
2019-7-5 16:48:44 org.springframework.web.context.ContextLoader initWebApplicationContext
```
###### 3.3.1.4 补充知识点
在上面LazyInitTestLinstener 中通过applicationContext.getBean()即期获取普通的bean（如LazyInitBeanExample）可直接通过类名；但是即若是基于AOP的代理，则只能通过接口或者beanName才可获取到Spring容器中的bean。

### 四、后续
计划调整，暂时分析到这儿，至于基于注解的AOP配置认为应该类似于@Aotuwrie注解的注入原理类似,

- 参考资料：
  1. https://www.cnblogs.com/junzi2099/p/8274813.html