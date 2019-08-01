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
        Set<String> props = new HashSet<String>();
        ManagedMap parameters = null;
        //遍历
        for (Method setter : beanClass.getMethods()) {
            String name = setter.getName();
            if (name.length() > 3 && name.startsWith("set")
                    && Modifier.isPublic(setter.getModifiers())
                    && setter.getParameterTypes().length == 1) {
                Class<?> type = setter.getParameterTypes()[0];
                String property = StringUtils.camelToSplitName(name.substring(3, 4).toLowerCase() + name.substring(4), "-");
                props.add(property);
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
                    continue;
                }
                if ("parameters".equals(property)) {
                    parameters = parseParameters(element.getChildNodes(), beanDefinition);
                } else if ("methods".equals(property)) {
                    parseMethods(id, element.getChildNodes(), beanDefinition, parserContext);
                } else if ("arguments".equals(property)) {
                    parseArguments(id, element.getChildNodes(), beanDefinition, parserContext);
                } else {
                    String value = element.getAttribute(property);
                    if (value != null) {
                    	value = value.trim();
                    	if (value.length() > 0) {
                    		if ("registry".equals(property) && RegistryConfig.NO_AVAILABLE.equalsIgnoreCase(value)) {
                            	RegistryConfig registryConfig = new RegistryConfig();
                            	registryConfig.setAddress(RegistryConfig.NO_AVAILABLE);
                            	beanDefinition.getPropertyValues().addPropertyValue(property, registryConfig);
                            } else if ("registry".equals(property) && value.indexOf(',') != -1) {
                    			parseMultiRef("registries", value, beanDefinition, parserContext);
                            } else if ("provider".equals(property) && value.indexOf(',') != -1) {
                            	parseMultiRef("providers", value, beanDefinition, parserContext);
                            } else if ("protocol".equals(property) && value.indexOf(',') != -1) {
                                parseMultiRef("protocols", value, beanDefinition, parserContext);
                            } else {
                                Object reference;
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
		                        beanDefinition.getPropertyValues().addPropertyValue(property, reference);
                            }
                    	}
                    }
                }
            }
        }
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
