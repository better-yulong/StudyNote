### 一.Spring Aware接口
Spring Aware的目的是为了让bean获取spring容器的服务，即实现Aware接口后，容器在bean实例化时会自动将容器本身的服务作为参参调用特定方法：
  - BeanNameAware ：可以获取容器中bean的名称
  - BeanFactoryAware:获取当前bean factory这也可以调用容器的服务
  - ApplicationContextAware： 当前的applicationContext， 这也可以调用容器的服务
  - MessageSourceAware：获得message source，这也可以获得文本信息
  - applicationEventPulisherAware：应用事件发布器，可以发布事件，
  - ResourceLoaderAware： 获得资源加载器，可以获得外部资源文件的内容；



### 二.BeanPostProcessor接口
如果我们想在Spring容器中完成bean实例化、配置以及其他初始化方法前后要添加一些自己逻辑处理。我们需要定义一个或多个BeanPostProcessor接口实现类，然后注册到Spring IoC容器中。而从上面的分析来看，Processor由DefaultListableBeanFactory(AbstractBeanFactory).addBeanPostProcessor(BeanPostProcessor)添加（即beanFactory属性List<BeanPostProcessor> beanPostProcessors).
1. prepareBeanFactory:即初始化prepareBeanFactory时会添加ApplicationContextAwareProcessor：该processor是对实现了Aware接口的bean的处理
