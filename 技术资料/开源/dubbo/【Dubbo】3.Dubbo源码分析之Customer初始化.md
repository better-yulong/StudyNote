Dubbo Customer端dubbo xml配置
```language
	<dubbo:application name="rpc-client" />
    <dubbo:registry address="zookeeper://127.0.0.1:2181" />
	<dubbo:reference id="dubboExampleService1" interface="com.aoe.demo.rpc.dubbo.DubboExampleInterf1" />
```

### 一、dubbo:reference标签解析
```language
    //element参数：[async="false", id="dubboExampleService1", interface="com.aoe.demo.rpc.dubbo.DubboExampleInterf1", timeout="0", version="0.0.0"]
    //beanClass参数:class com.alibaba.dubbo.config.spring.ReferenceBean
    @SuppressWarnings("unchecked")
    private static BeanDefinition parse(Element element, ParserContext parserContext, Class<?> beanClass, boolean required) {
        //实例化默认的beanDefinition 
        RootBeanDefinition beanDefinition = new RootBeanDefinition();
        //设置beanDefinition对应的BeanClass为ReferenceBean
        beanDefinition.setBeanClass(beanClass);
       //设置延迟初始化为false
        beanDefinition.setLazyInit(false);
       //获取dubbo:reference id="dubboExampleService1"
        String id = element.getAttribute("id");
         
       //忽略N行代码
    
        if (id != null && id.length() > 0) {
            if (parserContext.getRegistry().containsBeanDefinition(id))  {
        		throw new IllegalStateException("Duplicate spring bean id " + id);
            }
            //根据id:dubboExampleService及beanDefinition，将其注册到上下文（此处为Spring）的registry
            parserContext.getRegistry().registerBeanDefinition(id, beanDefinition);
            //为beanDefinition添加id属性及值
            beanDefinition.getPropertyValues().addPropertyValue("id", id);
        }
        if (ProtocolConfig.class.equals(beanClass)) {
            //分支处理ProtocolConfig
            for (String name : parserContext.getRegistry().getBeanDefinitionNames()) {
                BeanDefinition definition = parserContext.getRegistry().getBeanDefinition(name);
                PropertyValue property = definition.getPropertyValues().getPropertyValue("protocol");
                if (property != null) {
                    Object value = property.getValue();
                    if (value instanceof ProtocolConfig && id.equals(((ProtocolConfig) value).getName())) {
                        definition.getPropertyValues().addPropertyValue("protocol", new RuntimeBeanReference(id));
                    }
                }
            }
        } else if (ServiceBean.class.equals(beanClass)) {
            //分支处理ServiceBean
            String className = element.getAttribute("class");
            if(className != null && className.length() > 0) {
                RootBeanDefinition classDefinition = new RootBeanDefinition();
                classDefinition.setBeanClass(ReflectUtils.forName(className));
                classDefinition.setLazyInit(false);
                parseProperties(element.getChildNodes(), classDefinition);
                beanDefinition.getPropertyValues().addPropertyValue("ref", new BeanDefinitionHolder(classDefinition, id + "Impl"));
            }
        } else if (ProviderConfig.class.equals(beanClass)) {
            //分支处理ProviderConfig
            parseNested(element, parserContext, ServiceBean.class, true, "service", "provider", id, beanDefinition);
        } else if (ConsumerConfig.class.equals(beanClass)) {
           //分支处理ConsumerConfig
            parseNested(element, parserContext, ReferenceBean.class, false, "reference", "consumer", id, beanDefinition);
        }

	//各dubbo标签公共属性解析、验证及处理
        Set<String> props = new HashSet<String>();
        ManagedMap parameters = null;
        
        for (Method setter : beanClass.getMethods()) {
            //遍历beanClass对应的方法，根据set方法验证get、is方法；并针对完成标签解析（子标签解析）获取属性值，添加至beanDefinition。以上面dubbo:reference为例分析
            //比如setter方法：setInterface(String interfaceName)
            String name = setter.getName();
            if (name.length() > 3 && name.startsWith("set")
                    && Modifier.isPublic(setter.getModifiers())
                    && setter.getParameterTypes().length == 1) {
                //获取setInterface第一个参数对应的类型Class，即String
                Class<?> type = setter.getParameterTypes()[0];
                //根据方法名转换出标签的属性值（如属性首字母小写转换、多单词-分隔）
                String property = StringUtils.camelToSplitName(name.substring(3, 4).toLowerCase() + name.substring(4), "-");
                //添加property至props集合
                props.add(property);
                //获取getter（或者is）方法并验证
                Method getter = null;
                try {
                    getter = beanClass.getMethod("get" + name.substring(3), new Class<?>[0]);
                } catch (NoSuchMethodException e) {
                    try {
                        getter = beanClass.getMethod("is" + name.substring(3), new Class<?>[0]);
                    } catch (NoSuchMethodException e2) {
                    }
                }
                if (getter == null 
                        || ! Modifier.isPublic(getter.getModifiers())
                        || ! type.equals(getter.getReturnType())) {
                    //确认getter方法、修改符、返回值与set是参数类型是否匹配，type对应set的参数String
                    continue;
                }
                if ("parameters".equals(property)) {
                    parameters = parseParameters(element.getChildNodes(), beanDefinition);
                } else if ("methods".equals(property)) {
                    parseMethods(id, element.getChildNodes(), beanDefinition, parserContext);
                } else if ("arguments".equals(property)) {
                    parseArguments(id, element.getChildNodes(), beanDefinition, parserContext);
                } else {
                    //上面是根据proterty做针对性处理；而interface则会直接运行至此。获取interface属性对应的值：即com.aoe.demo.rpc.dubbo.DubboExampleInterf1
                    String value = element.getAttribute(property);
                    if (value != null) {
                    	value = value.trim();
                    	if (value.length() > 0) {
                    		if ("registry".equals(property) && RegistryConfig.NO_AVAILABLE.equalsIgnoreCase(value)) {
                                //属性包含registry且为N/A：即实例化registryConfig ，且设置其为注册中心不可用，那不会将dubbo服务注册到zk；此种用于直连，需要客户端指定address等来连接服务端
                            	RegistryConfig registryConfig = new RegistryConfig();
                            	registryConfig.setAddress(RegistryConfig.NO_AVAILABLE);
                            	beanDefinition.getPropertyValues().addPropertyValue(property, registryConfig);
                            } else if ("registry".equals(property) && value.indexOf(',') != -1) {
                                 //属性配置包含多个registry id，则拆分后组装RuntimeBeanReference列表并添加至beanDefinition
                    		parseMultiRef("registries", value, beanDefinition, parserContext);
                            } else if ("provider".equals(property) && value.indexOf(',') != -1) {
  				//属性配置包含多个providerid，则拆分后组装RuntimeBeanReference列表并添加至beanDefinition
                            	parseMultiRef("providers", value, beanDefinition, parserContext);
                            } else if ("protocol".equals(property) && value.indexOf(',') != -1) {
                                //属性配置包含多个protocol，则拆分后组装RuntimeBeanReference列表并添加至beanDefinition
                                parseMultiRef("protocols", value, beanDefinition, parserContext);
                            } else {
                                //同上，如果属性为
                                Object reference;
                               //当前示例type即为setInterface(String interfaceName)的参数String，其是Primitive类型
                                if (isPrimitive(type)) {
                                    if ("async".equals(property) && "false".equals(value)
                                            || "timeout".equals(property) && "0".equals(value)
                                            || "delay".equals(property) && "0".equals(value)
                                            || "version".equals(property) && "0.0.0".equals(value)
                                            || "stat".equals(property) && "-1".equals(value)
                                            || "reliable".equals(property) && "false".equals(value)) {
                                        // 兼容旧版本xsd中的default值
                                        value = null;
                                    }
                                    //将value值com.aoe.demo.rpc.dubbo.DubboExampleInterf1赋值给reference 
                                    reference = value;
                                } else if ("protocol".equals(property) 
                                        && ExtensionLoader.getExtensionLoader(Protocol.class).hasExtension(value)
                                        && (! parserContext.getRegistry().containsBeanDefinition(value)
                                                || ! ProtocolConfig.class.getName().equals(parserContext.getRegistry().getBeanDefinition(value).getBeanClassName()))) {
                                    if ("dubbo:provider".equals(element.getTagName())) {
                                        logger.warn("Recommended replace <dubbo:provider protocol=\"" + value + "\" ... /> to <dubbo:protocol name=\"" + value + "\" ... />");
                                    }
                                    // 兼容旧版本配置
                                    ProtocolConfig protocol = new ProtocolConfig();
                                    protocol.setName(value);
                                    reference = protocol;
                                } else if ("monitor".equals(property) 
                                        && (! parserContext.getRegistry().containsBeanDefinition(value)
                                                || ! MonitorConfig.class.getName().equals(parserContext.getRegistry().getBeanDefinition(value).getBeanClassName()))) {
                                    // 兼容旧版本配置
                                    reference = convertMonitor(value);
                                } else if ("onreturn".equals(property)) {
                                    int index = value.lastIndexOf(".");
                                    String returnRef = value.substring(0, index);
                                    String returnMethod = value.substring(index + 1);
                                    reference = new RuntimeBeanReference(returnRef);
                                    beanDefinition.getPropertyValues().addPropertyValue("onreturnMethod", returnMethod);
                                } else if ("onthrow".equals(property)) {
                                    int index = value.lastIndexOf(".");
                                    String throwRef = value.substring(0, index);
                                    String throwMethod = value.substring(index + 1);
                                    reference = new RuntimeBeanReference(throwRef);
                                    beanDefinition.getPropertyValues().addPropertyValue("onthrowMethod", throwMethod);
                                } else {
                                    if ("ref".equals(property) && parserContext.getRegistry().containsBeanDefinition(value)) {
                                        BeanDefinition refBean = parserContext.getRegistry().getBeanDefinition(value);
                                        if (! refBean.isSingleton()) {
                                            throw new IllegalStateException("The exported service ref " + value + " must be singleton! Please set the " + value + " bean scope to singleton, eg: <bean id=\"" + value+ "\" scope=\"singleton\" ...>");
                                        }
                                    }
                                   
                                    reference = new RuntimeBeanReference(value);
                                }
                      //将上面isPrimitive判断处实例化的reference 添加至beanDefinition
		      beanDefinition.getPropertyValues().addPropertyValue(property, reference);
                            }
                    	}
                    }
                }
            }
        }
        
        //兼容前面未解析到的属性并同步添加至beanDefinition
        NamedNodeMap attributes = element.getAttributes();
        int len = attributes.getLength();
        for (int i = 0; i < len; i++) {
            Node node = attributes.item(i);
            String name = node.getLocalName();
            if (! props.contains(name)) {
                if (parameters == null) {
                    parameters = new ManagedMap();
                }
                String value = node.getNodeValue();
                parameters.put(name, new TypedStringValue(value, String.class));
            }
        }
        if (parameters != null) {
            beanDefinition.getPropertyValues().addPropertyValue("parameters", parameters);
        }
        return beanDefinition;
    }

```
至此，dubbo:reference标签解析生成ReferenceBean对应的BeanDefinition对象分析完毕。

