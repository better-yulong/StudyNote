Dubbo Customer端dubbo xml配置
```language
	<dubbo:application name="rpc-client" />
    <dubbo:registry address="zookeeper://127.0.0.1:2181" />
	<dubbo:reference id="dubboExampleService1" interface="com.aoe.demo.rpc.dubbo.DubboExampleInterf1" />
```
class com.alibaba.dubbo.config.spring.ReferenceBean

[async="false", id="dubboExampleService1", interface="com.aoe.demo.rpc.dubbo.DubboExampleInterf1", timeout="0", version="0.0.0"]

