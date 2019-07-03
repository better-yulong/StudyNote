### 一.Spring Aware接口
Spring Aware的目的是为了让bean获取spring容器的服务，即实现Aware接口后，容器在bean实例化时会自动将容器本身的服务作为参参调用特定方法：
  - BeanNameAware ：可以获取容器中bean的名称
  - BeanFactoryAware:获取当前bean factory这也可以调用容器的服务
  - ApplicationContextAware： 当前的applicationContext， 这也可以调用容器的服务
  - MessageSourceAware：获得message source，这也可以获得文本信息
  - applicationEventPulisherAware：应用事件发布器，可以发布事件，
  - ResourceLoaderAware： 获得资源加载器，可以获得外部资源文件的内容；
#### 1.1 ApplicationContextAware示例
```
@Component
public class ApplicationContextAwareExample implements ApplicationContextAware {
	
	private String name ;
	
	private ApplicationContext applicationContext ;

	public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
		this.applicationContext = applicationContext ;
	}
	
	@PostConstruct
	public void init(){
		System.out.println("ApplicationContextAwareExample  name:" + this.name);
		System.out.println("ApplicationContextAwareExample  StartupDate:" + this.applicationContext.getStartupDate());
	}

}

```
应用启动日志：
```language
ApplicationContextAwareExample  DisplayName:Root WebApplicationContext
ApplicationContextAwareExample  StartupDate:1562129008373
```
至于原理呢？就是上一节分析的Spring包含了非常多的Processor，而其中ApplicationContextAwareProcessor则正好是对实现Aware相关接口bean实例的个性化处理。ApplicationContextAwareProcessor源码：
```language
        //ApplicationContextAwareProcessor源码
	private void invokeAwareInterfaces(Object bean) {
		if (bean instanceof Aware) {
			if (bean instanceof EnvironmentAware) {
				((EnvironmentAware) bean).setEnvironment(this.applicationContext.getEnvironment());
			}
			if (bean instanceof EmbeddedValueResolverAware) {
				((EmbeddedValueResolverAware) bean).setEmbeddedValueResolver(
						new EmbeddedValueResolver(this.applicationContext.getBeanFactory()));
			}
			if (bean instanceof ResourceLoaderAware) {
				((ResourceLoaderAware) bean).setResourceLoader(this.applicationContext);
			}
			if (bean instanceof ApplicationEventPublisherAware) {
				((ApplicationEventPublisherAware) bean).setApplicationEventPublisher(this.applicationContext);
			}
			if (bean instanceof MessageSourceAware) {
				((MessageSourceAware) bean).setMessageSource(this.applicationContext);
			}
                        //设置applicationContext
			if (bean instanceof ApplicationContextAware) {
				((ApplicationContextAware) bean).setApplicationContext(this.applicationContext);
			}
		}
	}
```

