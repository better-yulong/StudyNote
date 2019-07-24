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
#### 1.1 ExtensionLoader
##### 1.1.1 ExtensionLoader实例化
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
        Map<String, Class<?>> classes = cachedClasses.get();
        if (classes == null) {
            synchronized (cachedClasses) {
                classes = cachedClasses.get();
                if (classes == null) {
                    classes = loadExtensionClasses();
                    cachedClasses.set(classes);
                }
            }
        }
        return classes;
	}

    // 此方法已经getExtensionClasses方法同步过。
    private Map<String, Class<?>> loadExtensionClasses() {
        final SPI defaultAnnotation = type.getAnnotation(SPI.class);
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
        loadFile(extensionClasses, DUBBO_INTERNAL_DIRECTORY);
        loadFile(extensionClasses, DUBBO_DIRECTORY);
        loadFile(extensionClasses, SERVICES_DIRECTORY);
        return extensionClasses;
    }
    
    private void loadFile(Map<String, Class<?>> extensionClasses, String dir) {
        String fileName = dir + type.getName();
        try {
            Enumeration<java.net.URL> urls;
            ClassLoader classLoader = findClassLoader();
            if (classLoader != null) {
                urls = classLoader.getResources(fileName);
            } else {
                urls = ClassLoader.getSystemResources(fileName);
            }
            if (urls != null) {
                while (urls.hasMoreElements()) {
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
                                        String name = null;
                                        int i = line.indexOf('=');
                                        if (i > 0) {
                                            name = line.substring(0, i).trim();
                                            line = line.substring(i + 1).trim();
                                        }
                                        if (line.length() > 0) {
                                            Class<?> clazz = Class.forName(line, true, classLoader);
                                            if (! type.isAssignableFrom(clazz)) {
                                                throw new IllegalStateException("Error when load extension class(interface: " +
                                                        type + ", class line: " + clazz.getName() + "), class " 
                                                        + clazz.getName() + "is not subtype of interface.");
                                            }
                                            if (clazz.isAnnotationPresent(Adaptive.class)) {
                                                if(cachedAdaptiveClass == null) {
                                                    cachedAdaptiveClass = clazz;
                                                } else if (! cachedAdaptiveClass.equals(clazz)) {
                                                    throw new IllegalStateException("More than 1 adaptive class found: "
                                                            + cachedAdaptiveClass.getClass().getName()
                                                            + ", " + clazz.getClass().getName());
                                                }
                                            } else {
                                                try {
                                                    clazz.getConstructor(type);
                                                    Set<Class<?>> wrappers = cachedWrapperClasses;
                                                    if (wrappers == null) {
                                                        cachedWrapperClasses = new ConcurrentHashSet<Class<?>>();
                                                        wrappers = cachedWrapperClasses;
                                                    }
                                                    wrappers.add(clazz);
                                                } catch (NoSuchMethodException e) {
                                                    clazz.getConstructor();
                                                    if (name == null || name.length() == 0) {
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
                                                    String[] names = NAME_SEPARATOR.split(name);
                                                    if (names != null && names.length > 0) {
                                                        Activate activate = clazz.getAnnotation(Activate.class);
                                                        if (activate != null) {
                                                            cachedActivates.put(names[0], activate);
                                                        }
                                                        for (String n : names) {
                                                            if (! cachedNames.containsKey(clazz)) {
                                                                cachedNames.put(clazz, n);
                                                            }
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
                                    } catch (Throwable t) {
                                        IllegalStateException e = new IllegalStateException("Failed to load extension class(interface: " + type + ", class line: " + line + ") in " + url + ", cause: " + t.getMessage(), t);
                                        exceptions.put(line, e);
                                    }
                                }
                            } // end of while read lines
                        } finally {
                            reader.close();
                        }
                    } catch (Throwable t) {
                        logger.error("Exception when load extension class(interface: " +
                                            type + ", class file: " + url + ") in " + url, t);
                    }
                } // end of while urls
            }
        } catch (Throwable t) {
            logger.error("Exception when load extension class(interface: " +
                    type + ", description file: " + fileName + ").", t);
        }
    }
```


### Dubbo SPI之Protocol


