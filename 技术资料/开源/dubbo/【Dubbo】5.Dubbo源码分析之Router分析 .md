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
nterf1/providers，确实

