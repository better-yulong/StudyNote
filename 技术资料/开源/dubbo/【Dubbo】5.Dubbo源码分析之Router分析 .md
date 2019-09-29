为分析dubbo路由源码，于是新建两个tomcat  server，计划同时运行两个rpc-server以便于分析rpc-client请求rpc-server时的路由原理；而在启动时却发现基于原dubbo默认配置，server1启动成功可将服务注册至zookeeper，但server2却提示 Address already in use: bind; 原来因rpc-server服务发布基于dubbo协议，即默认使用netty模式发布服务，同时端口也为默认端口20890。那么在同一PC上基于当前配置肯定会出现端口暂用，那么有解决办法吗？根据之前的分析可知道如果切换至hessian方式发布服务则可以，那么仍使用dubbo方式发布呢？是否可以使用随机端口，还得研究看看。

### 一.DubboProtocol分析dubbo端口
根据之前的源码分析，可知道Dubbo协议与Hessian协议会以不同的方式发布服务；故推断相关源码不会在ServiceBean类，而应该在Dubbo相关类中，于是优先查看DubboProtocol源码（别说，还真找对了）：
```language
    //DubboProtocol类
    public static final int DEFAULT_PORT = 20880;
    
            final int defaultPort = ExtensionLoader.getExtensionLoader(Protocol.class).getExtension(name).getDefaultPort();
        if (port == null || port == 0) {
            port = defaultPort;
        }
        if (port == null || port <= 0) {
            port = getRandomPort(name);
            if (port == null || port < 0) {
                port = NetUtils.getAvailablePort(defaultPort);
                putRandomPort(name, port);
            }
            logger.warn("Use random available port(" + port + ") for protocol " + name);
        }
```
看到上面源码想必不需要作太多说明：	<dubbo:protocol name="dubbo" port="-1"></dubbo:protocol>，修改后再次验证本地双节点启动，查看zk：[zk: localhost:2181(CONNECTED) 8] ls /dubbo/com.aoe.demo.rpc.dubbo.DubboExampleI
nterf1/providers，确实两个服务提供者均注册成功。

### 二.Dubbo默认Router
未指定router配置时，默认zk中接口的providers对应服务提供者信息，consumer对应服务消费者信息，router对应路收信息（默认为空）。那么如上示例，rpc-client究竟会基于怎样的策略访问rpc-server呢？
AbstractClusterInvoker
```language
   //AbstractClusterInvoker类（AvailableClusterInvoker的父类）
    public Result invoke(final Invocation invocation) throws RpcException {
        //检查当前invoker状态是否为invoker
        checkWheatherDestoried();

        LoadBalance loadbalance;
        //基于invocation值及StaticDirectory（可理解为zk对应节点数据的抽象）获取List<Invoker<T>>，因上面有两个rpc-server节点分别发布了一个服务，故此处会有两个invoker实例。
        List<Invoker<T>> invokers = list(invocation);
        if (invokers != null && invokers.size() > 0) {
            //获取第一个invoker对应method是否有负载配置（loadbalance ),若没有则使用DEFAULT_LOADBALANCE，即random获取LoadBalance的实现类RandomLoadBalance的实例
            loadbalance = ExtensionLoader.getExtensionLoader(LoadBalance.class).getExtension(invokers.get(0).getUrl()
                    .getMethodParameter(invocation.getMethodName(),Constants.LOADBALANCE_KEY, Constants.DEFAULT_LOADBALANCE));
        } else {
            loadbalance = ExtensionLoader.getExtensionLoader(LoadBalance.class).getExtension(Constants.DEFAULT_LOADBALANCE);
        }
       //同步请求是空处理；如若为异步请求则会添加invocation id至 Attachment
        RpcUtils.attachInvocationIdIfAsync(getUrl(), invocation);
        //对应FailoverClusterInvoker的doInvoke方法
        return doInvoke(invocation, invokers, loadbalance);
    }
```

