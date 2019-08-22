- 通过第3篇分析，已可正常完成dubbo 消费端调用服务端示例。消费端默应用rpc-client启动时会默认从zk中获取dubbo服务信息并检验是否有可用的dubbo service。若启动检查无可用dubbo serviec则会抛出异常但应用可正常启动，但调用时则出现如下错误信息：
```
com.alibaba.dubbo.rpc.RpcException: Forbid consumer 100.119.69.44 access service com.aoe.demo.rpc.dubbo.DubboExampleInterf1 from registry 127.0.0.1:2181 use dubbo version 2.5.3, Please check registry access list (whitelist/blacklist).
	com.alibaba.dubbo.registry.integration.RegistryDirectory.doList(RegistryDirectory.java:579)
	com.alibaba.dubbo.rpc.cluster.directory.AbstractDirectory.list(AbstractDirectory.java:73)
	com.alibaba.dubbo.rpc.cluster.support.AbstractClusterInvoker.list(AbstractClusterInvoker.java:260)
	com.alibaba.dubbo.rpc.cluster.support.AbstractClusterInvoker.invoke(AbstractClusterInvoker.java:219)
	com.alibaba.dubbo.rpc.cluster.support.wrapper.MockClusterInvoker.invoke(MockClusterInvoker.java:72)
	com.alibaba.dubbo.rpc.proxy.InvokerInvocationHandler.invoke(InvokerInvocationHandler.java:52)
	com.alibaba.dubbo.common.bytecode.proxy0.serviceProvider(proxy0.java)
	com.aoe.demo.rpc.controller.EntryController.entry(EntryController.java:29)

```
另外，结合上一篇笔记Consumer的初始化分析，可知道ReferenceConfig的getObject返回的实例为：实现了ClassGenerator.DC, EchoService, DubboExampleInterf1 3个接口的实现类（proxy0）。
