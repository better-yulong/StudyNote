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
#### 2.4 FactoryBean接口
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
                //重点1：ZookeeperRegistry.register（完整url对象）--实际对应其父类FailbackRegistry的register方法
                //重点2 directory.subscribe：consumer://100.119.69.10/com.aoe.demo.rpc.dubbo.DubboExampleInterf1?application=rpc-client&default.group=rpc-demo&default.version=1.0.1-aoe&dubbo=2.5.3&interface=com.aoe.demo.rpc.dubbo.DubboExampleInterf1&methods=serviceProvider&pid=5660&revision=0.0.1-SNAPSHOT&side=consumer&timestamp=1565169597733
                //重点3：添加至cluster
                invoker = refprotocol.refer(interfaceClass, urls.get(0));
            } else {
                //稍后分析，如“<dubbo:reference id="dubboExampleService1" interface="com.aoe.demo.rpc.dubbo.DubboExampleInterf1"  url="dubbo://127.0.0.1:20813;dubbo://127.0.0.1:20814"  />”url有多个时会运行至此
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
#### 2.4.1 refprotocol.refer分析
dubbo:reference的配置可采用注册中心获取服务接口地址（关联registry），也可通过url方式直连（url显示指定）；而在xml解析生成dubbo:reference生成ReferenceBean时会根据url或registry解析生成dReferenceBean的url的属性值：
```language
<dubbo:registry id="local_zk" address="zookeeper://127.0.0.1:2181"></dubbo:registry>

<dubbo:reference id="dubboExampleService1" interface="com.aoe.demo.rpc.dubbo.DubboExampleInterf1" check="false" registry="local_zk"/>
	
<dubbo:reference id="dubboExampleService1" interface="com.aoe.demo.rpc.dubbo.DubboExampleInterf1"  url="dubbo://127.0.0.1:20813;dubbo://127.0.0.1:20814"  /> 
```
url属性值结合map参数对应的url toString结果：
```language
registry://127.0.0.1:2181/com.alibaba.dubbo.registry.RegistryService?application=rpc-client&dubbo=2.5.3&pid=5660&refer=application%3Drpc-client%26default.group%3Drpc-demo%26default.version%3D1.0.1-aoe%26dubbo%3D2.5.3%26interface%3Dcom.aoe.demo.rpc.dubbo.DubboExampleInterf1%26methods%3DserviceProvider%26pid%3D5660%26revision%3D0.0.1-SNAPSHOT%26side%3Dconsumer%26timestamp%3D1565169597733&registry=zookeeper&timestamp=1565169709392
```
```language
dubbo://127.0.0.1:20813/com.aoe.demo.rpc.dubbo.DubboExampleInterf1?application=rpc-client&default.group=rpc-demo&default.version=1.0.1-aoe&dubbo=2.5.3&interface=com.aoe.demo.rpc.dubbo.DubboExampleInterf1&methods=serviceProvider&pid=9904&revision=0.0.1-SNAPSHOT&side=consumer&timestamp=1565319685624, dubbo://127.0.0.1:20814/com.aoe.demo.rpc.dubbo.DubboExampleInterf1?application=rpc-client&default.group=rpc-demo&default.version=1.0.1-aoe&dubbo=2.5.3&interface=com.aoe.demo.rpc.dubbo.DubboExampleInterf1&methods=serviceProvider&pid=9904&revision=0.0.1-SNAPSHOT&side=consumer&timestamp=1565319685624]
```
那么回到 invoker = refprotocol.refer(interfaceClass, urls.get(0))，基于dubbo、registry分别分析。
##### 2.4.1.1  基于registry分析 refprotocol.refer(interfaceClass, urls.get(0))
refprotocol根据分析，对应Protocol的SPI实现类实例，无缺省值则实例会Protocol&Adaptiver实例，而其方法会根据Url的protocol值获取Protocol实现类，那么此处即为RegistryProtocol. 
```language
    //RegistryProtocol
    @SuppressWarnings("unchecked")
	public <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException {
        //url实例toString后如上，此即即将Parameters中registry（REGISTRY_KEY即为registry)值zookeeper设置到url实例的protocol参数并同步从Parameters中将registr从Parameters中移除。
        url = url.setProtocol(url.getParameter(Constants.REGISTRY_KEY, Constants.DEFAULT_REGISTRY)).removeParameter(Constants.REGISTRY_KEY);
        //此处即根据url的protocol值zookeeper获取registry实例，即为ZookeeperRegistry类实例
        Registry registry = registryFactory.getRegistry(url);
        if (RegistryService.class.equals(type)) {
        	return proxyFactory.getInvoker((T) registry, type, url);
        }

        // group="a,b" or group="*"
        Map<String, String> qs = StringUtils.parseQueryString(url.getParameterAndDecoded(Constants.REFER_KEY));
       //获取group属性值，如若group不为空则先getMergeableCluster获取MergeableCluster实例再调用doRefer方法
        String group = qs.get(Constants.GROUP_KEY);
        if (group != null && group.length() > 0 ) {
            if ( ( Constants.COMMA_SPLIT_PATTERN.split( group ) ).length > 1
                    || "*".equals( group ) ) {
                return doRefer( getMergeableCluster(), registry, type, url );
            }
        }
        //此处cluster为Cluster$Adpative实例
        return doRefer(cluster, registry, type, url);
    }

   private <T> Invoker<T> doRefer(Cluster cluster, Registry registry, Class<T> type, URL url) {
        //根据type（即示例发布的接口class、url实例实例化RegistryDirectory对象directory 
        RegistryDirectory<T> directory = new RegistryDirectory<T>(type, url);
        directory.setRegistry(registry);
        directory.setProtocol(protocol);
        //根据参数实例化订阅服务url实例：consumer://100.119.69.10/com.aoe.demo.rpc.dubbo.DubboExampleInterf1?application=rpc-client&check=false&default.group=rpc-demo&default.version=1.0.1-aoe&dubbo=2.5.3&interface=com.aoe.demo.rpc.dubbo.DubboExampleInterf1&methods=serviceProvider&pid=12928&revision=0.0.1-SNAPSHOT&side=consumer&timestamp=1565338372226
        URL subscribeUrl = new URL(Constants.CONSUMER_PROTOCOL, NetUtils.getLocalHost(), 0, type.getName(), directory.getUrl().getParameters());
        if (! Constants.ANY_VALUE.equals(url.getServiceInterface())
                && url.getParameter(Constants.REGISTER_KEY, true)) {
            //当前示例运行到此（用于处理consumer），即进入ZookeeperRegistry(父类为FailbackRegistry)实例的registry方法
            registry.register(subscribeUrl.addParameters(Constants.CATEGORY_KEY, Constants.CONSUMERS_CATEGORY,
                    Constants.CHECK_KEY, String.valueOf(false)));
        }
        directory.subscribe(subscribeUrl.addParameter(Constants.CATEGORY_KEY, 
                Constants.PROVIDERS_CATEGORY 
                + "," + Constants.CONFIGURATORS_CATEGORY 
                + "," + Constants.ROUTERS_CATEGORY));
        return cluster.join(directory);
    }
```
##### 2.4.1.2 消费者注册到zookeeper分析
ZookeeperRegistry(父类为FailbackRegistry)的register
```language
    //FailbackRegistry类
    @Override
    public void register(URL url) {
        //即将url添加至registered集合
        super.register(url);
        //从failedRegistered、failedUnregistered中移除ur实例
        failedRegistered.remove(url);
        failedUnregistered.remove(url);
        try {
            // 向服务器端发送注册请求（当前示例为子类ZookeeperRegistry的doRegistry方法
            doRegister(url);
        } catch (Exception e) {
            Throwable t = e;

            // 如果开启了启动时检测，则直接抛出异常
            boolean check = getUrl().getParameter(Constants.CHECK_KEY, true)
                    && url.getParameter(Constants.CHECK_KEY, true)
                    && ! Constants.CONSUMER_PROTOCOL.equals(url.getProtocol());
            boolean skipFailback = t instanceof SkipFailbackWrapperException;
            if (check || skipFailback) {
                if(skipFailback) {
                    t = t.getCause();
                }
                throw new IllegalStateException("Failed to register " + url + " to registry " + getUrl().getAddress() + ", cause: " + t.getMessage(), t);
            } else {
                logger.error("Failed to register " + url + ", waiting for retry, cause: " + t.getMessage(), t);
            }

            // 将失败的注册请求记录到失败列表，定时重试
            failedRegistered.add(url);
        }
    }
```
```language
   //ZookeeperRegistry类
    protected void doRegister(URL url) {
        try {
               引得即根据
               zkClient.create(toUrlPath(url), url.getParameter(Constants.DYNAMIC_KEY, true));
        } catch (Throwable e) {
            throw new RpcException("Failed to register " + url + " to zookeeper " + getUrl() + ", cause: " + e.getMessage(), e);
        }
    }
```
zkClient.create(toUrlPath(url), url.getParameter(Constants.DYNAMIC_KEY, true)); 主要为：
1. toUrlPath(url)基于url实例生成完整的url路径，即对应zk的完整节点路径 ：
```
/dubbo/com.aoe.demo.rpc.dubbo.DubboExampleInterf1/consumers/consumer%3A%2F%2F100.119.69.10%2Fcom.aoe.demo.rpc.dubbo.DubboExampleInterf1%3Fapplication%3Drpc-client%26category%3Dconsumers%26check%3Dfalse%26default.group%3Drpc-demo%26default.version%3D1.0.1-aoe%26dubbo%3D2.5.3%26interface%3Dcom.aoe.demo.rpc.dubbo.DubboExampleInterf1%26methods%3DserviceProvider%26pid%3D12928%26revision%3D0.0.1-SNAPSHOT%26side%3Dconsumer%26timestamp%3D1565338372226
```
2. 基于zkClient实例（ZkclientZookeeperClient），使用步骤1完整节点路径远程写入zk数据（zkClient会嵌套判断/，最终多次调用zk从顶级节点逐个创建）。而之前一直发现有个问题，当前示例为最简化配置，若zookeeper未启动会导致阻塞，应用启动时卡住了。为啥呢？zkClient作为ZookeeperRegistry实例的成员变量，其在ZookeeperRegistry实例化时即会被实例化，那么我们回到ZookeeperRegistry实例化方法看看。
##### 2.4.1.3 ZookeeperRegistry实例化分析
ReferenceBean创建时，会基于其url或者registry属性将其作为消费者注册至注册中心并通过refprotocol.refer(interfaceClass, urls.get(0))（其中refprotocol对应Protocol@Adaptive实例-->RegistryProtocol）获取调用器invoker实例
>  refprotocol.refer引用远程服务
1. 当用户调用refer()所返回的Invoker对象的invoke()方法时，协议需相应执行同URL远端export()传入的Invoker对象的invoke()方法。
2. refer()返回的Invoker由协议实现，协议通常需要在此Invoker中发送远程请求。
3. 当url中有设置check=false时，连接失败不能抛出异常，并内部自动恢复。
- 而注册至注册中心，即需要实例化ZookeeperRegistry：
```language
    //ZookeeperRegistry类
    public ZookeeperRegistry(URL url, ZookeeperTransporter zookeeperTransporter) {
        super(url);
        if (url.isAnyHost()) {
    		throw new IllegalStateException("registry address == null");
    	}
        String group = url.getParameter(Constants.GROUP_KEY, DEFAULT_ROOT);
        if (! group.startsWith(Constants.PATH_SEPARATOR)) {
            group = Constants.PATH_SEPARATOR + group;
        }
        this.root = group;
        //zookeeperTransporter对应ZookeeperTransporter$Adpative，运行时对应ZkclientZookeeperTransporter
        //url值：zookeeper://127.0.0.1:2181/com.alibaba.dubbo.registry.RegistryService?application=rpc-client&dubbo=2.5.3&interface=com.alibaba.dubbo.registry.RegistryService&pid=11672&timestamp=1565576932790
        zkClient = zookeeperTransporter.connect(url);
        zkClient.addStateListener(new StateListener() {
            public void stateChanged(int state) {
            	if (state == RECONNECTED) {
	            	try {
						recover();
					} catch (Exception e) {
						logger.error(e.getMessage(), e);
					}
            	}
            }
        });
    }
```
运行时对应ZkclientZookeeperTransporter,其底层仍是对应ZkClient的封装:
```language
        //ZookeeperRegistry类
	public ZkclientZookeeperClient(URL url) {
		super(url);
		client = new ZkClient(url.getBackupAddress());
		client.subscribeStateChanges(new IZkStateListener() {
			public void handleStateChanged(KeeperState state) throws Exception {
				ZkclientZookeeperClient.this.state = state;
				if (state == KeeperState.Disconnected) {
					stateChanged(StateListener.DISCONNECTED);
				} else if (state == KeeperState.SyncConnected) {
					stateChanged(StateListener.CONNECTED);
				}
			}
			public void handleNewSession() throws Exception {
				stateChanged(StateListener.RECONNECTED);
			}
		});
	}

```
ZkClient构造方法（可看出，默认连接超时时间为Integer.MAX_VALUE，所以才会导致类似于假死）
```language
   //serverstring值为127.0.0.1:2181
    public ZkClient(String serverstring) {
        this(serverstring, Integer.MAX_VALUE);
    }

    public ZkClient(String zkServers, int connectionTimeout) {
        this(new ZkConnection(zkServers), connectionTimeout);
    }
```
通过验证，当前dubbo源码不未提供zookeeper连接超时参数设置，即使如
> <dubbo:registry id="local_zk" address="zookeeper://127.0.0.1:2181" timeout="10"></dubbo:registry>
##### 2.4.1.4 消费者注册
回到ReferenceBean实例化，因其实现FactoryBean接口，故其实例化时会调用对应getObject方法，而会调用其父类RefrenceConfig类的init()方法，根据上在分析有重要几步：1.RegistryProtocol类refer根据zk的url从registryFactory.getRegistry(url)获取Registry实例（首次需创建ZookeeperRegistry对应bean；其中会合建zkClient检查其联通性）；2.RegistryProtocol类doRefer方法调用registry.register完成注册（实际为ZookeeperRegistry父类FailbackRegistry的register方法），其会调用zkClient写入数据；
##### 2.4.1.4 消费者服务订阅
RegistryProtocol类doRefer方法调用registry.register完成注册之后，directory.subscribe()方法
```language
    //RegistryDirectory类
    public void subscribe(URL url) {
        setConsumerUrl(url);
        registry.subscribe(url, this);  //最终调用ZookeeperRegistry的doSubscribe方法
    }
```
而RegistryDirectory构造方法中则根据router配置设计其对应的Routers值：1.根据Url实例的router参数routerkey基于ExtensionLoader获取RouterFactory对应的实现类；2.添加默认的MockInvokersSelector（即如若未设置routerkey则Routers仅包含默认的MockInvokersSelector）

- ZkClient实例化时（ZookeeperRegistry实例化会同步实例化ZkClient实例），ZkClient构造方法会同步实例化ZkConnection并同步调用ZkClient的connection方法，在ZkClient的connection方法内主要做这两点处理：1.根据connection对应的zk地址实例化守护线程ZkEventThread并同步启动；2.以Watchers（zookeeper)调用ZkConnection实例的connect方法，其会实例化ZooKeeper；ZooKeeper构造方法内实例化ClientCnxn (非Thread子类）并调用start()方法（start方法内就两行： this.sendThread.start();this.eventThread.start();）；即启动sendThread守护线程、eventThread守护线程。
- 该部分使用较多的zookeeper-3.3.3.jar包里的类，基于事件监听，zookeeper状态、数据变化会触发将事件并写入waitingEvents（LinkedBlockingQueue：链表实现的有界阻塞队列）。其根据事件类型：stateChanged状态变化（如连接、断开连接）、dataChanged（数据变化）调用同的逻辑处理。
```language
    //ZkClient类(处理数据变化事件：WatchedEvent state:SyncConnected type:NodeChildrenChanged path:/dubbo/com.aoe.demo.rpc.dubbo.DubboExampleInterf1/providers）
    private void processDataOrChildChange(WatchedEvent event) {
        ///path:dubbo/com.aoe.demo.rpc.dubbo.DubboExampleInterf1/providers
        final String path = event.getPath();

        if (event.getType() == EventType.NodeChildrenChanged || event.getType() == EventType.NodeCreated || event.getType() == EventType.NodeDeleted) {
          //_childListener:{/dubbo/com.aoe.demo.rpc.dubbo.DubboExampleInterf1/providers=[com.alibaba.dubbo.remoting.zookeeper.zkclient.ZkclientZookeeperClient$2@10c6c6b], /dubbo/com.aoe.demo.rpc.dubbo.DubboExampleInterf1/routers=[com.alibaba.dubbo.remoting.zookeeper.zkclient.ZkclientZookeeperClient$2@8f6d64], /dubbo/com.aoe.demo.rpc.dubbo.DubboExampleInterf1/configurators=[com.alibaba.dubbo.remoting.zookeeper.zkclient.ZkclientZookeeperClient$2@56ff18]}
            Set<IZkChildListener> childListeners = _childListener.get(path);
            if (childListeners != null && !childListeners.isEmpty()) {
                fireChildChangedEvents(path, childListeners);
            }
        }

        if (event.getType() == EventType.NodeDataChanged || event.getType() == EventType.NodeDeleted || event.getType() == EventType.NodeCreated) {
            Set<IZkDataListener> listeners = _dataListener.get(path);
            if (listeners != null && !listeners.isEmpty()) {
                fireDataChangedEvents(event.getPath(), listeners);
            }
        }
    }
```
断开连接事件:ZkEvent[State changed to Disconnected sent to com.alibaba.dubbo.remoting.zookeeper.zkclient.ZkclientZookeeperClient$1@1feb2ea]
连接事件：WatchedEvent state:SyncConnected type:None path:null
数据变化事件：WatchedEvent state:SyncConnected type:NodeChildrenChanged path:/dubbo/com.aoe.demo.rpc.dubbo.DubboExampleInterf1/providers
- consumer://100.119.69.44/com.aoe.demo.rpc.dubbo.DubboExampleInterf1?application=rpc-client&category=providers,configurators,routers&default.group=rpc-demo&default.version=1.0.1-aoe&dubbo=2.5.3&interface=com.aoe.demo.rpc.dubbo.DubboExampleInterf1&methods=serviceProvider&pid=13140&revision=0.0.1-SNAPSHOT&side=consumer&timestamp=1565750957716
根据如上消费者url基于规则装成生成订阅服务依赖的zk节点的节点url：
```language
/dubbo/com.aoe.demo.rpc.dubbo.DubboExampleInterf1/providers, /dubbo/com.aoe.demo.rpc.dubbo.DubboExampleInterf1/configurators, /dubbo/com.aoe.demo.rpc.dubbo.DubboExampleInterf1/routers
```
遍历节点url从zk获取服务发布信息，如DubboExampleInterf1接口由100.119.69.44 发布，其服务接口完整信息为：
> [dubbo%3A%2F%2F100.119.69.44%3A20890%2Fcom.aoe.demo.rpc.dubbo.DubboExampleInterf1%3Fanyhost%3Dtrue%26application%3Drpc-server%26default.timeout%3D1000%26dubbo%3D2.5.3%26interface%3Dcom.aoe.demo.rpc.dubbo.DubboExampleInterf1%26methods%3DserviceProvider%26pid%3D11880%26revision%3D0.0.1-SNAPSHOT%26side%3Dprovider%26timestamp%3D1565750298397]

- 在完成如上完成消费者订阅时(ZookeeperRegistry类的doSubscribe方法)，会同上根据消费者url生成服务提供的zk节点路径 （对对应上面的：/dubbo/com.aoe.demo.rpc.dubbo.DubboExampleInterf1/providers及configurators及routers），通过zkClient.addChildListener为其添加子节点Linstener（用于接受zk的dataChange事件)的同时会在其内部调用ZkClient的getChildren获取当前节点的Children列表。
- 如若Children列表为空则会根据消费者接口Url生成服务提供者Url实例（其protocol为empty，catagory为provider)，之后基于刚封装的empty Url实例列表notify对应的Linster；如若provider、configurators、routers子节点数据此处也会封装（封装时会同步比较消费者Url与从zk获取子节点数据。比如之前消费端rpc-client通过启动系统参数指定了Consumer的gro为rpc-demo；而prc-server发布服务未指定group属性，在获取children节点信息后会与消费端订阅服务的group比对，不一至也会实例化新的empty Url供后面使用）。封装完成则执行到RegistryDirectory的notify方法(其实现了NotifyListener）。
- RegistryDirectory的notify方法主要完成：1.遍历Urls列表封装empty Url实例至当前RegistryDirectory实例的invokerUrls，封装configurators Url至configuratorUrls，封装routers Url至routerUrls；2.解析configuratorUrls数据至RegistryDirectory实例；3.解析routerUrls数据至RegistryDirectory实例routers; 4.执行内部refreshInvokery方法（即rpc-server未启动之前未找到对应的Provider信息，Url类型为empty，那么此则会将当前RegistryDirectory实例的
forbidden置为true，置空methodInvokerMap，清空urlInvokerMap（当前实际本来就为null）。而notify还可用于在zk中数据变化如provider下线后消费端面监听后对应处理。
3. 基于cluster.join(directory)获取调用器（实际调用MockClusterWrapper的join方法）返回MockClusterInvoker实例
##### 2.4.1.5 服务发布者检测
完成invoker实例化之后，基于consumer消费者check设置（如若示显示指定则默认为true，即需验证provider状态），实际主要就是检查上面刚讲的urlInvokerMap是否有可数据。
##### 2.4.1.6 服务代理创建(T) proxyFactory.getProxy(invoker)
完成上述调用器invoker实例化及invoker可用性检查之后，会基于(T) proxyFactory.getProxy(invoker)生成服务代理对象。
- proxyFactory对应当前文章尾部的ProxyFactory$Adpative 源码，通过其可发现调用的是JavassistProxyFactory实例，通过修改源码将对应生成的Class文件写入磁盘后反编译如下（由于window文件名不区分大小写，Proxy0与proxy0两个class在写入磁盘后出现覆盖，通过debug方才获取到）
```language
public class JavassistProxyFactory extends AbstractProxyFactory {

    @SuppressWarnings("unchecked")
    public <T> T getProxy(Invoker<T> invoker, Class<?>[] interfaces) {
        return (T) Proxy.getProxy(interfaces).newInstance(new InvokerInvocationHandler(invoker));
    }
    .......
}
```
```language
import java.lang.reflect.InvocationHandler;

public class Proxy0
  extends Proxy
  implements ClassGenerator.DC
{
  public Object newInstance(InvocationHandler paramInvocationHandler)
  {
    return new proxy0(paramInvocationHandler);
  }
}

```
```language
public class proxy0
  implements ClassGenerator.DC, EchoService, DubboExampleInterf1
{
  public static Method[] methods;
  private InvocationHandler handler;
  
  public List serviceProvider(List paramList)
  {
    Object[] arrayOfObject = new Object[1];
    arrayOfObject[0] = paramList;
    Object localObject = this.handler.invoke(this, methods[0], arrayOfObject);
    return (List)localObject;
  }
  
  public Object $echo(Object paramObject)
  {
    Object[] arrayOfObject = new Object[1];
    arrayOfObject[0] = paramObject;
    Object localObject = this.handler.invoke(this, methods[1], arrayOfObject);
    return (Object)localObject;
  }
  
  public proxy0() {}
  
  public proxy0(InvocationHandler paramInvocationHandler)
  {
    this.handler = paramInvocationHandler;
  }
}

```
- 即通过如上源码可发现，ReferenceConfig的getObject返回的实例为：实现了ClassGenerator.DC, EchoService, DubboExampleInterf1 3个接口的实现类，其对应$echo方法（EchoService接口）、serviceProvider方法则是实际调用InvokerInvocationHandler(invoker)的invoke方法（即动态代理方式），其内部由是调用invoke对象（即MockClusterInvoker）的invoke方法
```language
     //MockClusterInvoker类
     public class MockClusterInvoker<T> implements Invoker<T>{
	
	private static final Logger logger = LoggerFactory.getLogger(MockClusterInvoker.class);

	private final Directory<T> directory ;
	
	private final Invoker<T> invoker;
   
    public MockClusterInvoker(Directory<T> directory, Invoker<T> invoker) {
       	this.directory = directory;
       	this.invoker = invoker;
    }
     //invoke方法：1.若是非Mock场景则是调用FailoverClusterInvoker（实际为FailoverClusterInvoker父类 ）的invoker方法（因为外层的invoker实际是MockClusterInvoker对FailoverClusterInvoker的包装；2.若是Mock场景则直接调用doMockInvoke方法
    public Result invoke(Invocation invocation) throws RpcException {
		Result result = null;
        
        String value = directory.getUrl().getMethodParameter(invocation.getMethodName(), Constants.MOCK_KEY, Boolean.FALSE.toString()).trim(); 
        if (value.length() == 0 || value.equalsIgnoreCase("false")){
        	//no mock
        	result = this.invoker.invoke(invocation);
        } else if (value.startsWith("force")) {
        	if (logger.isWarnEnabled()) {
        		logger.info("force-mock: " + invocation.getMethodName() + " force-mock enabled , url : " +  directory.getUrl());
        	}
        	//force:direct mock
        	result = doMockInvoke(invocation, null);
        } else {
        	//fail-mock
        	try {
        		result = this.invoker.invoke(invocation);
        	}catch (RpcException e) {
				if (e.isBiz()) {
					throw e;
				} else {
					if (logger.isWarnEnabled()) {
		        		logger.info("fail-mock: " + invocation.getMethodName() + " fail-mock enabled , url : " +  directory.getUrl(), e);
		        	}
					result = doMockInvoke(invocation, e);
				}
			}
        }
        return result;
	}
```


###### 服务注册说明：
ZookeeperRegistry的doSubscribe中如若Url值为：
> provider://100.119.69.44:20890/com.aoe.demo.rpc.dubbo.DubboExampleInterf1?anyhost=true&application=rpc-server&category=configurators&check=false&default.timeout=1000&dubbo=2.5.3&interface=com.aoe.demo.rpc.dubbo.DubboExampleInterf1&methods=serviceProvider&pid=14116&revision=0.0.1-SNAPSHOT&side=provider&timestamp=1565748909199：
则会根据如上Url生成如下zk节点并调用zkClient完成远程写入
```
[/dubbo/com.aoe.demo.rpc.dubbo.DubboExampleInterf1/providers, /dubbo/com.aoe.demo.rpc.dubbo.DubboExampleInterf1/configurators, /dubbo/com.aoe.demo.rpc.dubbo.DubboExampleInterf1/routers]
```

### 自定义变量示例
- 在分析dubbo:reference标签解析对ReferenceConfig实例化时，若未显示在为其配置dubbo:consumer标签属性，则会默认为ReferenceConfig实例的属性consumer属性实例化默认的ConsumerConfig实例（包含如lazy、timeout、reconnect、version、group等），并尝试从系统参数获取对应配置的值通过method.invoke方式反射为ConsumerConfig实例的属性完成赋值，下面即简单对此种方式做下示例（参考：https://www.cnblogs.com/yangmingke/p/6058898.html）
- 基于Eclipse+Tomcat方式设置属性以便可通过System.getProperty（“XXX”）获取自定义变量
1. 打开tomcat server（双击对应server），会显示常用的tomcat 端口号、启动或停止超时时间设置。
2. General Information中点击Open launch configuration（就面窗口而左上方），可打开"Edit Configuration"窗口。
3. 选择Arguments，在VM arguments输入框显示：
```language
-Dcatalina.base="D:\work\webcontainer\tomcat7" -Dcatalina.home="D:\work\webcontainer\tomcat7" -Dwtp.deploy="D:\work\webcontainer\tomcat7\wtpwebapps" -Djava.endorsed.dirs="D:\work\webcontainer\tomcat7\endorsed" -Ddubbo.consumer.version="1.0.1-aoe" -Ddubbo.consumer.group="rpc-demo"
```
如上在VM arguments后追加-DXXX=****(-D不能省略)，这样就可以通过 System.getProperty（“XXX”）获取****了；同样也可在catalina.bat（catalina.sh）通过JAVA_OPTS来添加自定义变量1. 

### 基于Javassist动态生成各class源码
```language
--------------------------------------------------------
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
		String extName = (url.getProtocol() == null ? "dubbo" : url.getProtocol());
		if (extName == null)
			throw new IllegalStateException("Fail to get extension(com.alibaba.dubbo.rpc.Protocol) name from url("
					+ url.toString() + ") use keys([protocol])");
		com.alibaba.dubbo.rpc.Protocol extension = (com.alibaba.dubbo.rpc.Protocol) ExtensionLoader
				.getExtensionLoader(com.alibaba.dubbo.rpc.Protocol.class).getExtension(extName);
		return extension.refer(arg0, arg1);
	}
}--------------------------------------------------------package com.alibaba.dubbo.rpc.cluster;import com.alibaba.dubbo.common.extension.ExtensionLoader;

public class Cluster$Adpative implements com.alibaba.dubbo.rpc.cluster.Cluster {
	public com.alibaba.dubbo.rpc.Invoker join(com.alibaba.dubbo.rpc.cluster.Directory arg0)
			throws com.alibaba.dubbo.rpc.cluster.Directory {
		if (arg0 == null)
			throw new IllegalArgumentException("com.alibaba.dubbo.rpc.cluster.Directory argument == null");
		if (arg0.getUrl() == null)
			throw new IllegalArgumentException("com.alibaba.dubbo.rpc.cluster.Directory argument getUrl() == null");
		com.alibaba.dubbo.common.URL url = arg0.getUrl();
		String extName = url.getParameter("cluster", "failover");
		if (extName == null)
			throw new IllegalStateException(
					"Fail to get extension(com.alibaba.dubbo.rpc.cluster.Cluster) name from url(" + url.toString()
							+ ") use keys([cluster])");
		com.alibaba.dubbo.rpc.cluster.Cluster extension = (com.alibaba.dubbo.rpc.cluster.Cluster) ExtensionLoader
				.getExtensionLoader(com.alibaba.dubbo.rpc.cluster.Cluster.class).getExtension(extName);
		return extension.join(arg0);
	}
}--------------------------------------------------------package com.alibaba.dubbo.rpc;import com.alibaba.dubbo.common.extension.ExtensionLoader;

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
		com.alibaba.dubbo.rpc.ProxyFactory extension = (com.alibaba.dubbo.rpc.ProxyFactory) ExtensionLoader
				.getExtensionLoader(com.alibaba.dubbo.rpc.ProxyFactory.class).getExtension(extName);
		return extension.getInvoker(arg0, arg1, arg2);
	}
}--------------------------------------------------------package com.alibaba.dubbo.registry;import com.alibaba.dubbo.common.extension.ExtensionLoader;

public class RegistryFactory$Adpative implements com.alibaba.dubbo.registry.RegistryFactory {
	public com.alibaba.dubbo.registry.Registry getRegistry(com.alibaba.dubbo.common.URL arg0) {
		if (arg0 == null)
			throw new IllegalArgumentException("url == null");
		com.alibaba.dubbo.common.URL url = arg0;
		String extName = (url.getProtocol() == null ? "dubbo" : url.getProtocol());
		if (extName == null)
			throw new IllegalStateException(
					"Fail to get extension(com.alibaba.dubbo.registry.RegistryFactory) name from url(" + url.toString()
							+ ") use keys([protocol])");
		com.alibaba.dubbo.registry.RegistryFactory extension = (com.alibaba.dubbo.registry.RegistryFactory) ExtensionLoader
				.getExtensionLoader(com.alibaba.dubbo.registry.RegistryFactory.class).getExtension(extName);
		return extension.getRegistry(arg0);
	}
}--------------------------------------------------------package com.alibaba.dubbo.remoting.zookeeper;import com.alibaba.dubbo.common.extension.ExtensionLoader;

public class ZookeeperTransporter$Adpative implements com.alibaba.dubbo.remoting.zookeeper.ZookeeperTransporter {
	public com.alibaba.dubbo.remoting.zookeeper.ZookeeperClient connect(com.alibaba.dubbo.common.URL arg0) {
		if (arg0 == null)
			throw new IllegalArgumentException("url == null");
		com.alibaba.dubbo.common.URL url = arg0;
		String extName = url.getParameter("client", url.getParameter("transporter", "zkclient"));
		if (extName == null)
			throw new IllegalStateException(
					"Fail to get extension(com.alibaba.dubbo.remoting.zookeeper.ZookeeperTransporter) name from url("
							+ url.toString() + ") use keys([client, transporter])");
		com.alibaba.dubbo.remoting.zookeeper.ZookeeperTransporter extension = (com.alibaba.dubbo.remoting.zookeeper.ZookeeperTransporter) ExtensionLoader
				.getExtensionLoader(com.alibaba.dubbo.remoting.zookeeper.ZookeeperTransporter.class)
				.getExtension(extName);
		return extension.connect(arg0);
	}
}--------------------------------------------------------package com.alibaba.dubbo.rpc.cluster;import com.alibaba.dubbo.common.extension.ExtensionLoader;

public class RouterFactory$Adpative implements com.alibaba.dubbo.rpc.cluster.RouterFactory {
	public com.alibaba.dubbo.rpc.cluster.Router getRouter(com.alibaba.dubbo.common.URL arg0) {
		if (arg0 == null)
			throw new IllegalArgumentException("url == null");
		com.alibaba.dubbo.common.URL url = arg0;
		String extName = url.getProtocol();
		if (extName == null)
			throw new IllegalStateException(
					"Fail to get extension(com.alibaba.dubbo.rpc.cluster.RouterFactory) name from url(" + url.toString()
							+ ") use keys([protocol])");
		com.alibaba.dubbo.rpc.cluster.RouterFactory extension = (com.alibaba.dubbo.rpc.cluster.RouterFactory) ExtensionLoader
				.getExtensionLoader(com.alibaba.dubbo.rpc.cluster.RouterFactory.class).getExtension(extName);
		return extension.getRouter(arg0);
	}
}--------------------------------------------------------package com.alibaba.dubbo.rpc.cluster;import com.alibaba.dubbo.common.extension.ExtensionLoader;

public class ConfiguratorFactory$Adpative implements com.alibaba.dubbo.rpc.cluster.ConfiguratorFactory {
	public com.alibaba.dubbo.rpc.cluster.Configurator getConfigurator(com.alibaba.dubbo.common.URL arg0) {
		if (arg0 == null)
			throw new IllegalArgumentException("url == null");
		com.alibaba.dubbo.common.URL url = arg0;
		String extName = url.getProtocol();
		if (extName == null)
			throw new IllegalStateException(
					"Fail to get extension(com.alibaba.dubbo.rpc.cluster.ConfiguratorFactory) name from url("
							+ url.toString() + ") use keys([protocol])");
		com.alibaba.dubbo.rpc.cluster.ConfiguratorFactory extension = (com.alibaba.dubbo.rpc.cluster.ConfiguratorFactory) ExtensionLoader
				.getExtensionLoader(com.alibaba.dubbo.rpc.cluster.ConfiguratorFactory.class).getExtension(extName);
		return extension.getConfigurator(arg0);
	}
}
```
