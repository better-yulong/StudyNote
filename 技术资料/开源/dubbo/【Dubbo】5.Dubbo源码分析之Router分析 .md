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
AvailableClusterInvoker   AbstractClusterInvoker
```language
   
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
       //同步
        RpcUtils.attachInvocationIdIfAsync(getUrl(), invocation);
        return doInvoke(invocation, invokers, loadbalance);
    }
```



interface com.aoe.demo.rpc.dubbo.DubboExampleInterf1 -> registry://10.118.239.202:3181/com.alibaba.dubbo.registry.RegistryService?application=rpc-client&backup=10.118.239.202:3182,10.118.239.202:3183&cluster=available&dubbo=2.5.3&pid=5888&refer=application%3Drpc-client%26default.group%3Drpc-demo%26default.version%3D1.0.1-aoe%26dubbo%3D2.5.3%26interface%3Dcom.aoe.demo.rpc.dubbo.DubboExampleInterf1%26methods%3DserviceProvider%26pid%3D5888%26revision%3D0.0.1-SNAPSHOT%26side%3Dconsumer%26timeout%3D1000%26timestamp%3D1568626754368&registry=zookeeper&timestamp=1568626754675