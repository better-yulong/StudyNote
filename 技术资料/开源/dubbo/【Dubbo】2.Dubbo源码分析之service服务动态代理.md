针对上一篇服务注册源码，如erviceConfig类源码(及参数）：
```language
    @SuppressWarnings({ "unchecked", "rawtypes" })
    private void exportLocal(URL url) {
        if (!Constants.LOCAL_PROTOCOL.equalsIgnoreCase(url.getProtocol())) {
            URL local = URL.valueOf(url.toFullString())
                    .setProtocol(Constants.LOCAL_PROTOCOL)
                    .setHost(NetUtils.LOCALHOST)
                    .setPort(0);
            Exporter<?> exporter = protocol.export(
                    proxyFactory.getInvoker(ref, (Class) interfaceClass, local));
            exporters.add(exporter);
            logger.info("Export dubbo service " + interfaceClass.getName() +" to local registry");
        }
    }

private static final Protocol protocol = ExtensionLoader.getExtensionLoader(Protocol.class).getAdaptiveExtension();
    
private static final ProxyFactory proxyFactory = ExtensionLoader.getExtensionLoader(ProxyFactory.class).getAdaptiveExtension();
```
1. local(toString()):injvm://127.0.0.1/com.aoe.demo.rpc.dubbo.DubboExampleInterf1?anyhost=true&application=rpc-server&default.timeout=1000&dubbo=2.5.3&interface=com.aoe.demo.rpc.dubbo.DubboExampleInterf1&methods=serviceProvider&pid=3196&revision=0.0.1-SNAPSHOT&side=provider&timestamp=1563938432923
2. ref:com.aoe.demo.rpc.dubbo.DubboExampleService1@1d7e4d6
3. interfaceClass:interface com.aoe.demo.rpc.dubbo.DubboExampleInterf1
4. exporter:com.alibaba.dubbo.rpc.listener.ListenerExporterWrapper@1e41869（其包含被包装实例exporter:InjvmExporter)

