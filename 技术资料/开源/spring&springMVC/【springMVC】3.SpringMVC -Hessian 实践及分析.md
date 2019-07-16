结合之前的分析，根据spring 文档（章节：3 Using Hessian or Burlap to remotely call services via HTTP），搭建hessian示例工程并运行。

### 一. demo示例编写
#### 1.1 创建示例工程
创建示例web 工程rpc-client、rpc-server,及java工程 rpc-skeleton，分别对于hessian客户端、hessian服务端、rpc骨架包
##### 1.1.1 rpc-skeleton 接口
```language
	package com.aoe.demo.rpc.hessian;

	import java.util.List;

	public interface HessianExampleInterf1 {

		List getServiceNames(List paramList);

	}
```
之后需在rpc-client、rpc-server 应用的pom.xml中配置对rpc-skeleton的依赖.
##### 1.1.2 rpc-server 服务
HessianExampleService1服务实现类
```language
package com.aoe.demo.rpc.hessian;

import java.util.ArrayList;
import java.util.List;

public class HessianExampleService1 implements HessianExampleInterf1 {
	
	/* (non-Javadoc)
	 * @see com.aoe.demo.rpc.hessian.HessianExampleInterf1#getServiceNames(java.util.List)
	 */
	public List getServiceNames(List paramList){
		System.out.println("param0:" + paramList.get(0));
		List serviceNames = new ArrayList<String>();
		serviceNames.add("service hessianExampleService1");
		return serviceNames ;
	}

}

```
spring-servlet.xml配置
```language
<bean name="hessianExampleService1"  class="com.aoe.demo.rpc.hessian.HessianExampleService1"></bean>
	
	<bean name="/hessianExampleService1" class="org.springframework.remoting.caucho.HessianServiceExporter">
	    <property name="service" ref="hessianExampleService1"/>
	    <property name="serviceInterface" value="com.aoe.demo.rpc.hessian.HessianExampleInterf1"/>
	</bean>
```
pom.xml中配置对hessian的jar依赖
```language
     <dependency>
	<groupId>com.caucho</groupId>
	<artifactId>com.springsource.com.caucho</artifactId>
	<version>3.2.1</version>
    </dependency>  
```
##### 1.1.3 rpc-client 客户端
外部请求Controller（用于发起hessian客户端请求）
```language
package com.aoe.demo.rpc.controller;

import java.util.ArrayList;
import java.util.List;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.ResponseBody;

import com.aoe.demo.rpc.hessian.HessianExampleInterf1;

@Controller
public class EntryController {
	
	@Autowired
	private HessianExampleInterf1 hessianExampleService;
	
	@RequestMapping("/entry")
	@ResponseBody
	public Object entry(){
		List<String> params =  new ArrayList<String>();
		params.add("parm");
		System.out.println("result0:" + hessianExampleService.getServiceNames(params).get(0));
		return "entry";
	}

}
```
spring-servlet.xml配置
```language
 <bean id="accountService" class="org.springframework.remoting.caucho.HessianProxyFactoryBean">
    	<property name="serviceUrl" value="http://localhost:8080/rpc-server/rpc/hessianExampleService1"/>
    	<property name="serviceInterface" value="com.aoe.demo.rpc.hessian.HessianExampleInterf1"/>
	</bean> 
```
同rpc-server在其pom.xml中配置对hessian的jar依赖
##### 1.1.4 验证
访问http://localhost:8080/rpc-client/rpc/entry，如预期完成hessian调用验证。