### 二、ReferenceBean实例化分析
```language
public class ReferenceBean<T> extends ReferenceConfig<T> implements FactoryBean, ApplicationContextAware, InitializingBean, DisposableBean {}
```
#### 2.1 ApplicationContextAware 接口
之前有分析过，此处不做过多分析，即ReferenceBean实例化后，因其实现ApplicationContextAware 故会将当前ApplicationContext设置到ReferenceBean实例化（即setApplicationContext方法）
#### 2.2 DisposableBean接口
可用于Spring容器关闭时销毁bean时调用
#### 2.3 InitializingBean接口
之前也有分析，其afterPropertiesSet方法可用于在Spring初始化ReferenceBean实例原始bean之后调用完成自定义的初始化工作。类似于之前分析ServiceBean，主要用于对ReferenceBean实例bean的consumer、application、module、registry、monitor等属性的赋值
#### 2.3 FactoryBean接口
```language
    //FactoryBean
    public Object getObject() throws Exception {
        return get();
    }

    public Class<?> getObjectType() {
        return getInterfaceClass();
    }
  
    @Parameter(excluded = true)
    public boolean isSingleton() {
        return true;
    }
```
```language
  //ReferenceConfig
  	public Class<?> getInterfaceClass() {
	    if (interfaceClass != null) {
	        return interfaceClass;
	    }
	    if (isGeneric()
            || (getConsumer() != null && getConsumer().isGeneric())) {
	        return GenericService.class;
	    }
	    try {
	        if (interfaceName != null && interfaceName.length() > 0) {
	            this.interfaceClass = Class.forName(interfaceName, true, Thread.currentThread()
                    .getContextClassLoader());
	        }
        } catch (ClassNotFoundException t) {
            throw new IllegalStateException(t.getMessage(), t);
        }
	    return interfaceClass;
    }

    public synchronized T get() {
        if (destroyed){
            throw new IllegalStateException("Already destroyed!");
        }
    	if (ref == null) {
    		init();
    	}
    	return ref;
    }

    private void init() {
	if (initialized) {
	    return;
	}
	initialized = true;
    	if (interfaceName == null || interfaceName.length() == 0) {
    	    throw new IllegalStateException("<dubbo:reference interface=\"\" /> interface not allow null!");
    	}
    	// 获取消费者全局配置(当前示例未配置)
    	checkDefault();//实例化默认ConsumerConfig（即如若未在xml中显示配置dubbo:consumer标签则初始化默认对象），并通过反射调用set方法设置ConsumerConfig及其父类AbstractReferenceConfig属性值（基于System.getProterty方式），如check（检查服务提供者是否存在 ）、lazy（延迟创建连接）、reconnect（重连）、stubevent、version、group等
        appendProperties(this);////通过反射调用set方法设置ReferenceConfig及其父类AbstractReferenceConfig属性值（基于System.getProterty方式），如check（检查服务提供者是否存在 ）、lazy（延迟创建连接）、reconnect（重连）、stubevent、version、group等
        if (getGeneric() == null && getConsumer() != null) {
            setGeneric(getConsumer().getGeneric());
        }
        if (ProtocolUtils.isGeneric(getGeneric())) {
            interfaceClass = GenericService.class;
        } else {
            try {
			interfaceClass = Class.forName(interfaceName, true, Thread.currentThread().getContextClassLoader());
			} catch (ClassNotFoundException e) {
				throw new IllegalStateException(e.getMessage(), e);
			}
           //检查interfaceClass是否为null且是否为接口；若methods（即MethodConfig）不为空则检查method是否存在（对应<dubbo:method>）
            checkInterfaceAndMethods(interfaceClass, methods);
        }
        //查找并获取接口对应resolve配置，即属性配置文件（-Ddubbo.resolve.file指定映射文件路径，此配置优先级高于<dubbo:reference>中的配置）
        String resolve = System.getProperty(interfaceName);
        String resolveFile = null;
        if (resolve == null || resolve.length() == 0) {
	        resolveFile = System.getProperty("dubbo.resolve.file");
	        if (resolveFile == null || resolveFile.length() == 0) {
                        //如获取File（2.0以上版本自动加载${user.home}/dubbo-resolve.properties文件不需要配置）
	        	File userResolveFile = new File(new File(System.getProperty("user.home")), "dubbo-resolve.properties");
	        	if (userResolveFile.exists()) {
	        		resolveFile = userResolveFile.getAbsolutePath();
	        	}
	        }
	        if (resolveFile != null && resolveFile.length() > 0) {
                        //获取到dubbo-resolve.properties，加载解析
	        	Properties properties = new Properties();
	        	FileInputStream fis = null;
	        	try {
	        	    fis = new FileInputStream(new File(resolveFile));
					properties.load(fis);
				} catch (IOException e) {
					throw new IllegalStateException("Unload " + resolveFile + ", cause: " + e.getMessage(), e);
				} finally {
				    try {
                        if(null != fis) fis.close();
                    } catch (IOException e) {
                        logger.warn(e.getMessage(), e);
                    }
				}
	        	resolve = properties.getProperty(interfaceName);
	        }
        }
        if (resolve != null && resolve.length() > 0) {
        	url = resolve;
        	if (logger.isWarnEnabled()) {
        		if (resolveFile != null && resolveFile.length() > 0) {
        			logger.warn("Using default dubbo resolve file " + resolveFile + " replace " + interfaceName + "" + resolve + " to p2p invoke remote service.");
        		} else {
        			logger.warn("Using -D" + interfaceName + "=" + resolve + " to p2p invoke remote service.");
        		}
    		}
        }
        //这部分同dubbo:service，如若当前未显示指定application、module、registries、monitor）则使用consumer等赋值，涉及根据优先级多次赋值
        if (consumer != null) {
            if (application == null) {
                application = consumer.getApplication();
            }
            if (module == null) {
                module = consumer.getModule();
            }
            if (registries == null) {
                registries = consumer.getRegistries();
            }
            if (monitor == null) {
                monitor = consumer.getMonitor();
            }
        }
        if (module != null) {
            if (registries == null) {
                registries = module.getRegistries();
            }
            if (monitor == null) {
                monitor = module.getMonitor();
            }
        }
        if (application != null) {
            if (registries == null) {
                registries = application.getRegistries();
            }
            if (monitor == null) {
                monitor = application.getMonitor();
            }
        }
        //检查application，同时根据系统属自自定义变量或者dubbo.properties.file对应的属性文件值多次赋值更新（
        checkApplication();
        //获取并检查local、stub、mock； 涉及使用Javassist加载localClass并验证class合法性
        checkStubAndMock(interfaceClass);
        Map<String, String> map = new HashMap<String, String>();
        Map<Object, Object> attributes = new HashMap<Object, Object>();
        map.put(Constants.SIDE_KEY, Constants.CONSUMER_SIDE);
        map.put(Constants.DUBBO_VERSION_KEY, Version.getVersion());
        map.put(Constants.TIMESTAMP_KEY, String.valueOf(System.currentTimeMillis()));
        if (ConfigUtils.getPid() > 0) {
            map.put(Constants.PID_KEY, String.valueOf(ConfigUtils.getPid()));
        }
        if (! isGeneric()) {
            String revision = Version.getVersion(interfaceClass, version);
            if (revision != null && revision.length() > 0) {
                map.put("revision", revision);
            }

            String[] methods = Wrapper.getWrapper(interfaceClass).getMethodNames();
            if(methods.length == 0) {
                logger.warn("NO method found in service interface " + interfaceClass.getName());
                map.put("methods", Constants.ANY_VALUE);
            }
            else {
                map.put("methods", StringUtils.join(new HashSet<String>(Arrays.asList(methods)), ","));
            }
        }

        //整合各属性配置到map
        map.put(Constants.INTERFACE_KEY, interfaceName);
        appendParameters(map, application);
        appendParameters(map, module);
        appendParameters(map, consumer, Constants.DEFAULT_KEY);
        appendParameters(map, this);
        String prifix = StringUtils.getServiceKey(map);
        if (methods != null && methods.size() > 0) {
            for (MethodConfig method : methods) {
                appendParameters(map, method, method.getName());
                String retryKey = method.getName() + ".retry";
                if (map.containsKey(retryKey)) {
                    String retryValue = map.remove(retryKey);
                    if ("false".equals(retryValue)) {
                        map.put(method.getName() + ".retries", "0");
                    }
                }
                appendAttributes(attributes, method, prifix + "." + method.getName());
                checkAndConvertImplicitConfig(method, map, attributes);
            }
        }
        //attributes通过系统context进行存储.
        StaticContext.getSystemContext().putAll(attributes);
        ref = createProxy(map);//上面已将所有属性整合至map，生成代理类
    }

    	@SuppressWarnings({ "unchecked", "rawtypes", "deprecation" })
	private T createProxy(Map<String, String> map) {
		URL tmpUrl = new URL("temp", "localhost", 0, map);
		final boolean isJvmRefer;
        if (isInjvm() == null) {
            //未显示指定暴本地服务则仅暴露远程服务
            if (url != null && url.length() > 0) { //指定URL的情况下（url对应点对点直连配置），不做本地引用
                isJvmRefer = false;
            } else if (InjvmProtocol.getInjvmProtocol().isInjvmRefer(tmpUrl)) {
                //默认情况下如果本地有服务暴露，则引用本地服务.
                isJvmRefer = true;
            } else {
                isJvmRefer = false;
            }
        } else {
            isJvmRefer = isInjvm().booleanValue();
        }
		
		if (isJvmRefer) {
			URL url = new URL(Constants.LOCAL_PROTOCOL, NetUtils.LOCALHOST, 0, interfaceClass.getName()).addParameters(map);
			invoker = refprotocol.refer(interfaceClass, url);
            if (logger.isInfoEnabled()) {
                logger.info("Using injvm service " + interfaceClass.getName());
            }
		} else {
            if (url != null && url.length() > 0) { // 用户指定URL，指定的URL可能是对点对直连地址，也可能是注册中心URL
                String[] us = Constants.SEMICOLON_SPLIT_PATTERN.split(url);
                if (us != null && us.length > 0) {
                    for (String u : us) {
                        URL url = URL.valueOf(u);
                        if (url.getPath() == null || url.getPath().length() == 0) {
                            url = url.setPath(interfaceName);
                        }
                        if (Constants.REGISTRY_PROTOCOL.equals(url.getProtocol())) {
                            urls.add(url.addParameterAndEncoded(Constants.REFER_KEY, StringUtils.toQueryString(map)));
                        } else {
                            urls.add(ClusterUtils.mergeUrl(url, map));
                        }
                    }
                }
            } else { // 通过注册中心配置拼装URL
                //如us：[registry://127.0.0.1:2181/com.alibaba.dubbo.registry.RegistryService?application=rpc-client&dubbo=2.5.3&pid=5660&registry=zookeeper&timestamp=1565169709392]
            	List<URL> us = loadRegistries(false);
            	if (us != null && us.size() > 0) {
                	for (URL u : us) {
                	    URL monitorUrl = loadMonitor(u);
                        if (monitorUrl != null) {
                            map.put(Constants.MONITOR_KEY, URL.encode(monitorUrl.toFullString()));
                        }
                        //根据全量配置map与us拼装完整urls：[registry://127.0.0.1:2181/com.alibaba.dubbo.registry.RegistryService?application=rpc-client&dubbo=2.5.3&pid=5660&refer=application%3Drpc-client%26default.group%3Drpc-demo%26default.version%3D1.0.1-aoe%26dubbo%3D2.5.3%26interface%3Dcom.aoe.demo.rpc.dubbo.DubboExampleInterf1%26methods%3DserviceProvider%26pid%3D5660%26revision%3D0.0.1-SNAPSHOT%26side%3Dconsumer%26timestamp%3D1565169597733&registry=zookeeper&timestamp=1565169709392]
                	 urls.add(u.addParameterAndEncoded(Constants.REFER_KEY, StringUtils.toQueryString(map)));
                    }
            	}
            	if (urls == null || urls.size() == 0) {
                    throw new IllegalStateException("No such any registry to reference " + interfaceName  + " on the consumer " + NetUtils.getLocalHost() + " use dubbo version " + Version.getVersion() + ", please config <dubbo:registry address=\"...\" /> to your spring config.");
                }
            }

            if (urls.size() == 1) {
                //interface com.aoe.demo.rpc.dubbo.DubboExampleInterf1
                //此处调用根据urls调用的RegistryProtocol.refer方法：其中会实例化ZookeeperRegistry实例、根据接口名称和url组装 RegistryDirectory对象（设置客户端protocol等）
                //重点1：ZookeeperRegistry.register（完整url）--实际对应其父类FailbackRegistry的registeryyif 
                //重点2 directory.subscribe：consumer://100.119.69.10/com.aoe.demo.rpc.dubbo.DubboExampleInterf1?application=rpc-client&default.group=rpc-demo&default.version=1.0.1-aoe&dubbo=2.5.3&interface=com.aoe.demo.rpc.dubbo.DubboExampleInterf1&methods=serviceProvider&pid=5660&revision=0.0.1-SNAPSHOT&side=consumer&timestamp=1565169597733
                //重点3：添加至cluster
                invoker = refprotocol.refer(interfaceClass, urls.get(0));
            } else {
                List<Invoker<?>> invokers = new ArrayList<Invoker<?>>();
                URL registryURL = null;
                for (URL url : urls) {
                    invokers.add(refprotocol.refer(interfaceClass, url));
                    if (Constants.REGISTRY_PROTOCOL.equals(url.getProtocol())) {
                        registryURL = url; // 用了最后一个registry url
                    }
                }
                if (registryURL != null) { // 有 注册中心协议的URL
                    // 对有注册中心的Cluster 只用 AvailableCluster
                    URL u = registryURL.addParameter(Constants.CLUSTER_KEY, AvailableCluster.NAME); 
                    invoker = cluster.join(new StaticDirectory(u, invokers));
                }  else { // 不是 注册中心的URL
                    invoker = cluster.join(new StaticDirectory(invokers));
                }
            }
        }

        Boolean c = check;
        if (c == null && consumer != null) {
            c = consumer.isCheck();
        }
        if (c == null) {
            c = true; // default true
        }
        if (c && ! invoker.isAvailable()) {
            throw new IllegalStateException("Failed to check the status of the service " + interfaceName + ". No provider available for the service " + (group == null ? "" : group + "/") + interfaceName + (version == null ? "" : ":" + version) + " from the url " + invoker.getUrl() + " to the consumer " + NetUtils.getLocalHost() + " use dubbo version " + Version.getVersion());
        }
        if (logger.isInfoEnabled()) {
            logger.info("Refer dubbo service " + interfaceClass.getName() + " from url " + invoker.getUrl());
        }
        // 创建服务代理
        return (T) proxyFactory.getProxy(invoker);
    }
```


