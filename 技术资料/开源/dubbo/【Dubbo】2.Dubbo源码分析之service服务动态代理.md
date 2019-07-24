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

### Dubbo SPI架构
- 开始有尝试调试如上代码，但感觉很难对整体有一个清晰的思路；于是决定换个角度，先从整体理解下Dubbo SPI的整体架构（Dubbo 的微内核设计，可参考资料：https://my.oschina.net/j4love/blog/1813040、https://www.jianshu.com/p/7daa38fc9711）。
- 从上面资料，可将ExtensionLoader对比