### 一.Dubbo SPI架构
- 开始有尝试调试如上代码，但感觉很难对整体有一个清晰的思路；于是决定换个角度，先从整体理解下Dubbo SPI的整体架构（Dubbo 的微内核设计，可参考资料：https://my.oschina.net/j4love/blog/1813040、https://www.jianshu.com/p/7daa38fc9711）。
- 从上面资料，可将ExtensionLoader对比ServiceLoader，而Protocol 、ProxyFactory、ExtensionFactory即为满足Dubbo SPI的两个自定义扩展点。
#### 1.1 ExtensionLoader实例化
参考如上的源码，ExtensionLoader实例化通过类似方式：ExtensionLoader.getExtensionLoader(Protocol.class)
```language
    @SuppressWarnings("unchecked")
    public static <T> ExtensionLoader<T> getExtensionLoader(Class<T> type) {
        if (type == null)
            throw new IllegalArgumentException("Extension type == null");
        //type必须为接口，即对应Dubbo SPI约定: 扩展点必须是 Interface 类型 ， 必须被 @SPI 注解
        if(!type.isInterface()) {
            throw new IllegalArgumentException("Extension type(" + type + ") is not interface!");
        }
        //即Dubbo SPI约定，检查当前class是否被 @SPI 注解（return type.isAnnotationPresent(SPI.class)）
        if(!withExtensionAnnotation(type)) {
            throw new IllegalArgumentException("Extension type(" + type + 
                    ") is not extension, because WITHOUT @" + SPI.class.getSimpleName() + " Annotation!");
        }
        //实例化当前扩展点type对应的ExtensionLoader
        ExtensionLoader<T> loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
        if (loader == null) {
            EXTENSION_LOADERS.putIfAbsent(type, new ExtensionLoader<T>(type));
            loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
        }
        return loader;
    }

    //ExtensionLoader 带参构造方法
    private ExtensionLoader(Class<?> type) {
        this.type = type;
        //同上，非ExtensionFactory实例化ExtensionFactory扩展点对应的ExtensionLoader，getAdaptiveExtension方法分析往下看；
        objectFactory = (type == ExtensionFactory.class ? null : ExtensionLoader.getExtensionLoader(ExtensionFactory.class).getAdaptiveExtension());
    }
     
     private Map<String, Class<?>> getExtensionClasses() {
        //classes示例：{spi=class com.alibaba.dubbo.common.extension.factory.SpiExtensionFactory, spring=class com.alibaba.dubbo.config.spring.extension.SpringExtensionFactory}
        Map<String, Class<?>> classes = cachedClasses.get();
        if (classes == null) {
            synchronized (cachedClasses) {
                classes = cachedClasses.get();
                if (classes == null) {
                    //加载扩展类s,分析往下看
                    classes = loadExtensionClasses();
                    cachedClasses.set(classes);
                }
            }
        }
        return classes;
	}


    private Map<String, Class<?>> loadExtensionClasses() {
       //获取SPI注解实例
        final SPI defaultAnnotation = type.getAnnotation(SPI.class);
       //检查SPI注解指定缺省适应扩展，如@SPI("spi") ，只允许指定1个
        if(defaultAnnotation != null) {
            String value = defaultAnnotation.value();
            if(value != null && (value = value.trim()).length() > 0) {
                String[] names = NAME_SEPARATOR.split(value);
                if(names.length > 1) {
                    throw new IllegalStateException("more than 1 default extension name on extension " + type.getName()
                            + ": " + Arrays.toString(names));
                }
                if(names.length == 1) cachedDefaultName = names[0];
            }
        }
        
        Map<String, Class<?>> extensionClasses = new HashMap<String, Class<?>>();
        loadFile(extensionClasses, DUBBO_INTERNAL_DIRECTORY);//值：META-INF/dubbo/internal/
        loadFile(extensionClasses, DUBBO_DIRECTORY);//值：META-INF/dubbo/
        loadFile(extensionClasses, SERVICES_DIRECTORY);//值：META-INF/services/
        return extensionClasses;
    }
    
    private void loadFile(Map<String, Class<?>> extensionClasses, String dir) {
        //比如：META-INF/dubbo/internal/com.alibaba.dubbo.rpc.Protocol
        String fileName = dir + type.getName();
        try {
            Enumeration<java.net.URL> urls;
            //获取WebappClassLoader
            ClassLoader classLoader = findClassLoader();
            if (classLoader != null) {
                //Finds all the resources with the given name,，即根据指定名查找所有资源
                urls = classLoader.getResources(fileName);
            } else {
                //使用SystemClassLoader或者BootstrapClassLoader根据指定名查找所有资源
                urls = ClassLoader.getSystemResources(fileName);
            }
            if (urls != null) {
                while (urls.hasMoreElements()) {
                    //如jar:file:/D:/***/study-code-effective-java/***/rpc-server/WEB-INF/lib/dubbo-registry-api-2.5.3.jar!/META-INF/dubbo/internal/com.alibaba.dubbo.rpc.Protocol
                    java.net.URL url = urls.nextElement();
                    try {
                        BufferedReader reader = new BufferedReader(new InputStreamReader(url.openStream(), "utf-8"));
                        try {
                            String line = null;
                            while ((line = reader.readLine()) != null) {
                                final int ci = line.indexOf('#');
                                if (ci >= 0) line = line.substring(0, ci);
                                line = line.trim();
                                if (line.length() > 0) {
                                    try {
                                        //Dubbo SPI采用=即key、value方式配置，此处截取
                                        String name = null;
                                        int i = line.indexOf('=');
                                        if (i > 0) {
                                            name = line.substring(0, i).trim();
                                            line = line.substring(i + 1).trim();
                                        }
                                        if (line.length() > 0) {
                                            //加载Dubbo SPI扩展接口实现类，如ProtocolFilterWrapper、ProtocolListenerWrapper、MockProtocol
                                            Class<?> clazz = Class.forName(line, true, classLoader);
                                           //验证clazz与type关系，clazz必须是type接口的实现类
                                            if (! type.isAssignableFrom(clazz)) {
                                                throw new IllegalStateException("Error when load extension class(interface: " +
                                                        type + ", class line: " + clazz.getName() + "), class " 
                                                        + clazz.getName() + "is not subtype of interface.");
                                            }
                                            //clazz就否有@Adaptive注解
                                            if (clazz.isAnnotationPresent(Adaptive.class)) {
                                                if(cachedAdaptiveClass == null) {
                                                    cachedAdaptiveClass = clazz;
                                                } else if (! cachedAdaptiveClass.equals(clazz)) {
                                                    //@Adaptive在类上,即当前类为缺省的适配扩展,每个扩展接口只允许有一个缺省适配扩展
                                                    throw new IllegalStateException("More than 1 adaptive class found: "
                                                            + cachedAdaptiveClass.getClass().getName()
                                                            + ", " + clazz.getClass().getName());
                                                }
                                            } else {
                                                try {
                                                    //获取还参的构造器(一个type接口有多个实现类)
                                                    clazz.getConstructor(type);
                                                    Set<Class<?>> wrappers = cachedWrapperClasses;
                                                    if (wrappers == null) {
                                                        cachedWrapperClasses = new ConcurrentHashSet<Class<?>>();
                                                        wrappers = cachedWrapperClasses;
                                                    }
                                                    //每个EstensionLoader实例对应一个扩展接口type,也对应一个用于存储所有接口实现类clas的set集合
                                                    wrappers.add(clazz);
                                                } catch (NoSuchMethodException e) {
                                                    //获取默认无参的构造器
                                                    clazz.getConstructor();
                                                     //name为空，即未用=号配置成key、value格式 
                                                    if (name == null || name.length() == 0) {
                                                       //获取默认名称 
                                                        name = findAnnotationName(clazz);
                                                        if (name == null || name.length() == 0) {
                                                            if (clazz.getSimpleName().length() > type.getSimpleName().length()
                                                                    && clazz.getSimpleName().endsWith(type.getSimpleName())) {
                                                                name = clazz.getSimpleName().substring(0, clazz.getSimpleName().length() - type.getSimpleName().length()).toLowerCase();
                                                            } else {
                                                                throw new IllegalStateException("No such extension name for the class " + clazz.getName() + " in the config " + url);
                                                            }
                                                        }
                                                    }
                                                     //,拆分name
                                                    String[] names = NAME_SEPARATOR.split(name);
                                                    if (names != null && names.length > 0) {
                                                        //处理@Adaptive注解，在扩展点的实现类表示一个扩展类被获取到的的条件
                                                        Activate activate = clazz.getAnnotation(Activate.class);
                                                        if (activate != null) {
                                                            cachedActivates.put(names[0], activate);
                                                        }
                                                        for (String n : names) {
                                                            //从这来看貌似只有第1个name有效
                                                            if (! cachedNames.containsKey(clazz)) {
                                                                cachedNames.put(clazz, n);
                                                            }
                                                            //如若同一name已有对应class则提示异常
                                                            Class<?> c = extensionClasses.get(n);
                                                            if (c == null) {
                                                                extensionClasses.put(n, clazz);
                                                            } else if (c != clazz) {
                                                                throw new IllegalStateException("Duplicate extension " + type.getName() + " name " + n + " on " + c.getName() + " and " + clazz.getName());
                                                            }
                                                        }
                                                    }
                                                }
                                            }
                                        }
    //省略千万行
    }
```
META-INF/dubbo/internal/com.alibaba.dubbo.rpc.Protocol文件：
```language
filter=com.alibaba.dubbo.rpc.protocol.ProtocolFilterWrapper
listener=com.alibaba.dubbo.rpc.protocol.ProtocolListenerWrapper
mock=com.alibaba.dubbo.rpc.support.MockProtocol
```
至此完成根据type实例化对应ExtensionLoader实例（同时完成SPI文件解析，初始化同一type所有及其所有实现类class、 name等信息）

