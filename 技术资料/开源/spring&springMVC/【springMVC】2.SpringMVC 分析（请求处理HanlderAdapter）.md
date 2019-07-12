- 上节回顾：
示例根据http://localhost:8080/springmvc3-analysis/exampleController 确实可基于BeanNameUrlHanlderMapping匹配到Hanlder（因为ExampleController类有显示使用@Controller（value="exampleController")),但在之后根据Hanlder获取HandlerAdapter：实现则是使用Hanlder作为参数，依次调用HttpRequestHandlerAdapter、SimpleControllerHandlerAdapter、AnnotationMethodHandlerAdapter的support方法并返回HandlerAdapter实例。
- 为便于演示，本节会新写示例；其中DispathcerServlet的getHandlerAdapter源码：
```language
	protected HandlerAdapter getHandlerAdapter(Object handler) throws ServletException {
		for (HandlerAdapter ha : this.handlerAdapters) {
			if (ha.supports(handler)) {
				return ha;
			}
		}
		throw new ServletException("No adapter for handler [" + handler +
```

### 一.HttpRequestHandlerAdapter示例



