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

### 一.HttpRequestHandlerAdapter实践
HttpRequestHandlerAdapter源码：
```language
public class HttpRequestHandlerAdapter implements HandlerAdapter {

	public boolean supports(Object handler) {
		return (handler instanceof HttpRequestHandler);
	}

	public ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler)
			throws Exception {

		((HttpRequestHandler) handler).handleRequest(request, response);
		return null;
	}

	public long getLastModified(HttpServletRequest request, Object handler) {
		if (handler instanceof LastModified) {
			return ((LastModified) handler).getLastModified(request);
		}
		return -1L;
	}

}
```
结合上面的源码，HttpRequestHandlerAdapter示例相对简单，即判断当前Hanlder是否为HttpRequestHandler的实例，若是则直接返回当前Hanlder（参考上一章，Hanlder可简单理解为对应ExampleController的实例），此处还是花费了比较多的时间，最终还是大致理解了下HttpRequestHandler的注释：HttpRequestHandler用于处理http请求；最简单的示例是在web.xml定义HttpRequestHandler bean并通过servlet-name与该beanName匹配；典型的实现就是直接构建binary响应而不需要view资源，这区别于SpringMVC