### 二.BeanPostProcessor接口
如果我们想在Spring容器中完成bean实例化、配置以及其他初始化方法前后要添加一些自己逻辑处理。我们需要定义一个或多个BeanPostProcessor接口实现类，然后注册到Spring IoC容器中。而从上面的分析来看，Processor由DefaultListableBeanFactory(AbstractBeanFactory).addBeanPostProcessor(BeanPostProcessor)添加（即beanFactory属性List<BeanPostProcessor> beanPostProcessors).
#### 1. prepareBeanFactory:即初始化prepareBeanFactory时会添加ApplicationContextAwareProcessor：该processor是对实现了Aware接口的bean的处理（如上分析）
#### 2. postProcessBeanFactory(beanFactory)：添加ServletContextAwareProcessor
#### 3. 通过调试即之前的分析还有如下Processor实例：
> [org.springframework.context.annotation.internalAutowiredAnnotationProcessor, org.springframework.context.annotation.internalRequiredAnnotationProcessor, org.springframework.context.annotation.internalCommonAnnotationProcessor, org.springframework.context.annotation.ConfigurationClassPostProcessor$ImportAwareBeanPostProcessor#0]
- 从这部分processp实例的名称即可知道，这部分processor与注解有关，那么根据之前的理解，Spring实例化之前会先行完成class对应的BeanDefinition注册：registry.registerBeanDefinition(beanName, definition)
##### 3.1 Processor的BeanDefinition注册
ClassPathBeanDefinitionScanner的scan(String... basePackages) 有两点：
1. 基于basePackages完成扫描并完成各class应对的BeanDefinition注册；
2. AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry); 完成注解配置Processor的BeanDefinition注册
```language
        //AnnotationConfigUtils
        	/**
	 * The bean name of the internally managed Configuration annotation processor.
	 */
	public static final String CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME =
			"org.springframework.context.annotation.internalConfigurationAnnotationProcessor";

	/**
	 * The bean name of the internally managed Autowired annotation processor.
	 */
	public static final String AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME =
			"org.springframework.context.annotation.internalAutowiredAnnotationProcessor";

	/**
	 * The bean name of the internally managed Required annotation processor.
	 */
	public static final String REQUIRED_ANNOTATION_PROCESSOR_BEAN_NAME =
			"org.springframework.context.annotation.internalRequiredAnnotationProcessor";

	/**
	 * The bean name of the internally managed JSR-250 annotation processor.
	 */
	public static final String COMMON_ANNOTATION_PROCESSOR_BEAN_NAME =
			"org.springframework.context.annotation.internalCommonAnnotationProcessor";

	public static Set<BeanDefinitionHolder> registerAnnotationConfigProcessors(
			BeanDefinitionRegistry registry, Object source) {

		Set<BeanDefinitionHolder> beanDefs = new LinkedHashSet<BeanDefinitionHolder>(4);

		if (!registry.containsBeanDefinition(CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME)) {
			RootBeanDefinition def = new RootBeanDefinition(ConfigurationClassPostProcessor.class);
			def.setSource(source);
			beanDefs.add(registerPostProcessor(registry, def, CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME));
		}

		if (!registry.containsBeanDefinition(AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME)) {
			RootBeanDefinition def = new RootBeanDefinition(AutowiredAnnotationBeanPostProcessor.class);
			def.setSource(source);
			beanDefs.add(registerPostProcessor(registry, def, AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME));
		}

		if (!registry.containsBeanDefinition(REQUIRED_ANNOTATION_PROCESSOR_BEAN_NAME)) {
			RootBeanDefinition def = new RootBeanDefinition(RequiredAnnotationBeanPostProcessor.class);
			def.setSource(source);
			beanDefs.add(registerPostProcessor(registry, def, REQUIRED_ANNOTATION_PROCESSOR_BEAN_NAME));
		}

		// Check for JSR-250 support, and if present add the CommonAnnotationBeanPostProcessor.
		if (jsr250Present && !registry.containsBeanDefinition(COMMON_ANNOTATION_PROCESSOR_BEAN_NAME)) {
			RootBeanDefinition def = new RootBeanDefinition(CommonAnnotationBeanPostProcessor.class);
			def.setSource(source);
			beanDefs.add(registerPostProcessor(registry, def, COMMON_ANNOTATION_PROCESSOR_BEAN_NAME));
		}

		// Check for JPA support, and if present add the PersistenceAnnotationBeanPostProcessor.
		 //省略JPA相关的Processor注册

		return beanDefs;
	}
```
通过如上对应关系：
> internalConfigurationAnnotationProcessor<-->ConfigurationClassPostProcessor
internalAutowiredAnnotationProcessor<-->AutowiredAnnotationBeanPostProcessor
internalRequiredAnnotationProcessor<-->RequiredAnnotationBeanPostProcessor
internalCommonAnnotationProcessor<-->CommonAnnotationBeanPostProcessor
##### 3.2 Processor的实例化
##### 3.2.1 ConfigurationClassPostProcessor实例化
DefaultListableBeanFactory.getBeansOfType(BeanDefinitionRegistryPostProcessor.class, true, false)
XmlWebApplicationContext(AbstractApplicationContext).invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory) line: 612	
##### 3.2.2 其他注解Process实例化
XmlWebApplicationContext(AbstractApplicationContext)类的refresh()会调用registerBeanPostProcessors(beanFactory)方法
```language
    internalAutowiredAnnotationProcessor<-->AutowiredAnnotationBeanPostProcessor
    internalRequiredAnnotationProcessor<-->RequiredAnnotationBeanPostProcessor
    internalCommonAnnotationProcessor<-->CommonAnnotationBeanPostProcessor
```
```language
         //AbstractApplicationContext类
	/**
	 * Instantiate and invoke all registered BeanPostProcessor beans,
	 * respecting explicit order if given.
	 * <p>Must be called before any instantiation of application beans.
	 */
	protected void registerBeanPostProcessors(ConfigurableListableBeanFactory beanFactory) {
                //即可获取internalAutowiredAnnotationProcessor、internalRequiredAnnotationProcessor、internalCommonAnnotationProcessor；该方法会遍历所有的beanDefinitionNames及BeanDefinition与BeanPostProcessor进行匹配
		String[] postProcessorNames = beanFactory.getBeanNamesForType(BeanPostProcessor.class, true, false);

		// Register BeanPostProcessorChecker that logs an info message when
		// a bean is created during BeanPostProcessor instantiation, i.e. when
		// a bean is not eligible for getting processed by all BeanPostProcessors.
		int beanProcessorTargetCount = beanFactory.getBeanPostProcessorCount() + 1 + postProcessorNames.length;
		beanFactory.addBeanPostProcessor(new BeanPostProcessorChecker(beanFactory, beanProcessorTargetCount));

		// Separate between BeanPostProcessors that implement PriorityOrdered,
		// Ordered, and the rest.
		List<BeanPostProcessor> priorityOrderedPostProcessors = new ArrayList<BeanPostProcessor>();
		List<BeanPostProcessor> internalPostProcessors = new ArrayList<BeanPostProcessor>();
		List<String> orderedPostProcessorNames = new ArrayList<String>();
		List<String> nonOrderedPostProcessorNames = new ArrayList<String>();
		for (String ppName : postProcessorNames) {
			if (isTypeMatch(ppName, PriorityOrdered.class)) {
                                //获取Processor实例（内部会调用doCreateBean方法创建实例）
				BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
				priorityOrderedPostProcessors.add(pp);
				if (pp instanceof MergedBeanDefinitionPostProcessor) {
					internalPostProcessors.add(pp);
				}
			}
			else if (isTypeMatch(ppName, Ordered.class)) {
				orderedPostProcessorNames.add(ppName);
			}
			else {
				nonOrderedPostProcessorNames.add(ppName);
			}
		}

		// First, register the BeanPostProcessors that implement PriorityOrdered.
		OrderComparator.sort(priorityOrderedPostProcessors);
		registerBeanPostProcessors(beanFactory, priorityOrderedPostProcessors);

		// Next, register the BeanPostProcessors that implement Ordered.
		List<BeanPostProcessor> orderedPostProcessors = new ArrayList<BeanPostProcessor>();
		for (String ppName : orderedPostProcessorNames) {
			BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
			orderedPostProcessors.add(pp);
			if (pp instanceof MergedBeanDefinitionPostProcessor) {
				internalPostProcessors.add(pp);
			}
		}
		OrderComparator.sort(orderedPostProcessors);
		registerBeanPostProcessors(beanFactory, orderedPostProcessors);

		// Now, register all regular BeanPostProcessors.
		List<BeanPostProcessor> nonOrderedPostProcessors = new ArrayList<BeanPostProcessor>();
		for (String ppName : nonOrderedPostProcessorNames) {
			BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
			nonOrderedPostProcessors.add(pp);
			if (pp instanceof MergedBeanDefinitionPostProcessor) {
				internalPostProcessors.add(pp);
			}
		}
		registerBeanPostProcessors(beanFactory, nonOrderedPostProcessors);

		// Finally, re-register all internal BeanPostProcessors.
		OrderComparator.sort(internalPostProcessors);
		//将上面实例化的3个Process实例添加至beanFactory对应的beanPostProcessors列表List
		registerBeanPostProcessors(beanFactory, internalPostProcessors);

		beanFactory.addBeanPostProcessor(new ApplicationListenerDetector());
	}
```
通过上在的分析，Process的Bean定义、实例化、调用逻辑基本清晰了，那么该如何实现自定义的BeanPostProcessor呢?