#### 1.2 ExtensionLoader.getAdaptiveExtension()方法
上面已分析完ExtensionLoader实例化过程，而获取具体扩展实例呢？注意上面分析时会判断class的@Adaptive注解，即是否反省的扩展实现，并赋值到ExtensionLoader实例的cachedAdaptiveClass 属性。
```language
   @SuppressWarnings("unchecked")
    public T getAdaptiveExtension() {
         //获取缺省扩展实现类实例(首次获取为null）
        Object instance = cachedAdaptiveInstance.get();
        if (instance == null) {
            if(createAdaptiveInstanceError == null) {
                synchronized (cachedAdaptiveInstance) {
                    instance = cachedAdaptiveInstance.get();
                    if (instance == null) {
                        try {
                            //创建缺省扩展实现类实例
                            instance = createAdaptiveExtension();
                            cachedAdaptiveInstance.set(instance);
                        } catch (Throwable t) {
                            createAdaptiveInstanceError = t;
                            throw new IllegalStateException("fail to create adaptive instance: " + t.toString(), t);
                        }
                    }
                }
            }
            else {
                throw new IllegalStateException("fail to create adaptive instance: " + createAdaptiveInstanceError.toString(), createAdaptiveInstanceError);
            }
        }

        return (T) instance;
    }

        @SuppressWarnings("unchecked")
    private T createAdaptiveExtension() {
        try {
           //1.获取@Adaptive对应的实现类class; 2.调用class.newInstance实例化；3.注入扩展实现injectExtension
            return injectExtension((T) getAdaptiveExtensionClass().newInstance());
        } catch (Exception e) {
            throw new IllegalStateException("Can not create adaptive extenstion " + type + ", cause: " + e.getMessage(), e);
        }
    }

    private Class<?> getAdaptiveExtensionClass() {
        //1.优先从cachedClasses获取所有扩展实现类信息；2.若为null则在上面有分析即解析SPI文件初始化
        getExtensionClasses();
        //获取缺省的扩展实现类，即若@Adaptive注解有指定，上一步的初始化会设置到cachedAdaptiveClass 
        if (cachedAdaptiveClass != null) {
            return cachedAdaptiveClass;
        }
       //无缺省adaptiveClass，需创建一个默认的AdaptiveExtensionClass；通过后面源码分析，此处是基于Javassit动态代理生成新的动态代理class并实例化，而不会实例化SPI扩展实现类的bean；因为没有使用@Adaptive指定其实并不知道实例化哪一个扩展实现类的bean对象
        return cachedAdaptiveClass = createAdaptiveExtensionClass();
    }
        
    private Class<?> createAdaptiveExtensionClass() {
        //这个方法眼前一亮：1.判断是否有方法使用注解 @Adaptive ；如若没有会抛出异常（每个扩展接口必须 有一个缺省实现）；2.如若有Method级别的@Adaptive注解，则会使用StringBuillder方式拼接一个java文件，
    	//如codeBuidler.append("\npublic class " + type.getSimpleName() + "$Adpative" + " implements " + type.getCanonicalName() + " {");--对应：public class Protocol$Adpative implements com.alibaba.dubbo.rpc.Protocol 
        String code = createAdaptiveExtensionClassCode();
        //获取类加载器 ExtensionLoader.class.getClassLoader()
        ClassLoader classLoader = findClassLoader();
        //根据dubbo SPI获取Compier实例 AdaptiveCompiler的实例
        com.alibaba.dubbo.common.compiler.Compiler compiler = ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.common.compiler.Compiler.class).getAdaptiveExtension();
         //调用AdaptiveCompiler、JavassistCompiler（父类AbstractCompiler）的compile方法对code进行基本检查、加载已有Protocol$Adpative的class文件、若没有则调用JavassistCompiler.doCompile(String name, String source)完成java文件的编译及类加载
        //JavassistCompiler的doCompile方法则涉及到javassist使用，基于CtClass创建化class对象，并调用其addInterface、addMethod（CtNewMethod）等，最终调用toClass组装生成class对象
        return compiler.compile(code, classLoader);
    }

```
createAdaptiveExtensionClass方法生成Protoco的AdaptiveExtensionClassCode，即完全拼接生成java代码：
```language
package com.alibaba.dubbo.rpc;

import com.alibaba.dubbo.common.extension.ExtensionLoader;

public class Protocol$Adpative implements com.alibaba.dubbo.rpc.Protocol {
	public void destroy() {
		throw new UnsupportedOperationException(
				"method public abstract void com.alibaba.dubbo.rpc.Protocol.destroy() of interface com.alibaba.dubbo.rpc.Protocol is not adaptive method!");
	}

	public int getDefaultPort() {
		throw new UnsupportedOperationException(
				"method public abstract int com.alibaba.dubbo.rpc.Protocol.getDefaultPort() of interface com.alibaba.dubbo.rpc.Protocol is not adaptive method!");
	}

	public com.alibaba.dubbo.rpc.Exporter export(com.alibaba.dubbo.rpc.Invoker arg0)
			throws com.alibaba.dubbo.rpc.Invoker {
		if (arg0 == null)
			throw new IllegalArgumentException("com.alibaba.dubbo.rpc.Invoker argument == null");
		if (arg0.getUrl() == null)
			throw new IllegalArgumentException("com.alibaba.dubbo.rpc.Invoker argument getUrl() == null");
		com.alibaba.dubbo.common.URL url = arg0.getUrl();
		String extName = (url.getProtocol() == null ? "dubbo" : url.getProtocol());
		if (extName == null)
			throw new IllegalStateException("Fail to get extension(com.alibaba.dubbo.rpc.Protocol) name from url("
					+ url.toString() + ") use keys([protocol])");
		com.alibaba.dubbo.rpc.Protocol extension = (com.alibaba.dubbo.rpc.Protocol) ExtensionLoader
				.getExtensionLoader(com.alibaba.dubbo.rpc.Protocol.class).getExtension(extName);
		return extension.export(arg0);
	}

	public com.alibaba.dubbo.rpc.Invoker refer(java.lang.Class arg0, com.alibaba.dubbo.common.URL arg1)
			throws java.lang.Class {
		if (arg1 == null)
			throw new IllegalArgumentException("url == null");
		com.alibaba.dubbo.common.URL url = arg1;
                //获取协议使名称
		String extName = (url.getProtocol() == null ? "dubbo" : url.getProtocol());
		if (extName == null)
			throw new IllegalStateException("Fail to get extension(com.alibaba.dubbo.rpc.Protocol) name from url("
					+ url.toString() + ") use keys([protocol])");
                //getExtensionLoader为根据协议名称获取对应的扩展实现类；getExtension则获取对应扩展实的实例
		com.alibaba.dubbo.rpc.Protocol extension = (com.alibaba.dubbo.rpc.Protocol) ExtensionLoader
				.getExtensionLoader(com.alibaba.dubbo.rpc.Protocol.class).getExtension(extName);
		return extension.refer(arg0, arg1);
	}
}
```
- JDK动态代理与Javassit代理均是动态生成class；但JDK动态代理有其局限，其是基于接口、及接口实现者实现生成动态代理代理class（https://www.cnblogs.com/lxyit/p/9272319.html），其底层使用DataOutputStream对象方法基于class文件结构完成手动组装生成class文件（相对Javassit性能较好，但需要对class文件结构及相关方法足够熟悉）。如Object o = Proxy.newProxyInstance(classLoader, interfaces, handler)，其中handler是InvocationHandler实现类，其通过invoke方法即实现了对原对象的代理，handler通过其构造方法将被代理对象（注意此处是对象）注入至handler实现。而newProxyInstance方法则最终基于被代理接口interfaces、handler（封装被代理对象及代理逻辑）动态生成interfacesr实现类，而在其对应方法调用handlerr的invoke方法完成代理逻辑及被代理对象自身逻辑的调用。
- Javassist是一个开源的分析、编辑和创建Java字节码的类库，相对简单其可直接使用java编码的形式生成动态代理class（但性能相比JDK动态代理差一些），可认为其是基于java源码结构生成class文件。
- 同上，涉及ProxyFactory动态代理类如：
```language
package com.alibaba.dubbo.rpc;

import com.alibaba.dubbo.common.extension.ExtensionLoader;

public class ProxyFactory$Adpative implements com.alibaba.dubbo.rpc.ProxyFactory {
	public java.lang.Object getProxy(com.alibaba.dubbo.rpc.Invoker arg0) throws com.alibaba.dubbo.rpc.Invoker {
		if (arg0 == null)
			throw new IllegalArgumentException("com.alibaba.dubbo.rpc.Invoker argument == null");
		if (arg0.getUrl() == null)
			throw new IllegalArgumentException("com.alibaba.dubbo.rpc.Invoker argument getUrl() == null");
		com.alibaba.dubbo.common.URL url = arg0.getUrl();
		String extName = url.getParameter("proxy", "javassist");
		if (extName == null)
			throw new IllegalStateException("Fail to get extension(com.alibaba.dubbo.rpc.ProxyFactory) name from url("
					+ url.toString() + ") use keys([proxy])");
		com.alibaba.dubbo.rpc.ProxyFactory extension = (com.alibaba.dubbo.rpc.ProxyFactory) ExtensionLoader
				.getExtensionLoader(com.alibaba.dubbo.rpc.ProxyFactory.class).getExtension(extName);
		return extension.getProxy(arg0);
	}

	public com.alibaba.dubbo.rpc.Invoker getInvoker(java.lang.Object arg0, java.lang.Class arg1,
			com.alibaba.dubbo.common.URL arg2) throws java.lang.Object {
		if (arg2 == null)
			throw new IllegalArgumentException("url == null");
		com.alibaba.dubbo.common.URL url = arg2;
		String extName = url.getParameter("proxy", "javassist");
		if (extName == null)
			throw new IllegalStateException("Fail to get extension(com.alibaba.dubbo.rpc.ProxyFactory) name from url("
					+ url.toString() + ") use keys([proxy])");
                //ProxyFactory SPI接口javassis对应实现类JavassistProxyFactory
		com.alibaba.dubbo.rpc.ProxyFactory extension = (com.alibaba.dubbo.rpc.ProxyFactory) ExtensionLoader
				.getExtensionLoader(com.alibaba.dubbo.rpc.ProxyFactory.class).getExtension(extName);
                //结合JavassistProxyFactory.getInvoker方法，返回实例为AbstractProxyInvoker的实例，其doInvoker方法可通过
		return extension.getInvoker(arg0, arg1, arg2);
	}
}
```
即根据上面示例分析，ServiceBean实例化会同步实例化基于Javassit动态代理生成的实例化对象：protocol（对应Proxy&Adpative） 、proxyFactory（对应ProxyFactory$Adpative）
```language
public class ServiceConfig<T> extends AbstractServiceConfig {

    private static final long   serialVersionUID = 3033787999037024738L;

    private static final Protocol protocol = ExtensionLoader.getExtensionLoader(Protocol.class).getAdaptiveExtension();
    
    private static final ProxyFactory proxyFactory = ExtensionLoader.getExtensionLoader(ProxyFactory.class).getAdaptiveExtension();
    ......	
}
```
回到之前分析URL注册时源码：
```language
            //源码为protocol.export中嵌套proxyFactory.getInvoker，但为方便调试此处调整为2行分别实现；其中proxyFactory对应对应ProxyFactory$Adpative
            Invoker  invoker = proxyFactory.getInvoker(ref, (Class) interfaceClass, local);
            Exporter<?> exporter = protocol.export(invoker);
            exporters.add(exporter);
```
其中根据Javassist生成的ProxyFactory$Adpative源码分析，代码重点在如下两行：1.获取ProxyFactory扩展实例extension（基于Dubbo SPI获取 JavassistProxyFactory实例）; 2.extension.getInvoker获取调用器invoker实例
```language
//ProxyFactory SPI接口javassis对应实现类JavassistProxyFactory; extName如若没有指定则默认为javassist
		com.alibaba.dubbo.rpc.ProxyFactory extension = (com.alibaba.dubbo.rpc.ProxyFactory) ExtensionLoader
				.getExtensionLoader(com.alibaba.dubbo.rpc.ProxyFactory.class).getExtension(extName);
                //结合JavassistProxyFactory.getInvoker方法，返回实例为AbstractProxyInvoker的实例，其doInvoker方法可通过
		return extension.getInvoker(arg0, arg1, arg2);
```
#### 1.2.1 调用器invoker实例化分析
根据上面分析，即对应JavassistProxyFactory.getInvoker方法,参数对应：
1. invoker : DubboExampleService1
2. interfaces : interface com.aoe.demo.rpc.dubbo.DubboExampleInterf1
3. url : injvm://127.0.0.1/com.aoe.demo.rpc.dubbo.DubboExampleInterf1?anyhost=true&application=rpc-server&default.timeout=1000&dubbo=2.5.3&interface=com.aoe.demo.rpc.dubbo.DubboExampleInterf1&methods=serviceProvider&pid=9424&revision=0.0.1-SNAPSHOT&side=provider&timestamp=1564390430345
```language
/**
 * JavaassistRpcProxyFactory 

 * @author william.liangf
 */
public class JavassistProxyFactory extends AbstractProxyFactory {

    @SuppressWarnings("unchecked")
    public <T> T getProxy(Invoker<T> invoker, Class<?>[] interfaces) {
        return (T) Proxy.getProxy(interfaces).newInstance(new InvokerInvocationHandler(invoker));
    }

    public <T> Invoker<T> getInvoker(T proxy, Class<T> type, URL url) {
        // TODO Wrapper类不能正确处理带$的类名
        final Wrapper wrapper = Wrapper.getWrapper(proxy.getClass().getName().indexOf('$') < 0 ? proxy.getClass() : type);
        return new AbstractProxyInvoker<T>(proxy, type, url) {
            @Override
            protected Object doInvoke(T proxy, String methodName, 
                                      Class<?>[] parameterTypes, 
                                      Object[] arguments) throws Throwable {
                return wrapper.invokeMethod(proxy, methodName, parameterTypes, arguments);
            }
        };
    }

}
```
##### 1.2.1.1 Wrapper动态代理包装对象分析（Wrapper.getWrapper）
Wrapper.getWrapper（Class clazz)方法对clazz初始判断后调用Wrapper.makeWrapper(Class clazz)生成包装类（内部使用封装了Javassisst的ClassGenerator来动态生成包装类class并实例化；为方便查看基于Javassist动态生成包装类，故在ClassGenerator方法中，基于CtClass.writeFile("d:/test") 可将class定向写入到指定目录，之后通过反编译工具反编译class可读性较好，如：
```language
package com.alibaba.dubbo.common.bytecode;

import com.aoe.demo.rpc.dubbo.DubboExampleService1;
import java.lang.reflect.InvocationTargetException;
import java.util.List;
import java.util.Map;

public class Wrapper1
  extends Wrapper
  implements ClassGenerator.DC
{
  public static String[] pns;
  public static Map pts;
  public static String[] mns;
  public static String[] dmns;
  public static Class[] mts0;
  
  public String[] getPropertyNames()
  {
    return pns;
  }
  
  public boolean hasProperty(String paramString)
  {
    return pts.containsKey(paramString);
  }
  
  public Class getPropertyType(String paramString)
  {
    return (Class)pts.get(paramString);
  }
  
  public String[] getMethodNames()
  {
    return mns;
  }
  
  public String[] getDeclaredMethodNames()
  {
    return dmns;
  }
  
  public void setPropertyValue(Object paramObject1, String paramString, Object paramObject2)
  {
    try
    {
      DubboExampleService1 localDubboExampleService1 = (DubboExampleService1)paramObject1;
    }
    catch (Throwable localThrowable)
    {
      throw new IllegalArgumentException(localThrowable);
    }
    throw new NoSuchPropertyException("Not found property \"" + paramString + "\" filed or setter method in class com.aoe.demo.rpc.dubbo.DubboExampleService1.");
  }
  
  public Object getPropertyValue(Object paramObject, String paramString)
  {
    try
    {
      DubboExampleService1 localDubboExampleService1 = (DubboExampleService1)paramObject;
    }
    catch (Throwable localThrowable)
    {
      throw new IllegalArgumentException(localThrowable);
    }
    throw new NoSuchPropertyException("Not found property \"" + paramString + "\" filed or setter method in class com.aoe.demo.rpc.dubbo.DubboExampleService1.");
  }
  
  public Object invokeMethod(Object paramObject, String paramString, Class[] paramArrayOfClass, Object[] paramArrayOfObject)
    throws InvocationTargetException
  {
    DubboExampleService1 localDubboExampleService1;
    try
    {
      localDubboExampleService1 = (DubboExampleService1)paramObject;
    }
    catch (Throwable localThrowable1)
    {
      throw new IllegalArgumentException(localThrowable1);
    }
    try
    {
      if ((!"serviceProvider".equals(paramString)) || (paramArrayOfClass.length == 1)) {
        return localDubboExampleService1.serviceProvider((List)paramArrayOfObject[0]);
      }
    }
    catch (Throwable localThrowable2)
    {
      throw new InvocationTargetException(localThrowable2);
    }
    throw new NoSuchMethodException("Not found method \"" + paramString + "\" in class com.aoe.demo.rpc.dubbo.DubboExampleService1.");
  }
}

```
##### 1.2.1.2 基于包装对象wrapper实例化Invoker对象
```language
return new AbstractProxyInvoker<T>(proxy, type, url) {
            @Override
            protected Object doInvoke(T proxy, String methodName, 
                                      Class<?>[] parameterTypes, 
                                      Object[] arguments) throws Throwable {
                return wrapper.invokeMethod(proxy, methodName, parameterTypes, arguments);
            }
        };
```
即实例化AbstractProxyInvoker类对象，doInvoke方法中重写wrapper的invokeMethod方法（其中proxy则应对发布的dubbo服务DubboExampleService1对象；基url、type也同步赋值给 invoker实例）；而wrapper的invokeMethod方法通过上面反编译的Wrapper1即可发现，其invokeMethod方法核心代码：
   //如若调用的方法为serviceProvider由调用localDubboExampleService1.serviceProvider方法
