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

### 一.HttpRequestHandlerAdapter 实践
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
- 结合上面的源码，HttpRequestHandlerAdapter示例相对简单，即判断当前Hanlder是否为HttpRequestHandler的实例，若是则直接返回当前Hanlder（参考上一章，Hanlder可简单理解为对应ExampleController的实例），此处还是花费了比较多的时间，最终还是大致理解了下HttpRequestHandler的注释：HttpRequestHandler用于处理http请求；最简单的示例是在web.xml定义HttpRequestHandler bean并通过servlet-name与该beanName匹配；典型的实现就是直接构建binary响应而不需要view资源，这区别于SpringMVC的Controller，仅支持简单返回ModeAndView。同Spring2.0一样，基于Spring的远程服务提供，诸如HttpInvokerServiceExporter、HessianServiceExporter。实现该接口比Controller接口更具扩展性，因其对于spring web基础构建的最小依赖。
- 


### 二.SimpleControllerHandlerAdapter 实践
```language
public class SimpleControllerHandlerAdapter implements HandlerAdapter {

	public boolean supports(Object handler) {
		return (handler instanceof Controller);
	}

	public ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler)
			throws Exception {

		return ((Controller) handler).handleRequest(request, response);
	}

	public long getLastModified(HttpServletRequest request, Object handler) {
		if (handler instanceof LastModified) {
			return ((LastModified) handler).getLastModified(request);
		}
		return -1L;
	}

}
```
通过源码分析，即hanlder对应类需是Controller的子类；于是乎开始尝试编写示例代码（如若使用@Controller注解与implements Controller时里特殊处理）两种不同的写法：
```language
 import org.springframework.web.servlet.mvc.AbstractController;
 
 @Controller(value="/beanNameUrlHttp")
public class SimpleControllerHandlerAdapterController extends AbstractController {

	@Override
	protected ModelAndView handleRequestInternal(HttpServletRequest request, HttpServletResponse response)
			throws Exception {
		  ModelAndView model = new ModelAndView("/index.jsp");
	      model.addObject("message", "Welcome!");
	      return model;
	}

}
```
```language
@Controller(value="/beanNameUrlHttp")
public class SimpleControllerHandlerAdapterController implements org.springframework.web.servlet.mvc.Controller {

	public ModelAndView handleRequest(HttpServletRequest request, HttpServletResponse response) throws Exception {
		 ModelAndView model = new ModelAndView("/index.jsp");
	      model.addObject("message", "Welcome!");
	      return model;
	}
} 
```
即子类继承或实现对应Conroller接口时需实现父类如上方法，示例即返回应用/下的index.jsp页面，访问 http://localhost:8080/springmvc3-analysis/example/beanNameUrlHttp 确实如预期。从示例来看，即获取HandlerAdapter时判断当前Controller是Controller接口的子类实例 ，故返回的是SimpleControllerHandlerAdapter 的实例ha，而在之后调用ha的handle方法，其内部即是直接调用Handler实例（即SimpleControllerHandlerAdapterController实例）的handleRequest（）方法，可发现即访问默认方法（之前一直有纠结，基于BeanNameUrlHandlerMapping虽然关联到对应的class或者说bean实例，但是如何识别到方法呢？其实是思考方式错了）

### 三.AnnotationMethodHandlerAdapter实践
AnnotationMethodHandlerAdapter为三个适配器Adaptor的最后一个，即前面的两个Adaptor若未匹配到则全部由该Adaptor处理，该方法的support返回AnnotationMethodHandlerAdapter实例
```language
	public boolean supports(Object handler) {
		return getMethodResolver(handler).hasHandlerMethods();
	}
       
        private ServletHandlerMethodResolver getMethodResolver(Object handler) {
		Class handlerClass = ClassUtils.getUserClass(handler);
                //此处handlerClass被首次访问时默认为null，之后的访问则可直接从cache获取提升性能
		ServletHandlerMethodResolver resolver = this.methodResolverCache.get(handlerClass);
		//首次访问handlerClass为null，即实例化resolver 并设置至缓存（采用double check)
                if (resolver == null) {
			synchronized (this.methodResolverCache) {
				resolver = this.methodResolverCache.get(handlerClass);
				if (resolver == null) {
                           //名称即为解析器，其会解析handlerClass的Methode及Method&RequestMapping映射关系
					resolver = new ServletHandlerMethodResolver(handlerClass);
					this.methodResolverCache.put(handlerClass, resolver);
				}
			}
		}
		return resolver;
	}
```
而在获取AnnotationMethodHandlerAdapter实例之后，会调用其handler方法，该方法会根据urlPath解析处理的地址/helloWorld （如访问：http://localhost:8080/springmvc3-analysis/example/helloWorld）与之前实例化的resolver 中的信息匹配获得Method，最后基反射调用方法方法返回result。