#### 2.1 验证负载策略
之前为验证mock机制，DubboExampleInterf1 的实现类DubboExampleService1方法serviceProvider会sleep 20秒，以便触发服务端超时的mock机制；同时为便于验证负载分发策略，基于dubbo已提供的工具类在serviceProvider方法中打印出对应接收请求JVM的PID：String.valueOf(ConfigUtils.getPid())。请求时却发现，rpc-client第请求1次，rpc-server两个server均会接受到请求，初步推断是dubbo的失败重试策略；那么取消取serviceProvider方法的sleep逻辑，多次验证后发现基于random时当前这种验证方式也可做到两个server 均衡收到请求？额，这个就和之前代码分析时理解的有差异了。
##### 2.1.1 Random负载
- 1 个rpc-client server，2 个rpc-server实例（通过<dubbo:protocol name="dubbo" port="-1"></dubbo:protocol>可使用随机端口发布dubbo服务，可用于同一台服务器同时运行多个dubbo server而避免出现端口占用）
- 服务端接口实现类方法部分代码：
```language
	List serviceNames = new ArrayList<String>();
	serviceNames.add("DubboExampleService1:" + String.valueOf(ConfigUtils.getPid()));
	return serviceNames ;
```
- 响应结果：
```language
DubboExampleService1 result:DubboExampleService1:10820
DubboExampleService1 result:DubboExampleService1:8220
DubboExampleService1 result:DubboExampleService1:8220
DubboExampleService1 result:DubboExampleService1:8220
DubboExampleService1 result:DubboExampleService1:10820
DubboExampleService1 result:DubboExampleService1:10820
DubboExampleService1 result:DubboExampleService1:10820
DubboExampleService1 result:DubboExampleService1:10820
DubboExampleService1 result:DubboExampleService1:8220
DubboExampleService1 result:DubboExampleService1:8220
DubboExampleService1 result:DubboExampleService1:8220
DubboExampleService1 result:DubboExampleService1:10820
DubboExampleService1 result:DubboExampleService1:10820
DubboExampleService1 result:DubboExampleService1:8220
```
整体来看，2个rpc-server几乎上各占50%，那具体请求是如何分发的呢？
FailoverClusterInvoker(AbstractClusterInvoker).invoke(Invocation)，该方法中会确认loadbalance方式，若为指定则默认使用Random方式，之后则调用 FailoverClusterInvoker.doInvoke(Invocation, List<Invoker<T>>, LoadBalance)：
```language
       @SuppressWarnings({ "unchecked", "rawtypes" })
    public Result doInvoke(Invocation invocation, final List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
    	List<Invoker<T>> copyinvokers = invokers;
    	checkInvokers(copyinvokers, invocation);
        //获取retry重试配置，如若示配置则默认为3；如若为负，则设置为1即不重试。
        int len = getUrl().getMethodParameter(invocation.getMethodName(), Constants.RETRIES_KEY, Constants.DEFAULT_RETRIES) + 1;
        if (len <= 0) {
            len = 1;
        }
        // retry loop.
        RpcException le = null; // last exception.
        List<Invoker<T>> invoked = new ArrayList<Invoker<T>>(copyinvokers.size()); // invoked invokers.
        Set<String> providers = new HashSet<String>(len);
        for (int i = 0; i < len; i++) {
        	//重试时，进行重新选择，避免重试时invoker列表已发生变化.
        	//注意：如果列表发生了变化，那么invoked判断会失效，因为invoker示例已经改变
        	if (i > 0) {
        		checkWheatherDestoried();
        		copyinvokers = list(invocation);
        		//重新检查一下
        		checkInvokers(copyinvokers, invocation);
        	}
            /Random时对应RandomLoadBalance的doSelect方法
            Invoker<T> invoker = select(loadbalance, invocation, copyinvokers, invoked);
            invoked.add(invoker);
            RpcContext.getContext().setInvokers((List)invoked);
            try {
                Result result = invoker.invoke(invocation);
                if (le != null && logger.isWarnEnabled()) {
                    logger.warn("Although retry the method " + invocation.getMethodName()
                            + " in the service " + getInterface().getName()
                            + " was successful by the provider " + invoker.getUrl().getAddress()
                            + ", but there have been failed providers " + providers 
                            + " (" + providers.size() + "/" + copyinvokers.size()
                            + ") from the registry " + directory.getUrl().getAddress()
                            + " on the consumer " + NetUtils.getLocalHost()
                            + " using the dubbo version " + Version.getVersion() + ". Last error is: "
                            + le.getMessage(), le);
                }
                return result;
            } catch (RpcException e) {
                if (e.isBiz()) { // biz exception.
                    throw e;
                }
                le = e;
            } catch (Throwable e) {
                le = new RpcException(e.getMessage(), e);
            } finally {
                providers.add(invoker.getUrl().getAddress());
            }
        }
        throw new RpcException(le != null ? le.getCode() : 0, "Failed to invoke the method "
                + invocation.getMethodName() + " in the service " + getInterface().getName() 
                + ". Tried " + len + " times of the providers " + providers 
                + " (" + providers.size() + "/" + copyinvokers.size() 
                + ") from the registry " + directory.getUrl().getAddress()
                + " on the consumer " + NetUtils.getLocalHost() + " using the dubbo version "
                + Version.getVersion() + ". Last error is: "
                + (le != null ? le.getMessage() : ""), le != null && le.getCause() != null ? le.getCause() : le);
    }
```