```language
     if ((!"serviceProvider".equals(paramString)) || (paramArrayOfClass.length == 1)) {
        return localDubboExampleService1.serviceProvider((List)paramArrayOfObject[0]);
      }
```
##### 1.2.1.3 invoker生成服务发布对象exporter
上一步基于proxyFactory.getInvoker生成invoker实例（AbstractProxyInvoker类），之后调用exporter = protocol.export(invoker)（即对应上在分析的Protocol&Adaptive类的export方法）：
```language
	public com.alibaba.dubbo.rpc.Exporter export(com.alibaba.dubbo.rpc.Invoker arg0)
			throws com.alibaba.dubbo.rpc.Invoker {
		if (arg0 == null)
			throw new IllegalArgumentException("com.alibaba.dubbo.rpc.Invoker argument == null");
		if (arg0.getUrl() == null)
			throw new IllegalArgumentException("com.alibaba.dubbo.rpc.Invoker argument getUrl() == null");
		com.alibaba.dubbo.common.URL url = arg0.getUrl();
		String extName = (url.getProtocol() == null ? "dubbo" : url.getProtocol());
		if (extName == null)
			throw new IllegalStateException("Fail to get extension(com.alibaba.dubbo.rpc.Protocol) name from url("
					+ url.toString() + ") use keys([protocol])");
		com.alibaba.dubbo.rpc.Protocol extension = (com.alibaba.dubbo.rpc.Protocol) ExtensionLoader
				.getExtensionLoader(com.alibaba.dubbo.rpc.Protocol.class).getExtension(extName);
		return extension.export(arg0);
	}
```
即获取Invoker对象Url信息，根据protocol获取Protocol实例（默认为DubboProtocol，可根据url的protocol动态获取），那么此处就涉及DubboProtocol.export（）方法（当前示例入参为Dubbo的Url实例）
```language
public class DubboProtocol extends AbstractProtocol {

    public static final String NAME = "dubbo";

    public static final String COMPATIBLE_CODEC_NAME = "dubbo1compatible";
    
    public static final int DEFAULT_PORT = 20880;
    
    public final ReentrantLock lock = new ReentrantLock();
    
    private final Map<String, ExchangeServer> serverMap = new ConcurrentHashMap<String, ExchangeServer>(); // <host:port,Exchanger>
    
    private final Map<String, ReferenceCountExchangeClient> referenceClientMap = new ConcurrentHashMap<String, ReferenceCountExchangeClient>(); // <host:port,Exchanger>
    
    private final ConcurrentMap<String, LazyConnectExchangeClient> ghostClientMap = new ConcurrentHashMap<String, LazyConnectExchangeClient>();
    
    //consumer side export a stub service for dispatching event
    //servicekey-stubmethods
    private final ConcurrentMap<String, String> stubServiceMethodsMap = new ConcurrentHashMap<String, String>();
    
    private static final String IS_CALLBACK_SERVICE_INVOKE = "_isCallBackServiceInvoke";
    
    //发布dubbo服务
    public <T> Exporter<T> export(Invoker<T> invoker) throws RpcException {
	/获取URL对象
        URL url = invoker.getUrl();
        
        // export service.
        //基于url的对应的serviceGroup、serviceName、serviceVersion、port组装生成服务唯一标识，如 com.aoe.demo.rpc.dubbo.DubboExampleInterf1:20890
        String key = serviceKey(url);
        //实例化DubboExporter并存储至exporterMap
        DubboExporter<T> exporter = new DubboExporter<T>(invoker, key, exporterMap);
        exporterMap.put(key, exporter);
        
        //export an stub service for dispaching event
       //stub为本地存根，用于在客户端做缓存、预处理、参数验证等
        Boolean isStubSupportEvent = url.getParameter(Constants.STUB_EVENT_KEY,Constants.DEFAULT_STUB_EVENT);
        //是否是回调服务
        Boolean isCallbackservice = url.getParameter(Constants.IS_CALLBACK_SERVICE, false);
        if (isStubSupportEvent && !isCallbackservice){
            //支持本地存根但非回调服务
            String stubServiceMethods = url.getParameter(Constants.STUB_EVENT_METHODS_KEY);
            if (stubServiceMethods == null || stubServiceMethods.length() == 0 ){
                if (logger.isWarnEnabled()){
                    logger.warn(new IllegalStateException("consumer [" +url.getParameter(Constants.INTERFACE_KEY) +
                            "], has set stubproxy support event ,but no stub methods founded."));
                }
            } else {
                //保存本地存根信息
                stubServiceMethodsMap.put(url.getServiceKey(), stubServiceMethods);
            }
        }
        //见如下方法
        openServer(url);
        
        return exporter;
    }
    
    private void openServer(URL url) {
        // find server.
        String key = url.getAddress();
        //client 也可以暴露一个只有server可以调用的服务。
        boolean isServer = url.getParameter(Constants.IS_SERVER_KEY,true);
        if (isServer) {
        	ExchangeServer server = serverMap.get(key);
        	if (server == null) {
                        //信息交换层（Exchange）：封装请求响应模式，以Request和Response为中心
        		serverMap.put(key, createServer(url));
        	} else {
        		//server支持reset,配合override功能使。
        		server.reset(url);
        	}
        }
    }
    
    private ExchangeServer createServer(URL url) {
        //默认开启server关闭时发送readonly事件
        url = url.addParameterIfAbsent(Constants.CHANNEL_READONLYEVENT_SENT_KEY, Boolean.TRUE.toString());
        //默认开启heartbeat
        url = url.addParameterIfAbsent(Constants.HEARTBEAT_KEY, String.valueOf(Constants.DEFAULT_HEARTBEAT));
        String str = url.getParameter(Constants.SERVER_KEY, Constants.DEFAULT_REMOTING_SERVER);

        if (str != null && str.length() > 0 && ! ExtensionLoader.getExtensionLoader(Transporter.class).hasExtension(str))
            throw new RpcException("Unsupported server type: " + str + ", url: " + url);

        url = url.addParameter(Constants.CODEC_KEY, Version.isCompatibleVersion() ? COMPATIBLE_CODEC_NAME : DubboCodec.NAME);
        ExchangeServer server;
        try {
            //根据url的exchange参数值实例化Exchanger（若exchange参数为空则实例化HeaderExchanger），并调用Exchanger的bind方法返回ExchangeServer（HeaderExchanger的bind方法返回HeaderExchangeServer）具体可往下看
            server = Exchangers.bind(url, requestHandler);
        } catch (RemotingException e) {
            throw new RpcException("Fail to start server(url: " + url + ") " + e.getMessage(), e);
        }
        str = url.getParameter(Constants.CLIENT_KEY);
        if (str != null && str.length() > 0) {
            Set<String> supportedTypes = ExtensionLoader.getExtensionLoader(Transporter.class).getSupportedExtensions();
            if (!supportedTypes.contains(str)) {
                throw new RpcException("Unsupported client type: " + str);
            }
        }
        return server;
    }
```
```language
/**
 * DefaultMessenger
 * 
 * @author william.liangf
 */
public class HeaderExchanger implements Exchanger {
    
    public static final String NAME = "header";

    public ExchangeClient connect(URL url, ExchangeHandler handler) throws RemotingException {
        return new HeaderExchangeClient(Transporters.connect(url, new DecodeHandler(new HeaderExchangeHandler(handler))));
    }

    public ExchangeServer bind(URL url, ExchangeHandler handler) throws RemotingException {
        return new HeaderExchangeServer(Transporters.bind(url, new DecodeHandler(new HeaderExchangeHandler(handler))));
    }

}
```
- 信息交换层（Exchange）：封装请求响应模式，同步转异步，以Request和Response为中心，扩展接口为Exchanger、ExchangeChannel、ExchangeClient和ExchangeServer。
- 网络传输层（Transport）：采用了SPI的扩展方式抽象mina和netty为统一接口，以Message为中心，扩展接口为Channel、Transporter、Client、Server和Codec。比如netty，mian，grizzly等，默认采用netty方式。
- 怎么理解呢？个人理解服务发布主要做如下几点：
1. 基于Url信息动态代理生成Wrapper实例，并使用Dubbo实例bean对象、Url、Wrapper实例实例化Invoker实例（JavassistProxyFactory.getInvoker）；
2. 基于invoker实例化DubboExpoter实例（默认为Dubbo方式发布服务）并保存至exporterMap（即根据url信息集成的key可最终关联到实例Dubbo服务及方法）
3. 这点如何理解 ？根据之前分析hessian、SpirngMVC源码来看，前面2点是Service服务信息初始化；但是，涉及到消费者、发布者之前接口调用、信息转换、网络传输呢？这个就应该与上面提到的Exchange、Transport相关了。所以这一步在DubboProtocol中会同样根据 Url实例实例化Server



### Dubbo SPI之Protocol


  