### 三.自定义Processor实践
#### 3.1个人理解，自定义Processor可关注这几点：
- 1.需实现BeanPostProcessor接口
- 2.BeanDefinition如何注册
    1. 可在xml基于bean标签配置，即Spring容器启动时会自动完成BeanDefinition注册及默认实例化单例；
    2. 基于注解实现BeanDefinition注册及默认实例化单例；
#### 3.2 基于framework-aoe-beans 实现自定义Processor
```language
@Component
public class AnnotationProcessorExample implements BeanPostProcessor, InitializingBean {

	public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
		System.out.println("AnnotationProcessorExample postProcessBeforeInitialization。。。");
		return null;
	}

	public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
		System.out.println("AnnotationProcessorExample postProcessAfterInitialization。。。");
		return null;
	}

	public void afterPropertiesSet() throws Exception {
		System.out.println("AnnotationProcessorExample afterPropertiesSet。。。");
	}

}
```
```language
public class SimpleProcessorExample implements BeanPostProcessor, InitializingBean {

	public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
		System.out.println("SimpleProcessorExample postProcessBeforeInitialization。。。");
		return null;
	}

	public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
		System.out.println("SimpleProcessorExample postProcessAfterInitialization。。。");
		return null;
	}

	public void afterPropertiesSet() throws Exception {
		System.out.println("SimpleProcessorExample afterPropertiesSet。。。");
	}

}
```
```language
<bean name="simpleProcessorExample" class="com.framework.aoe.beans.factory.support.process1.SimpleProcessorExample"></bean>
```
spring3-analysis引入framework-aoe-beans 工程后，运行发现抛出了异常，是实例化"beanExample"时调用其init方法时NullPointException。开始还没太明白，细看才发现实例"beanExample"之前已经成功实例化，那为何在调用Process时调用其init方法时会出现"beanExample"为null呢？开始没及理解也没去了解BeanPostProcessor的源码，就如上简单做了下实例。之后分析BeanPostProcessor的源码及分析才发现Processor是通过此方式调用的（即将返回值赋值给使用的result; 而我上面示例粗暴的返回null正好将之前实例化的对象又置为null了）：
```
for (BeanPostProcessor beanProcessor : getBeanPostProcessors()) {
			result = beanProcessor.postProcessBeforeInitialization(result, beanName);
			if (result == null) {
				return result;
			}
		}
```
那修改就简单了，将上面两个Processor示例的对应方法的"retrun null;"统一修改为"return bean;"即可，再将启动应用，发现麻利的就OK了，启动日志：
```language
==> aoe ContextLoaderListener contextInitialized start。。。
2019-7-3 18:12:38 org.apache.catalina.core.ApplicationContext log
信息: Initializing Spring root WebApplicationContext
2019-7-3 18:12:38 org.springframework.web.context.ContextLoader initWebApplicationContext
信息: Root WebApplicationContext: initialization started
2019-7-3 18:12:39 org.springframework.context.support.AbstractApplicationContext prepareRefresh
信息: Refreshing Root WebApplicationContext: startup date [Wed Jul 03 18:12:39 CST 2019]; root of context hierarchy
2019-7-3 18:12:39 org.springframework.beans.factory.xml.XmlBeanDefinitionReader loadBeanDefinitions
信息: Loading XML bean definitions from class path resource [applicationContext.xml]
SimpleProcessorExample afterPropertiesSet。。。
2019-7-3 18:12:41 org.springframework.beans.factory.support.DefaultListableBeanFactory preInstantiateSingletons
信息: Pre-instantiating singletons in org.springframework.beans.factory.support.DefaultListableBeanFactory@1531164: defining beans [beanExample,beanExample1,beanExample2,annotationBean,annotationBeanInjectExample,applicationContextAwareExample,org.springframework.context.annotation.internalConfigurationAnnotationProcessor,org.springframework.context.annotation.internalAutowiredAnnotationProcessor,org.springframework.context.annotation.internalRequiredAnnotationProcessor,org.springframework.context.annotation.internalCommonAnnotationProcessor,simpleProcessorExample,org.springframework.context.annotation.ConfigurationClassPostProcessor$ImportAwareBeanPostProcessor#0]; root of factory hierarchy
SimpleProcessorExample postProcessBeforeInitialization。。。
BeanExample init-method is called:27503148
******************************
SimpleProcessorExample postProcessAfterInitialization。。。
SimpleProcessorExample postProcessBeforeInitialization。。。
BeanExample init-method is called:33116517
******************************
SimpleProcessorExample postProcessAfterInitialization。。。
SimpleProcessorExample postProcessBeforeInitialization。。。
BeanExample init-method is called:21943671
******************************
SimpleProcessorExample postProcessAfterInitialization。。。
SimpleProcessorExample postProcessBeforeInitialization。。。
AnnotationBean afterPropertiesSet has been created:10344162
SimpleProcessorExample postProcessAfterInitialization。。。
SimpleProcessorExample postProcessBeforeInitialization。。。
AnnotationBeanInjectExample init-method is called:272782
******************************
beanExample:27503148
beanExample1:33116517
beanExample2:21943671
annotationBean:10344162
AnnotationBeanInjectExample afterPropertiesSet has been created:272782
SimpleProcessorExample postProcessAfterInitialization。。。
SimpleProcessorExample postProcessBeforeInitialization。。。
ApplicationContextAwareExample  name:null
ApplicationContextAwareExample  DisplayName:Root WebApplicationContext
ApplicationContextAwareExample  StartupDate:1562148759136
SimpleProcessorExample postProcessAfterInitialization。。。
==> aoe ContextLoaderListener contextInitialized end。。。
2019-7-3 18:12:41 org.springframework.web.context.ContextLoader initWebApplicationContext
信息: Root WebApplicationContext: initialization completed in 3208 ms
2019-7-3 18:12:41 org.apache.coyote.AbstractProtocolHandler start
信息: Starting ProtocolHandler ["http-bio-8080"]
2019-7-3 18:12:41 org.apache.coyote.AbstractProtocolHandler start
信息: Starting ProtocolHandler ["ajp-bio-8009"]
2019-7-3 18:12:41 org.apache.catalina.startup.Catalina start
信息: Server startup in 4760 ms
```
额，然而，发现SimpleProcessorExample 确实实例化并执行了，但基于注解的AnnotationProcessorExample并未被实例化并调用？开始来纠结了一小会会儿，原因很简单，因为AnnotationProcessorExample 的package并没有在component-scan中配置，调整配置重启，一切OK。