```language
    //RandomLoadBalance类
    protected <T> Invoker<T> doSelect(List<Invoker<T>> invokers, URL url, Invocation invocation) {
        int length = invokers.size(); // 总个数
        int totalWeight = 0; // 总权重
        boolean sameWeight = true; // 权重是否都一样
        for (int i = 0; i < length; i++) {
            int weight = getWeight(invokers.get(i), invocation);
            totalWeight += weight; // 累计总权重
            if (sameWeight && i > 0
                    && weight != getWeight(invokers.get(i - 1), invocation)) {
                sameWeight = false; // 计算所有权重是否一样
            }
        }
        if (totalWeight > 0 && ! sameWeight) {
            // 如果权重不相同且权重大于0则按总权重数随机
            int offset = random.nextInt(totalWeight);
            // 并确定随机值落在哪个片断上
            for (int i = 0; i < length; i++) {
                offset -= getWeight(invokers.get(i), invocation);
                if (offset < 0) {
                    return invokers.get(i);
                }
            }
        }
        // 如果权重相同或权重为0则均等随机
        return invokers.get(random.nextInt(length));
    }
```
- 因为上面示例是同时运行两个rpc-server，且并款配置权重，会执行最后一行invokers.get(random.nextInt(length))来随机返回invokers中的其中一个（length为2），而由于此处的random为java.util.Random，实际可认为是为随机算法，即在0、1两个数字之间生成的随机实际也是均匀分布的；通过这种方式来实现整体均衡。
- 如若配置权重，则通过如上方式在总权重生成随机数之后确认获取哪个invoker对象用于系统调用；但个人感觉如若invoker节点非常多，此种方式会相对的消耗过多的CPU资源（节点越多越明显）；但同时也提供一种可根据权重灵活调整负载的方法。
 
### 三、负载算法分析
参考上面的Random源码，可明白在调用FailoverClusterInvoker的doInvoke方法之前，会基于router参考结合SPI获取loadBanlance实例，而之后在则会基于loadbalance的select方法获取调用器invoker实例。dubbo提供了4种负载均衡算法 ，分别对应AbstractLoadBalance接口的4个实现类：ConsistentHashLoadBalance（一致性哈希）、LeastActiveLoadBalance（最小活跃）、RandomLoadBalance（均等随机）、RoundRobinLoadBalance（取模轮循），同时均会结合权重进行负载。
#### 3.1 ConsistentHashLoadBalance（一致性哈希）


### 扩展知识点：
#### 服务提供者配置Mock时会同时在zk的/dubbo/com.aoe.demo.rpc.dubbo.DubboExampleInterf1/providers中注册两个provider，即两个服务提供者，具体如下：
```
<dubbo:service interface="com.aoe.demo.rpc.dubbo.DubboExampleInterf1" mock="com.aoe.demo.rpc.dubbo.DubboExampleInterf1Mock" ref="dubboExampleService1" application="rpc-server"  protocol="dubbo"  version="1.0.1-aoe" group="rpc-demo"></dubbo:service>
```
zk服务2个提供者：
```
dubbo%3A%2F%2F100.119.69.1%3A20881%2Fcom.aoe.demo.rpc.dubbo.DubboExampleInterf12
%3Fanyhost%3Dtrue%26application%3Drpc-server%26default.timeout%3D1000%26dubbo%3D
2.5.3%26group%3Drpc-demo%26interface%3Dcom.aoe.demo.rpc.dubbo.DubboExampleInterf
1%26methods%3DserviceProvider%26pid%3D13880%26revision%3D0.0.1-SNAPSHOT%26side%3
Dprovider%26timestamp%3D1569373873898%26version%3D1.0.1-aoe

dubbo%3A%2F%2F100.119.69.1%3A20881%2Fcom.aoe.demo.rpc.dubbo.DubboExampleInterf1%
3Fanyhost%3Dtrue%26application%3Drpc-server%26default.timeout%3D1000%26dubbo%3D2
.5.3%26group%3Drpc-demo%26interface%3Dcom.aoe.demo.rpc.dubbo.DubboExampleInterf1
%26methods%3DserviceProvider%26mock%3Dcom.aoe.demo.rpc.dubbo.DubboExampleInterf1
Mock%26pid%3D13880%26revision%3D0.0.1-SNAPSHOT%26side%3Dprovider%26timestamp%3D1
569373869639%26version%3D1.0.1-aoe

```