### 二. hessian 服务端（HessianServiceExporter）
HessianServiceExporter源码
```language
public class HessianServiceExporter extends HessianExporter implements HttpRequestHandler {

	/**
	 * Processes the incoming Hessian request and creates a Hessian response.
	 */
	public void handleRequest(HttpServletRequest request, HttpServletResponse response)
			throws ServletException, IOException {

		if (!"POST".equals(request.getMethod())) {
			throw new HttpRequestMethodNotSupportedException(request.getMethod(),
					new String[] {"POST"}, "HessianServiceExporter only supports POST requests");
		}

		response.setContentType(CONTENT_TYPE_HESSIAN);
		try {
		  invoke(request.getInputStream(), response.getOutputStream());
		}
		catch (Throwable ex) {
		  throw new NestedServletException("Hessian skeleton invocation failed", ex);
		}
	}

}
```
HessianServiceExporter 类本身的源码仅包含handleRequest方法；分析之前先回顾下之前分析SpringMVC关于获取Hanlder、HanlderAdapter及调用原理:
1. hanlder注册:SpringMVC容器启动初始化上下文时，会检测所有的Hanlder(注解或者beanName）；而当前示例hessian服务发布时beanName为"/hessianExampleService1"，符合该规则即在rpc-server启动时，会调用this.handlerMap.put(urlPath, resolvedHandler) 完成urlPath("/hessianExampleService1")与resolvedHandler（HessianServiceExporter实例）的映射；
2. hanlder获取：根据rpc-client的配置客户端发起hessian服务调用时（<property name="serviceUrl" value="http://localhost:8080/rpc-server/rpc/hessianExampleService1"/>），获取到步骤1注册时对应的hanlder（HessianServiceExporter实例）;
3. HandlerAdapter获取：获取hanlder实例后，DispathcerServlet类getHandlerAdapter方法中，调用HttpRequestHandlerAdapter.support方法判断hanlder类型匹配（handler instanceof HttpRequestHandler）直接返回HttpRequestHandlerAdapter实例ha; 
4. 请求处理 ha.handle：对应代码ha.handle(processedRequest, response, mappedHandler.getHandler())，而ha.handle实际调用的是handler.handleRequest; 即可理解为实际调用为HessianServiceExporter类handleRequest方法：
```language
	public void handleRequest(HttpServletRequest request, HttpServletResponse response)
			throws ServletException, IOException {

		if (!"POST".equals(request.getMethod())) {
			throw new HttpRequestMethodNotSupportedException(request.getMethod(),
					new String[] {"POST"}, "HessianServiceExporter only supports POST requests");
		}

		response.setContentType(CONTENT_TYPE_HESSIAN);
		try {
		  invoke(request.getInputStream(), response.getOutputStream());
		}
		catch (Throwable ex) {
		  throw new NestedServletException("Hessian skeleton invocation failed", ex);
		}
	}
```
HessianServiceExporter类handleRequest方法相对简单：1、要求http请求必须为POST；2.设置ContentType并调用invoke方法(其又调用HessianExporter的doInvoke方法)：
```language 
        //skeleton封装_apiClass：interface com.aoe.demo.rpc.hessian.HessianExampleInterf1；_methodMap：方法列表；_sercice:提供被服实例的Proxy对象
	protected void doInvoke(HessianSkeleton skeleton, InputStream inputStream, OutputStream outputStream)
			throws Throwable {
                //使用当前环境的ClassLoaders覆盖当前线程的ClassLoader（tomcat一般均为WebappClassLoader）
		ClassLoader originalClassLoader = overrideThreadContextClassLoader();
		try {
			InputStream isToUse = inputStream;
			OutputStream osToUse = outputStream;
                        //debug级别：实例化debug级别的Input、Output日志相关实例并启动并包装默认IO对象
			if (this.debugLogger != null && this.debugLogger.isDebugEnabled()) {
				PrintWriter debugWriter = new PrintWriter(new CommonsLogWriter(this.debugLogger));
				HessianDebugInputStream dis = new HessianDebugInputStream(inputStream, debugWriter);
				dis.startTop2();
				HessianDebugOutputStream dos = new HessianDebugOutputStream(outputStream, debugWriter);
				dos.startTop2();
				isToUse = dis;
				osToUse = dos;
			}
                        //非debug级别的 isToUse 实例化
			if (!isToUse.markSupported()) {
				isToUse = new BufferedInputStream(isToUse);
				isToUse.mark(1);
			}
                        //读取IO输入流（hessian版本标识hcar的int值，见下面代码）
			int code = isToUse.read();
			int major;
			int minor;

			AbstractHessianInput in;
			AbstractHessianOutput out;
                        //根据客户端hessian版本标识及主次版本号调用不同实现将输入、输出io对象墙头成对应的hessian的Input或Output对象
			if (code == 'H') {
				// Hessian 2.0 stream
				major = isToUse.read();
				minor = isToUse.read();
				if (major != 0x02) {
					throw new IOException("Version " + major + "." + minor + " is not understood");
				}
				in = new Hessian2Input(isToUse);
				out = new Hessian2Output(osToUse);
				in.readCall();
			}
			else if (code == 'C') {
				// Hessian 2.0 call... for some reason not handled in HessianServlet!
				isToUse.reset();
				in = new Hessian2Input(isToUse);
				out = new Hessian2Output(osToUse);
				in.readCall();
			}
			else if (code == 'c') {
				// Hessian 1.0 call
				major = isToUse.read();
				minor = isToUse.read();
				in = new HessianInput(isToUse);
				if (major >= 2) {
					out = new Hessian2Output(osToUse);
				}
				else {
					out = new HessianOutput(osToUse);
				}
			}
			else {
				throw new IOException("Expected 'H'/'C' (Hessian 2.0) or 'c' (Hessian 1.0) in hessian input at " + code);
			}
                        //设置com.caucho.hessian.io.SerializerFactory
			if (this.serializerFactory != null) {
				in.setSerializerFactory(this.serializerFactory);
				out.setSerializerFactory(this.serializerFactory);
			}
                        //反射调用
			try { 
                                //实际调用 invoke(_service, in, out);第1个参数即为代理对象。而invoke方法根据hessian序列化协议从in获取methodName、argLength匹配实际调用的Method对象，并读取参数类型、长度获取放入参。最终通过 result = method.invoke(service, values)反射调用实际业务方法
				skeleton.invoke(in, out);
			}
			finally {
				try {
					in.close();
					isToUse.close();
				}
				catch (IOException ex) {
					// ignore
				}
				try {
					out.close();
					osToUse.close();
				}
				catch (IOException ex) {
					// ignore
				}
			}
		}
		finally {
			resetThreadContextClassLoader(originalClassLoader);
		}
	}
```
- 每次请求处理均需实例化HessianInput、HessianOutput、Hessian2Input、Hessian2Output实例（原型模式）；而HessianInput及Hessian2Input自定义实现readObject方法、HessianOutput及Hessian2Output重写writeObject，其有序列化及反序列化之后，会先行判断并实例化SerializerFactory并从当前支持的_staticSerializerMap、_staticDeserializerMap获取对应的Serializer、Deserializer实例。而在hessian3.2.1版本（即当前分析的版本）中每次都会创建一个SerializerFactory会导致性能问题（见HessianInput、HessianOutput类带IO参数的构造方法，其中会判断_serializerFactory是否为null并new SerializerFactory(),也可见：https://javatar.iteye.com/blog/852663）。从源码看使用Hessian2则已修复该问题，至于如何升级到Hessian2协议？后面再补充
- hessian调用逻辑，比之前相像的相对简单些；但是看调用发现，与之前spring的bean使用一样，其在实例化时应该也有做不少的处理；于是乎继续分析HessianServiceExporter源码，发现其直接父类HessianExporter有实现InitializingBean接口，既然看到了这个熟悉的接口，就明白该看啥了吧？瞄了下，afterPropertiesSet()方法调用prepare():
```language
        public void prepare() {
               //获取实例service变量值：对应xml配置的hessianExampleService1（即被发布为hessian的bean）
		checkService();
               //获取实例ServiceInterface变量值：对应xml配置的的interface接口名
		checkServiceInterface();
                //实例化HessianSkeleton：其中核心部分就是为当前被代理的bean基于AOPProxy生成代理对象同时也会对interceptor进行增强处理
		this.skeleton = new HessianSkeleton(getProxyForService(), getServiceInterface());
	} 
```

### 三. hessian 客户端请求（HessianProxyFactoryBean)
HessianProxyFactoryBean间接继承InitializingBean、FactoryBean，直接继承HessianClientInterceptor；
#### 3.1 InitializingBean初始化
##### 3.1.1 HessianProxyFactoryBean类afterPropertiesSet
```language
	@Override
	public void afterPropertiesSet() {
		super.afterPropertiesSet();
		this.serviceProxy = new ProxyFactory(getServiceInterface(), this).getProxy(getBeanClassLoader());
	}
```
1. HessianClientInterceptor类afterPropertiesSet
```language
	@Override
	public void afterPropertiesSet() {
                //检验serviceUrl不可为空
		super.afterPropertiesSet();
                //对应下面方法创建hessianProxy 
		prepare();
	}

	/**
	 * Initialize the Hessian proxy for this interceptor.
	 * @throws RemoteLookupFailureException if the service URL is invalid
	 */
	public void prepare() throws RemoteLookupFailureException {
		try {
			this.hessianProxy = createHessianProxy(this.proxyFactory);
		}
		catch (MalformedURLException ex) {
			throw new RemoteLookupFailureException("Service URL [" + getServiceUrl() + "] is invalid", ex);
		}
	}

	/**
	 * Create the Hessian proxy that is wrapped by this interceptor.
	 * @param proxyFactory the proxy factory to use
	 * @return the Hessian proxy
	 * @throws MalformedURLException if thrown by the proxy factory
	 * @see com.caucho.hessian.client.HessianProxyFactory#create
	 */
	protected Object createHessianProxy(HessianProxyFactory proxyFactory) throws MalformedURLException {
		Assert.notNull(getServiceInterface(), "'serviceInterface' is required");
                //见HessianProxyFactory类create方法
		return proxyFactory.create(getServiceInterface(), getServiceUrl());
	}
```
最终调用HessianProxyFactory类create方法基于动态代理创建代理bean
```language
  public Object create(Class api, String urlName, ClassLoader loader)
    throws MalformedURLException
  {
    if (api == null)
      throw new NullPointerException("api must not be null for HessianProxyFactory.create()");
    InvocationHandler handler = null;

    URL url = new URL(urlName); 
    handler = new HessianProxy(this, url);

    return Proxy.newProxyInstance(loader,
                                  new Class[] { api,
                                                HessianRemoteObject.class },
                                  handler);
  }
```
从上面代码可看出，动态代理bean各方法最终会统一调用HessianProxy的invoke方法
```language
  //HessianProxy类invoke、sendRequest方法
  public Object invoke(Object proxy, Method method, Object []args)
    throws Throwable
  {
    String mangleName;
    //经验证，如若并发访问同一方法时，在此处会由于synchronized出现瞬间阻塞，具体性能影响多大后面有空再验证
    synchronized (_mangleMap) {
      mangleName = _mangleMap.get(method);
    }

    if (mangleName == null) {
      String methodName = method.getName();
      Class []params = method.getParameterTypes();

      // equals and hashCode are special cased
      if (methodName.equals("equals")
	  && params.length == 1 && params[0].equals(Object.class)) {
	Object value = args[0];
	if (value == null || ! Proxy.isProxyClass(value.getClass()))
	  return Boolean.FALSE;

	Object proxyHandler = Proxy.getInvocationHandler(value);

	if (! (proxyHandler instanceof HessianProxy))
	  return Boolean.FALSE;
	
	HessianProxy handler = (HessianProxy) proxyHandler;

	return new Boolean(_url.equals(handler.getURL()));
      }
      else if (methodName.equals("hashCode") && params.length == 0)
	return new Integer(_url.hashCode());
      else if (methodName.equals("getHessianType"))
	return proxy.getClass().getInterfaces()[0].getName();
      else if (methodName.equals("getHessianURL"))
	return _url.toString();
      else if (methodName.equals("toString") && params.length == 0)
	return "HessianProxy[" + _url + "]";
      
      if (! _factory.isOverloadEnabled())
	mangleName = method.getName();
      else
        mangleName = mangleName(method);
      //此处类似于缓存机制，此处put执行之后上面的get方法即可获取
      synchronized (_mangleMap) {
	_mangleMap.put(method, mangleName);
      }
    }

    InputStream is = null;
    URLConnection conn = null;
    HttpURLConnection httpConn = null;
    
    try {
      if (log.isLoggable(Level.FINER))
	log.finer("Hessian[" + _url + "] calling " + mangleName);
      //基于HessianProxy代理对像Url:url.openConnection(),设置Content-Type、校验信息、addRequestHeaders（默认为空方法，可用于扩展实现）--具体分析查看该方法下面的源码分析
      conn = sendRequest(mangleName, args);

      if (conn instanceof HttpURLConnection) {
	httpConn = (HttpURLConnection) conn;
        int code = 500;

        try {
          code = httpConn.getResponseCode();
        } catch (Exception e) {
        }

        parseResponseHeaders(conn);

        if (code != 200) {
          StringBuffer sb = new StringBuffer();
          int ch;

          try {
            is = httpConn.getInputStream();

            if (is != null) {
              while ((ch = is.read()) >= 0)
                sb.append((char) ch);

              is.close();
            }

            is = httpConn.getErrorStream();
            if (is != null) {
              while ((ch = is.read()) >= 0)
                sb.append((char) ch);
            }
          } catch (FileNotFoundException e) {
            throw new HessianConnectionException("HessianProxy cannot connect to '" + _url, e);
          } catch (IOException e) {
	    if (is == null)
	      throw new HessianConnectionException(code + ": " + e, e);
	    else
	      throw new HessianConnectionException(code + ": " + sb, e);
          }

          if (is != null)
            is.close();

          throw new HessianConnectionException(code + ": " + sb.toString());
        }
      }

      is = conn.getInputStream();

      if (log.isLoggable(Level.FINEST)) {
	PrintWriter dbg = new PrintWriter(new LogWriter(log));
	HessianDebugInputStream dIs
	  = new HessianDebugInputStream(is, dbg);

	dIs.startTop2();
	
	is = dIs;
      }

      AbstractHessianInput in;

      int code = is.read();

      if (code == 'H') {
	int major = is.read();
	int minor = is.read();

	in = _factory.getHessian2Input(is);

	return in.readReply(method.getReturnType());
      }
      else if (code == 'r') {
	in = _factory.getHessianInput(is);

	in.startReply();

	Object value = in.readObject(method.getReturnType());

	if (value instanceof InputStream) {
	  value = new ResultInputStream(httpConn, is, in, (InputStream) value);
	  is = null;
	  httpConn = null;
	}
	else
	  in.completeReply();

	return value;
      }
      else
	throw new HessianProtocolException("'" + (char) code + "' is an unknown code");
    } catch (HessianProtocolException e) {
      throw new HessianRuntimeException(e);
    } finally {
      try {
	if (is != null)
	  is.close();
      } catch (Exception e) {
	log.log(Level.FINE, e.toString(), e);
      }
      
      try {
	if (httpConn != null)
	  httpConn.disconnect();
      } catch (Exception e) {
	log.log(Level.FINE, e.toString(), e);
      }
    }
  }
  //sendRequest方法
  protected URLConnection sendRequest(String methodName, Object []args)
    throws IOException
  {
    URLConnection conn = null;
    
    conn = _factory.openConnection(_url);
    boolean isValid = false;

    try {
      // Used chunked mode when available, i.e. JDK 1.5.
      if (_factory.isChunkedPost() && conn instanceof HttpURLConnection) {
	try {
	  HttpURLConnection httpConn = (HttpURLConnection) conn;

	  httpConn.setChunkedStreamingMode(8 * 1024);
	} catch (Throwable e) {
	}
      }
      //emtpty方法，可扩展用于设置请求头
      addRequestHeaders(conn);

      OutputStream os = null;

      try {
	os = conn.getOutputStream();
      } catch (Exception e) {
	throw new HessianRuntimeException(e);
      }

      if (log.isLoggable(Level.FINEST)) {
	PrintWriter dbg = new PrintWriter(new LogWriter(log));
	os = new HessianDebugOutputStream(os, dbg);
      }
      //该方法即获取HessianOutput实例，即_isHessian2Request标识为true则会使用Hessian2Output(_isHessian2Reques在当前版本默认为false)
      AbstractHessianOutput out = _factory.getHessianOutput(os);
      //该方法即有之前分析服务端方法时的疑问：hessian标识、版本号、以及方法名长度、方法名（即遵循对应的hessian协议）；最后则是调用writeObject写入arg参数，并最终写入结束标识z（间接说明，具体写入的hessian版本等信息与out实例有前）
      out.call(methodName, args);
      out.flush();

      isValid = true;

      return conn;
    } finally {
      if (! isValid && conn instanceof HttpURLConnection)
	((HttpURLConnection) conn).disconnect();
    }
  }
```
getHessianOutput方法根据_isHessian2Request参数确认返回Output实例类型；而_isHessian2Request为HessianProxyFactory类的实例变量，默认为false，但可通过HessianClientInterceptor类的setHessian2(boolean hessian2)指定是否使用hessian2。那么到此就好处理了，因为HessianClientInterceptor是HessianProxyFactoryBean的直接父类，根据之前配置url的思路，那在xml中配置应该就可以，于是验证。
```language
  public AbstractHessianOutput getHessianOutput(OutputStream os)
  {
    AbstractHessianOutput out;

    if (_isHessian2Request)
      out = new Hessian2Output(os);
    else {
      HessianOutput out1 = new HessianOutput(os);
      out = out1;

      if (_isHessian2Reply)
        out1.setVersion(2);
    }
      
    out.setSerializerFactory(getSerializerFactory());

    return out;
  }
```
修改rpc-client的spring 的bean注入xml配置：
```language
	 <bean id="accountService" class="org.springframework.remoting.caucho.HessianProxyFactoryBean">
    	<property name="serviceUrl" value="http://localhost:8080/rpc-server/rpc/hessianExampleService1"/>
    	<property name="serviceInterface" value="com.aoe.demo.rpc.hessian.HessianExampleInterf1"/>
    	<property name="hessian2" value="true"></property>
	</bean>
```
额，运行OK，客户端调试发现使用的确实如预期为Hessian2Output；那服务端呢？调试如预期。