### 自定义变量示例
- 在分析dubbo:reference标签解析对ReferenceConfig实例化时，若未显示在为其配置dubbo:consumer标签属性，则会默认为ReferenceConfig实例的属性consumer属性实例化默认的ConsumerConfig实例（包含如lazy、timeout、reconnect、version、group等），并尝试从系统参数获取对应配置的值通过method.invoke方式反射为ConsumerConfig实例的属性完成赋值，下面即简单对此种方式做下示例（参考：https://www.cnblogs.com/yangmingke/p/6058898.html）

#### 基于Eclipse+Tomcat方式设置属性以便可通过System.getProperty（“XXX”）获取自定义变量
1. 打开tomcat server（双击对应server），会显示常用的tomcat 端口号、启动或停止超时时间设置。
2. General Information中点击Open launch configuration（就面窗口而左上方），可打开"Edit Configuration"窗口。
3. 选择Arguments，在VM arguments输入框显示：
```language
-Dcatalina.base="D:\work\webcontainer\tomcat7" -Dcatalina.home="D:\work\webcontainer\tomcat7" -Dwtp.deploy="D:\work\webcontainer\tomcat7\wtpwebapps" -Djava.endorsed.dirs="D:\work\webcontainer\tomcat7\endorsed" -Ddubbo.consumer.version="1.0.1-aoe" -Ddubbo.consumer.group="rpc-demo"
```
如上在VM arguments后追加-DXXX=****(-D不能省略)，这样就可以通过 System.getProperty（“XXX”）获取****了；同样也可在catalina.bat（catalina.sh）通过JAVA_OPTS来添加自定义变量