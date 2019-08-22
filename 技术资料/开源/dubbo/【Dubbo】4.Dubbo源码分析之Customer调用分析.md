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
- 结合上一篇笔记Consumer的初始化分析，可知道ReferenceConfig的getObject返回的实例为：实现了ClassGenerator.DC, EchoService, DubboExampleInterf1 3个接口的实现类（proxy0）。
- proxy0类对应$echo方法（EchoService接口）、serviceProvider方法则是实际调用InvokerInvocationHandler(invoker)的invoke方法（即动态代理方式），其内部由是调用invoke对象（即MockClusterInvoker）的invoke方法
- MockClusterInvoker的invoke方法：1.若是非Mock场景则是调用FailoverClusterInvoker（实际为FailoverClusterInvoker父类AbstractClusterInvoker）的invoker方法（因为外层的invoker实际是MockClusterInvoker对FailoverClusterInvoker的包装；2.若是Mock场景则直接调用doMockInvoke方法

### 一.基于Mock场景分析 
那么mock怎么配置呢？首先看看ReferenceBean类的父类AbstractMethodConfig，其定义了String类型的mock。而在解析xml时则会根据xml标签mock配置实例化ReferenceBean实例，而在ReferenceBean的间接父类 AbstractInterfaceConfig
（其是AbstractMethodConfig的子类）的checkStubAndMock方法中，会以interface为参数完成如下验证：
```languag
      if (ConfigUtils.isNotEmpty(mock)) {
            //1.若mock值为 return1，即此处截取值并通过MockInvoker.parseMockValue返回对应Mock对象（可支持null、String、数值、Map、List等类型
            if (mock.startsWith(Constants.RETURN_PREFIX)) {
                String value = mock.substring(Constants.RETURN_PREFIX.length());
                try {
                    MockInvoker.parseMockValue(value);
                } catch (Exception e) {
                    throw new IllegalStateException("Illegal mock json value in <dubbo:service ... mock=\"" + mock + "\" />");
                }
            } else {
                //2.检查mock值是否为default或者true，若是则加载interfaceClassMock；否则直接以mock作为class名称加载对应class
                Class<?> mockClass = ConfigUtils.isDefault(mock) ? ReflectUtils.forName(interfaceClass.getName() + "Mock") : ReflectUtils.forName(mock);
                if (! interfaceClass.isAssignableFrom(mockClass)) {
                    throw new IllegalStateException("The mock implemention class " + mockClass.getName() + " not implement interface " + interfaceClass.getName());
                }
                try {
                    //基于class通过反射获取其无参数且public可访问的构造方法（后续会反射实例化bean，此处仅做验证）
                    mockClass.getConstructor(new Class<?>[0]);
                } catch (NoSuchMethodException e) {
                    throw new IllegalStateException("No such empty constructor \"public " + mockClass.getSimpleName() + "()\" in mock implemention class " + mockClass.getName());
                }
            }
        }
```


```language
public class InvokerInvocationHandler implements InvocationHandler {

    private final Invoker<?> invoker;
    
    public InvokerInvocationHandler(Invoker<?> handler){
        this.invoker = handler;
    }

    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        String methodName = method.getName();
        Class<?>[] parameterTypes = method.getParameterTypes();
        if (method.getDeclaringClass() == Object.class) {
            return method.invoke(invoker, args);
        }
        if ("toString".equals(methodName) && parameterTypes.length == 0) {
            return invoker.toString();
        }
        if ("hashCode".equals(methodName) && parameterTypes.length == 0) {
            return invoker.hashCode();
        }
        if ("equals".equals(methodName) && parameterTypes.length == 1) {
            return invoker.equals(args[0]);
        }
        return invoker.invoke(new RpcInvocation(method, args)).recreate();
    }

}
```
因当前验证Mock模式，故无需启动rpc-server应用，可配置dubbo:reference的check="false"避免实例化dubbo ReferenceBean时检查是否有可用的Provider。通过上面的源码分析，如@Autowired private DubboExampleInterf1 dubboExampleService1；时，dubboExampleService1实例为proxy0实例（其实现DubboExampleInterf1接口），
