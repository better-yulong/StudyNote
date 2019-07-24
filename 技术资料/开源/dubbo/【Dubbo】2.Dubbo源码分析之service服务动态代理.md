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
                            //获取缺省扩展实现类实例
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
       
        return cachedAdaptiveClass = createAdaptiveExtensionClass();
    }

        
    private Class<?> createAdaptiveExtensionClass() {
        String code = createAdaptiveExtensionClassCode();
        ClassLoader classLoader = findClassLoader();
        com.alibaba.dubbo.common.compiler.Compiler compiler = ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.common.compiler.Compiler.class).getAdaptiveExtension();
        return compiler.compile(code, classLoader);
    }

```














ExtensionLoader.getExtensionLoader(ProxyFactory.class).getAdaptiveExtension()


### Dubbo SPI之Protocol


  