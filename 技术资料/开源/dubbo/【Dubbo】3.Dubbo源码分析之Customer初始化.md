Dubbo Customer端dubbo xml配置
```language
	<dubbo:application name="rpc-client" />
    <dubbo:registry address="zookeeper://127.0.0.1:2181" />
	<dubbo:reference id="dubboExampleService1" interface="com.aoe.demo.rpc.dubbo.DubboExampleInterf1" />
```
