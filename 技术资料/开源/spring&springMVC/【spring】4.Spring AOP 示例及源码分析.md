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
2. AopConfigUtils.forceAutoProxyCreatorToUseClassProxying(registry)、AopConfigUtils.forceAutoProxyCreatorToExposeProxy(registry)即解析是否有

