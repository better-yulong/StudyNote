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
1. prepareBeanFactory:即初始化prepareBeanFactory时会添加ApplicationContextAwareProcessor：该processor是对实现了Aware接口的bean的处理（如上分析）
2. postProcessBeanFactory(beanFactory)：添加ServletContextAwareProcessor
