Dubbo2.7由apache维护，因公司使用的仍是com.alibaba的Dubbo版本，故选择bubbo2.5.3 作为此次分析：github:https://github.com/apache/dubbo（选择tag:2.5.3）,doc:http://dubbo.apache.org/zh-cn/docs/dev/release.html

### 一.准备
#### 1.1 环境准备
#### 1.1.1 dubbo源码下载
eclipse基于https://github.com/apache/dubbo，选择2.5.3为初始化版本，创建本地仓库。Eclipse中import git仓库代码并copy至workspace。
#### 1.1.2 dubbo intalll至本地仓库
1.  dubbo解压缩后/目录运行mvn clean install -Dmaven.test.skip ，报错提示：opensesame相关报错。https://github.com/alibaba/opensesame 下载opensesame 的zip包（该包为alibaba开源项目的顶级父类项目），解压opensesame的zip包在其/目录运行 mvn clean install -Dmaven.test.skip ，相对比较顺利：opensesame成功编译并被安装至本地；
2. 再次切到dubbo /目录运行mvn clean install -Dmaven.test.skip，但又发现缺少hessian-lite依赖，于是首各种资料（尝试找源码编译、找可用的maven仓库），花了不少时间根据资料（https://github.com/apache/dubbo/issues/21）在mavne的 settings.xml文件添加mirror源（可同时解决opensesame及hessian-lite问题；即第1步的手动编译可忽略）：
```language
    <mirror>  
        <id>cuisongliu</id>  
        <mirrorOf>*</mirrorOf>    
        <name>cubisongliu</name>  
        <url>http://maven.cuisongliu.com/content/groups/public</url>  
    </mirror> 
```
3. 步骤2完成后，再次切到dubbo /目录运行mvn clean install -Dmaven.test.skip，此时运行发现dubbo-common包中的JSONTest.java测试方法报错无法编译通过。同样百度各种尝试如手动编译（却发现无源码）或更换fastjson版本号，各种不爽；无奈之下，因其是Test方法那粗暴点注释该方法，重新编译还真OK。于是乎后面同样的方式处理ClientReconnectTest报错。
4. 步骤3运行一段时间之后再次中断，系统报错原因：PermGen space 溢出；该问题看似比较简单，但在我本地还是花了点时间，常用方法即是修改mvn.bat（windows平台）的MAVEN_OPTS参数（set MAVEN_OPTS=-Xms512m -Xmx2048m -XX:MaxPermSize=256m）。然额，修改后使用命令mvn -v查看提示Error，而仅修改了该参数；之后研究其他方式配置或者调整jdk版本，但并没生效，经过折腾及尝试，mvn.bat调整为：
```language
set MAVEN_OPTS=-XX:MaxPermSize=256m  //同时配置Xms、Xmx就不行，仅配置这一个确实可生效，先解决暂不定位原因
set JAVA_HOME=D:\work\java\JDK6 //考虑到jdk兼容，采用JDK6编译，因为demo工程基本也是使用的JRE6
```
4. 同步骤3解决NettyClientTest编译报错及其他类似报错，耗时13分钟终于OK。--其实整个还是很花了点时间

#### 1.1.2 dubbo 源码导入至eclipse
解决错误

#### 1.1.3  zookeeper
1. windows版本（zookeeper-3.3.6.tar；已上传至StudyNote-Resource）
2. 解压缩之后进入conf目录，复制zoo_simple.cfg文件为zoo.cfg,并指定端口号、数据存储目录等
```language
# The number of milliseconds of each tick
tickTime=2000
# The number of ticks that the initial 
# synchronization phase can take
initLimit=10
# The number of ticks that can pass between 
# sending a request and getting an acknowledgement
syncLimit=5
# the directory where the snapshot is stored.
dataDir=E:/study/zkdata
# the port at which the clients will connect
clientPort=2181
```
3. 运行bin目录下的zkServer即可启动zookeeper；另外也可运行同目录下zkCli进行客户端命令行模式（本地运行IP：127.0.0.1；端口2181)

### 二.dubbo 服务端配置及示例验证
dubbo系统示例工程沿用Spring&SpirngMVC分析时hessian示例使用的三个工程:rpc-server、rpc-client、rpc-skeleton.
#### 2.1 配置方式
根据dubbo的官方文档（http://dubbo.apache.org/zh-cn/docs/user/configuration/xml.html）， dubbo支持多种配置：XML配置、API配置、注解配置，当前示例先选择xml配置方式
##### 2.1.1 dubbo 服务接口（rpc-skeleton)
```language
package com.aoe.demo.rpc.dubbo;

import java.util.List;

public interface DubboExampleInterf1 {
	
	public List serviceProvider(List params);

}
```
##### 2.1.2 dubbo服务实现(rpc-server)
######  2.1.2.1 DubboExampleService1服务实现
```language
public class DubboExampleService1 implements DubboExampleInterf1 {

	public List serviceProvider(List params) {
		System.out.println("DubboExampleService1 param0:" + params.get(0));
		List serviceNames = new ArrayList<String>();
		serviceNames.add("DubboExampleService1");
		return serviceNames ;
	}
}
```
######  2.1.2.2 xml配置
参考官网添加xml头(xmlns、xsd），之后基于dubbo 标签完成bean及dubbo相关配置，然而启动后始终报错：
org.springframework.beans.factory.parsing.BeanDefinitionParsingException: Configuration problem: Unable to locate Spring NamespaceHandler for XML schema namespace [http://dubbo.apache.org/schema/dubbo]。
- 最初基于官网配置，并未意识到未引入dubbo 的jar包；
开始没太明白，初步怀疑是xml头文件修改时出现错误，然而多次确认发现并无明显错误；于是基于之前自定义NamespaceHanlder的经验，于是全文搜索"dubbo.apache.org/schema/dubbo"确认对应hanlder配置发现并没有找到（步骤一下载的dubbo源码并无匹配项）。本来还有些许疑虑,于是换个思路分析 *.halder文件中关于Namespace配置，至此才恍然大悟，官网基于apache dubbo配置，而当前dubbo版本为alibaba版本：
```language
http\://code.alibabatech.com/schema/dubbo=com.alibaba.dubbo.config.spring.schema.DubboNamespaceHandler
```
调整xml头并添加dubbo服务配置：
```language
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
		xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
		xmlns:context="http://www.springframework.org/schema/context"
		xmlns:dubbo="http://code.alibabatech.com/schema/dubbo"
		xmlns:aop="http://www.springframework.org/schema/aop"
		xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-2.5.xsd
		        http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop.xsd
				http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context-2.5.xsd
				http://code.alibabatech.com/schema/dubbo http://code.alibabatech.com/schema/dubbo/dubbo.xsd">
	
	<context:component-scan base-package="com.aoe.demo.rpc"/>
	
	<bean name="hessianExampleService1"  class="com.aoe.demo.rpc.hessian.HessianExampleService1"></bean>
	
	<bean name="/hessianExampleService1" class="org.springframework.remoting.caucho.HessianServiceExporter">
	    <property name="service" ref="hessianExampleService1"/>
	    <property name="serviceInterface" value="com.aoe.demo.rpc.hessian.HessianExampleInterf1"/>
	</bean>
	
	<dubbo:application name="rpc-server"></dubbo:application>
	<dubbo:registry address="zookeeper://127.0.0.1:2181"></dubbo:registry>
	<dubbo:protocol name="dubbo" port="20890"></dubbo:protocol>
	<bean id="dubboExampleService1" class="com.aoe.demo.rpc.dubbo.DubboExampleService1"></bean>
	<dubbo:service interface="com.aoe.demo.rpc.dubbo.DubboExampleInterf1" ref="dubboExampleService1"></dubbo:service>
</beans>
```
######  2.1.2.3 pom.xml配置
添加dubbo、zk依赖包
```language
		<dependency>
			<groupId>com.alibaba</groupId>
			<artifactId>dubbo</artifactId>
			<version>2.5.3</version>
			<exclusions>
				<exclusion>
					<groupId>org.springframework</groupId>
					<artifactId>spring</artifactId>
					<!-- 可以使用*号全部排除 -->
					<!-- <groupId>*</groupId> <artifactId>*</artifactId> -->
				</exclusion>
			</exclusions>
		</dependency>
		<dependency>
			<groupId>com.github.sgroschupf</groupId>
			<artifactId>zkclient</artifactId>
			<version>0.1</version>
		</dependency>
```
- 有两点需要注意：
1. 引入dubbo 2.5.3 之后启动应用会提示spring某个class对应method找不到，即NoSuchMethodFoundException；经分析因dubbo 自动依赖2.5 版本的spring包; 而之前框架整体依赖Spring 3.1.0，导致spring版本不兼容，故需排除dubbo引入 的spring包。但是又不可采用*全部排除，因为dubbo 包依赖项目依赖的dubbo-config、dubbo-remoting等包，即按需排除即可；
2. 若不引入zk的包，启动时会提示找不到zk相关的class，根据pom.xml确认dubbo确实没有依赖zk包，需此处需单独配置zk依赖jar 
######  2.1.2.4 启动rpc-server 应用
上面提示的部分问题也是启动过程出现的，但逐步解决，另外还有两点：
1. 运行时提示找不到CurrentHashMap的某方法，根据经验系class 对应的jdk不一致不兼容；遂逐个将dubbo及rpc项目的jdk编译版本统一指定为JDK6（其实也主要因为我本地有多个jdk版本，从JDK5至JDK8）
2. 启动时系统假死，通过线程栈可发现是因为尝试连接zookeeper，启动zk即可。
######  2.1.2.5 查看zookeeper注册信息
运行zookeeper bin目录中的zkCli.cmd文件即会可自动连接至zookeeper（命令行模式)。使用 ls / 可发现根节点新节dubbo节点；使用ls /dubbo 可发现节点 [com.aoe.demo.rpc.dubbo.DubboExampleInterf1] 注册成功；继续查看可看到当前服务接口相关的服务使用者、提供者、路由、配置等各种信息:
```language
[zk: localhost:2181(CONNECTED) 0] ls /dubbo/com.aoe.demo.rpc.dubbo.DubboExampleI
com.aoe.demo.rpc.dubbo.DubboExampleInterf1/
consumers       configurators   routers         providers

[zk: localhost:2181(CONNECTED) 1] ls /dubbo/com.aoe.demo.rpc.dubbo.DubboExampleI
nterf1/consumers
[consumer%3A%2F%2F100.119.69.32%2Fcom.aoe.demo.rpc.dubbo.DubboExampleInterf1%3Fa
pplication%3Drpc-client%26category%3Dconsumers%26check%3Dfalse%26dubbo%3D2.5.3%2
6interface%3Dcom.aoe.demo.rpc.dubbo.DubboExampleInterf1%26methods%3DserviceProvi
der%26pid%3D13576%26revision%3D0.0.1-SNAPSHOT%26side%3Dconsumer%26timestamp%3D15
63442468270]
```
 
### 三.dubbo客户端示例及验证
根据之前分析源码经验，同时dubbo 历史版本的文档官网已比较难找，于是乎既然又源码了，那就直接从源码着手。参考dubbo-demo-consumer.xml配置rpc-client的spring xml配置文件（同rpc-server添加xml头）：
```language
	<dubbo:reference id="dubboExampleService1" interface="com.aoe.demo.rpc.dubbo.DubboExampleInterf1" />
``` 
一看感觉就少了什么东西（从哪儿获取注册的服务？），咋整？先不管，启动rpc-client会发现提示未配置dubbo:application，之后同样会提示未配置dubbo:registry，参考源码示例配置修改spring xml配置：
```language
   <dubbo:application name="rpc-client" />
   <dubbo:registry address="zookeeper://127.0.0.1:2181" />
   <dubbo:reference id="dubboExampleService1" interface="com.aoe.demo.rpc.dubbo.DubboExampleInterf1" />
```
运再次启动，还有俩问题：1.zookeeper未启动，启动rpc-client应用也会卡住，通过栈信息即可确定是无法连接zookeeper；2.zookeeper启动，但zk中无服务注册信息或者rpc-server未启动，启动均会报错提示无可用的服务提供者
```language
Failed to check the status of the service com.aoe.demo.rpc.dubbo.DubboExampleInterf1. No provider available for the service com.aoe.demo.rpc.dubbo.DubboExampleInterf1 from the url zookeeper://127.0.0.1:2181/com.alibaba.dubbo.registry.RegistryService?application=rpc-client&dubbo=2.5.3&interface=com.aoe.demo.rpc.dubbo.DubboExampleInterf1&methods=serviceProvider&pid=13576&revision=0.0.1-SNAPSHOT&side=consumer&timestamp=1563442468270 to the consumer 100.119.69.32 use dubbo version 2.5.3
```
- 添加服务调用示例代码
```language
@Controller
public class EntryController {
	
	@Autowired
	private HessianExampleInterf1 hessianExampleService;
	
	@Autowired
	private DubboExampleInterf1 dubboExampleService1;
	
	@RequestMapping("/entry")
	@ResponseBody
	public Object entry(){
		List<String> params =  new ArrayList<String>();
		params.add("parm");
		System.out.println("hessianExampleService result0:" + hessianExampleService.getServiceNames(params).get(0));
		System.out.println("dubboExampleService1 result0:" + dubboExampleService1.serviceProvider(params).get(0));
		return "entry";
	}

}
```
解决上面所有问题，依次启动zk、rpc-server、rpc-client，启动过程正常并无报错，然后调用entry方法入口，验证通过。

### 四.dubbo服务端源码分析
示例工程rpc-server spring的xm中有关于dubbo的配置（暂无其他配置）：
```language
	<dubbo:application name="rpc-server"></dubbo:application>
	<dubbo:registry address="zookeeper://127.0.0.1:2181"></dubbo:registry>
	<dubbo:protocol name="dubbo" port="20890"></dubbo:protocol>
	<bean id="dubboExampleService1" class="com.aoe.demo.rpc.dubbo.DubboExampleService1"></bean>
	<dubbo:service interface="com.aoe.demo.rpc.dubbo.DubboExampleInterf1" ref="dubboExampleService1"></dubbo:service>
```
结合之前分析经验，那肯定需先从自定义命名空间dubbo入手；参考Dubbo官方文档（http://dubbo.apache.org/zh-cn/docs/user/quick-start.html）：
> Dubbo 采用全 Spring 配置方式，透明化接入应用，对应用没有任何 API 侵入，只需用 Spring 加载 Dubbo 的配置即可，Dubbo 基于 Spring 的 Schema 扩展 进行加载（spring中常用的自定义标签：context、aop等实际都此用方式实现）。如果不想使用 Spring 配置，可以通过 API 的方式 进行调用（API使用范围说明：API 仅用于 OpenAPI, ESB, Test, Mock 等系统集成，普通服务提供方或消费方，请采用XML 配置方式使用 Dubbo）。
#### 4.1 dubbo schema扩展分析
- 根据xml文件头dubbo的 xmlns:dubbo="http://code.alibabatech.com/schema/dubbo",可根据code.alibabatech.com/schema/dubbo搜索，即可在dubbo-config-spring工程META-INF\spring.schemas找到对应配置：
> http\://code.alibabatech.com/schema/dubbo/dubbo.xsd=META-INF/dubbo.xsd；
- 同时亦可在dubbo-config-spring工程META-INF\spring.handlers找到handler配置：
> http\://code.alibabatech.com/schema/dubbo=com.alibaba.dubbo.config.spring.schema.DubboNamespaceHandler
- 其实，根据之前分析命名一般会默认规范，如ContextNamespaceHandler、DubboNamespaceHandler；解析spring xml时遇到自定义标签dubbo时，会查找其对应的NamespaceHanlder类实例化后调用其init方法（spring scheme扩展），完成当前dubbo命名空间对应标签的BeaDefinitionParser类，而xml解析遇到对应标签节点则会通过对应的BeaDefinitionParser来完成解析
- #### 4.1 DubboNamespaceHandler源码
```language
public class DubboNamespaceHandler extends NamespaceHandlerSupport {

	static {
		Version.checkDuplicate(DubboNamespaceHandler.class);
	}

	public void init() {
	    registerBeanDefinitionParser("application", new DubboBeanDefinitionParser(ApplicationConfig.class, true));
        registerBeanDefinitionParser("module", new DubboBeanDefinitionParser(ModuleConfig.class, true));
        registerBeanDefinitionParser("registry", new DubboBeanDefinitionParser(RegistryConfig.class, true));
        registerBeanDefinitionParser("monitor", new DubboBeanDefinitionParser(MonitorConfig.class, true));
        registerBeanDefinitionParser("provider", new DubboBeanDefinitionParser(ProviderConfig.class, true));
        registerBeanDefinitionParser("consumer", new DubboBeanDefinitionParser(ConsumerConfig.class, true));
        registerBeanDefinitionParser("protocol", new DubboBeanDefinitionParser(ProtocolConfig.class, true));
        registerBeanDefinitionParser("service", new DubboBeanDefinitionParser(ServiceBean.class, true));
        registerBeanDefinitionParser("reference", new DubboBeanDefinitionParser(ReferenceBean.class, false));
        registerBeanDefinitionParser("annotation", new DubboBeanDefinitionParser(AnnotationBean.class, true));
    }

}
```
##### 4.1.1 Version.checkDuplicate(DubboNamespaceHandler.class)
- 代码逻辑相对简单，即根据class全限定名：com/alibaba/dubbo/config/spring/schema/DubboNamespaceHandler.class在Version.class所在ClassCloader查找：ClassHelper.getCallerClassLoader(Version.class).getResources(path);判断是否有重复class（其中path即为class的全限定名；ClassLoader有两个名称极其相似的方法：getResources、getResource，仅最后一个s的差别）
- 该类主要Version.checkDuplicate主要用于检查是否存在重复的jar包
#### 4.2 各标签解析
- 结合DubboNamespaceHandler及之前其他源码分析，此处完成xml的解析，根据标签生成ApplicationConfig、RegistryConfig、ProviderConfig等类的BeanDefinition，并调用parserContext.getRegistry().registerBeanDefinition(id, beanDefinition)注册。而在Spring容器实例化bean时会从Registry根据id获取对应的beanDefinition(getBeayn方法)---具体各标签后面另行讲解
- 既然此处将xml解析生成各种BeanDefinition，那么根据之前分析spring过程，对应的Bean实例化之后可通过类似initial方式或者Processor方式来处理，于是乎调试看看。发现调试时始终无法关联到当前eclipse中的spring源码，而是关联到spring jar中的class反编译文件；定位发现dubbo工程自动依赖了之前提到的spring 2.5的jar,于是乎强制将各dubbo源码工程对spring 2.5 的依赖强制升级到spring 3.1.0 
#### 4.3 bean实例化分析
根据DubboNamespaceHandler可发现，涉及dubbo的标签会实例化ApplicationConfig、ModuleConfig、RegistryConfig、MonitorConfig、ProviderConfig、ConsumerConfig、ProtocolConfig、ServiceBean、ReferenceBean、AnnotationBean 这10类bean；而前面7种相对简单，即可认为是简单数据对象；而ServiceBean、ReferenceBean、AnnotationBean则不同。
##### 4.3.1 ServiceBean(dubbo:service)

```language
	<bean id="dubboExampleService1" class="com.aoe.demo.rpc.dubbo.DubboExampleService1"></bean>
	<dubbo:service interface="com.aoe.demo.rpc.dubbo.DubboExampleInterf1" ref="dubboExampleService1"></dubbo:service>
```
```language
public class ServiceBean<T> extends ServiceConfig<T> implements InitializingBean, DisposableBean, ApplicationContextAware, ApplicationListener, BeanNameAware
```
关于ServiceBean实现的接口之前分析Spring源码时曾有讲解过，基本作用相对比较清楚，即解析dubbo:service时会化BeanDefinition并完成注册（beanName默认取interface值，如com.aoe.demo.rpc.dubbo.DubboExampleInterf1），那么在完成com.aoe.demo.rpc.dubbo.DubboExampleInterf1对应的ServiceBean原始bean实例化后会依次调用：
###### 1.ServiceBean类setBeanName方法
即对应DefaultListableBeanFactory(AbstractAutowireCapableBeanFactory).invokeAwareMethods(final String beanName, final Object bean)；即当前bean为BeanNameAware接口的实例，自动设置当前bean的beanName（值为：com.aoe.demo.rpc.dubbo.DubboExampleInterf1）

###### 2.ServiceBean类setApplicationContext方法（ApplicationListener）
当前bean为ApplicationContextAware接口的实例（故步骤1的invokeAwareMethods中也会判断当前bean是否为ApplicationContextAware接口的实例），自动设置当前bean的applicationContext（即Spring容器的上下文），同时因当前ServiceBean实现ApplicationListener，该方法内会同时反射调用applicationContext的addApplicationListener方法，将当前ServiceBean实例添加至Listeners列表；同时此处也会将applicationContext设置到dubbo扩展实现的SpringExtensionFactory.addApplicationContext(applicationContext)。

###### 3.ServiceBean类afterPropertiesSet()方法（对应InitializingBean）,即对ServiceBean 实例的初始化配置
- dubbo:provider为dubbo:service提供缺省配置;dubbo:protocol指定协议配置.
- 每个dubbo:provider会对应一个ProviderConfig实例（服务提供方最大可接受连接数accepts、请求及响应数据包大小限制payload、协议编码方式codec、协议序列化方式、serialization、提供者上下文路径path、服务端协议、Register等）;基于provider、protocol可指定全局配置(此处protocol仅是兼容处理；后面会在标准化处理)，也可针对单个dubbo:service精细化配置.
###### 3.1 对应provider配置(对应dubbo:provider或者provider--ProviderConfig)，实际有三种（结合源码示例）：
1. 全局provider配置（即对应ServiceBean类afterPropertiesSet()方法的处理全局provider配置）
```language
<dubbo:provider timeout="1000" protocol="dubbo"></dubbo:provider>
	<dubbo:service interface="com.aoe.demo.rpc.dubbo.DubboExampleInterf1" ref="dubboExampleService1" ></dubbo:service>
```

2. 单个dubbo:service指定provider（嵌套标签实现；apache dubbo可采用类似d protocol方式来指定，无需嵌套标签）
```
<dubbo:protocol name="dubbo" port="20890"></dubbo:protocol>
<dubbo:provider timeout="1000" protocol="dubbo">
	      <dubbo:service interface="com.aoe.demo.rpc.dubbo.DubboExampleInterf1" ref="dubboExampleService1"  protocol="dubbo"></dubbo:service>
	</dubbo:provider>
```
此种方式会在ServiceBean原始bean实例后设置properties属性时完成provider属性的赋值（ServiceBean从父类ServiceConfig类继承provider属性）
3. 不配置provider：后续bean实例化后期会针对ServiceBean实例无provider时实例化默认的ProiderConfig对象。

###### 3.2 对应application配置(对应dubbo:application或者application--ApplicationConfig)，实际有三种（结合源码示例）：
1. 全局application配置（即对应ServiceBean类afterPropertiesSet()方法的处理全局application配置；此处注意还有个前提：即provider没有指定application，若已指定则忽略全局application），仅用于注册中心计算应用间依赖关系（不影响服务调用）
```language
  <dubbo:application name="rpc-server"></dubbo:application>
```
2. 单个dubbo:service指定关联applacition
```language
<dubbo:application name="rpc-server"></dubbo:application>
<dubbo:service interface="com.aoe.demo.rpc.dubbo.DubboExampleInterf1" ref="dubboExampleService1" application="rpc-server" ></dubbo:service>
```
同provider，此种方式会在ServiceBean原始bean实例后设置properties属性时完成applacition属性的赋值（ServiceBean从间接父类AbstractInterfaceConfig类继承applacition属性）
3. 不配置applacition：后续bean实例化后期会针对ServiceBean实例无applacition时实例化默认的ApplicationConfig对象。
###### 3.3 对应module配置(对应dubbo:module或者module--ModuleConfig)，方式同上（可不配置、全局配置、Provider配置、Service配置）；module配置主要用于模块信息配置，用于注册中心计算模块间依赖关系
###### 3.4 对应registry配置(对应dubbo:registry或者registry--RegistryConfig)，方式同上（可不配置、全局配置、Provider配置、Application配置、Service配置）；registry配置主要用于注册中心配置。
###### 3.5 对应monitor配置(对应dubbo:monitor或者monitor--MonitorConfig)，方式同上（可不配置、全局配置、Provider配置、Application配置、Service配置）；monitor配置主要用于监控中心配置。
###### 3.6 对应protocol配置(对应dubbo:protocol或者protocol--ProtocolConfig)，方式同上（可不配置、全局配置、Provider配置、Service配置）；protocol配置主要用于服务提供者协议配置；如果需要支持多协议，可以声明多个<dubbo:protocol>标签，并在<dubbo:service>中通过protocol属性指定使用的协议。说明：如果需要支持多协议，可以声明多个<dubbo:protocol>标签，并在<dubbo:service>中通过protocol属性指定使用的协议（通讯协议
序列化协议）。
###### 3.7 对应path配置（对应dubbo:service path="")，方式同上（可不配置、Provider配置、Service配置）；path配置用于服务路径配置。
###### 3.8 如若当前ServiceBean服务非延迟注册（provider、service设置delay)，则调用SreviceConfig类（ServiceBean父类)的export方法（稍后延迟注册也会分析到export方法）

##### 4.ServiceBean类onApplicationEvent方法
- ServiceBean实现ApplicationListener，即需实现onApplicationEvent方法。而spring容器初始化完成之后调用finishRefresh，会经由SimpleApplicationEventMulticaster.multicastEvent(ApplicationEvent)广播事件ContextRefreshedEvent，同时会判断当前ServiceBean服务是否延迟注册（provider、service设置delay)、是否已注册（ServiceBean服务注册后会修改exported状态）、是否已卸载（ServiceBean服务的destroy方法可完成服务卸载：isUnexported）
- 延迟注册且未注册则调用SreviceConfig类（ServiceBean父类)的export方法（非延迟注册的服务已于上一步完成服务注册）：
```language
    public synchronized void export() {
        if (provider != null) {
            if (export == null) {
                export = provider.getExport();
            }
            if (delay == null) {
                delay = provider.getDelay();
            }
        }
        if (export != null && ! export.booleanValue()) {
            return;
        }
        if (delay != null && delay > 0) {
            Thread thread = new Thread(new Runnable() {
                public void run() {
                    try {
                        Thread.sleep(delay);
                    } catch (Throwable e) {
                    }
                    doExport();
                }
            });
            thread.setDaemon(true);
            thread.setName("DelayExportServiceThread");
            thread.start();
        } else {
            doExport();
        }
    }
```
export方法：判断当前ServiceBean(SreviceConfig)的export及delay配置（会判断状态避免重复注册），之后根据delay配置判断：1.若delay有配置则启动异步Demon线程sleep指定delay时间后调用doExport方法；2.无delay配置则直接调用doExport方法。
- doExport方法(SreviceConfig类）：1、验证是否已卸载 、已注册、interfaceName是否不配置；2、检查并实例化ServiceBean对象默认的ProviderConfig及完成provider对象初始化；3、根据provider对象及当前ServiceBean实例的application、module、registries、monitor、protocols（registries、monitor配置可在module、application配置，但application优先级大于module）；4.检查interfaceClass或interfaceName及类加载（涉及泛化调用，后面分析）；5.local标识判断及类加载（服务接口的本地实现类； 新版本local机制已经废弃,被stub属性所替换）；6.stub配置判断及处理（涉及本地代理调用，后面分析）；6、Application、Registry、Protocol检查及初始属性值设置；7、补充属性值
 appendProperties（当前测试工程无属性增加）；8、AbstractInterfaceConfig类checkStubAndMock(interfaceClass)，该方法内部根据local/stub/mock配置进行本地调试或者mock处理（后面分析）；9、调用doExportUrls方法（ServiceConfig类）
###### 服务发布核心：doExportUrls方法（ServiceConfig类）
```language
    private void doExportUrls() {
        //根据register配置及其他参数组装完整的registerUrl列表
        List<URL> registryURLs = loadRegistries(true);
        //基于protocol完成服务注册
        for (ProtocolConfig protocolConfig : protocols) {
            doExportUrlsFor1Protocol(protocolConfig, registryURLs);
        }
    }
```
- 方法loadRegistries(true)（AbstractInterfaceConfig类；可配置多个registry；包含兼容处理），即根据register配置及其他参数组装完整的registerUrl(如[registry://127.0.0.1:2181/com.alibaba.dubbo.registry.RegistryService?application=rpc-server&dubbo=2.5.3&pid=13916&registry=zookeeper&timestamp=1563867985933]）
- doExportUrlsFor1Protocol(protocolConfig, registryURLs)方法（内容较多，选择性分析
```language
   private void doExportUrlsFor1Protocol(ProtocolConfig protocolConfig, List<URL> registryURLs) {
        String name = protocolConfig.getName();
        if (name == null || name.length() == 0) {
            name = "dubbo";
        }

        String host = protocolConfig.getHost();
        if (provider != null && (host == null || host.length() == 0)) {
            host = provider.getHost();
        }
        boolean anyhost = false;
        if (NetUtils.isInvalidLocalHost(host)) {
            anyhost = true;
            try {
                //获取本机IP（该部分不同的dubbo版本实现会有差异，因为针对多网卡或者获取IP错误场景会导致服务注册异常，可参考：http://dubbo.apache.org/zh-cn/blog/dubbo-network-interfaces.html
               host = InetAddress.getLocalHost().getHostAddress();
            } catch (UnknownHostException e) {
                logger.warn(e.getMessage(), e);
            }
            if (NetUtils.isInvalidLocalHost(host)) {
                if (registryURLs != null && registryURLs.size() > 0) {
                    for (URL registryURL : registryURLs) {
                        try {
                            Socket socket = new Socket();
                            try {
                                SocketAddress addr = new InetSocketAddress(registryURL.getHost(), registryURL.getPort());
                                socket.connect(addr, 1000);
                                host = socket.getLocalAddress().getHostAddress();
                                break;
                            } finally {
                                try {
                                    socket.close();
                                } catch (Throwable e) {}
                            }
                        } catch (Exception e) {
                            logger.warn(e.getMessage(), e);
                        }
                    }
                }
                if (NetUtils.isInvalidLocalHost(host)) {
                    host = NetUtils.getLocalHost();
                }
            }
        }

        Integer port = protocolConfig.getPort();
        if (provider != null && (port == null || port == 0)) {
            port = provider.getPort();
        }
        //根据name从对应的网络协议对象获取默认port配置（如DubboProtocol为20880；后续分析）
        final int defaultPort = ExtensionLoader.getExtensionLoader(Protocol.class).getExtension(name).getDefaultPort();
        if (port == null || port == 0) {
            port = defaultPort;
        }
        if (port == null || port <= 0) {
            port = getRandomPort(name);
            if (port == null || port < 0) {
                //未指定端口在及默认端口，获取可用的随机端口
                port = NetUtils.getAvailablePort(defaultPort);
                putRandomPort(name, port);
            }
            logger.warn("Use random available port(" + port + ") for protocol " + name);
        }

        Map<String, String> map = new HashMap<String, String>();
        if (anyhost) {
            map.put(Constants.ANYHOST_KEY, "true");
        }
        map.put(Constants.SIDE_KEY, Constants.PROVIDER_SIDE);//provider
        map.put(Constants.DUBBO_VERSION_KEY, Version.getVersion());//从MANIFEST.MF规范、jar文件名等获取版本号
        map.put(Constants.TIMESTAMP_KEY, String.valueOf(System.currentTimeMillis()));//获取时间戳为key
        if (ConfigUtils.getPid() > 0) {
            //基于RuntimeMXBean获取当前JVM进程 的PID
            map.put(Constants.PID_KEY, String.valueOf(ConfigUtils.getPid()));
        }
        //通过反射将各对象参数put至map
        appendParameters(map, application);
        appendParameters(map, module);
        appendParameters(map, provider, Constants.DEFAULT_KEY);
        appendParameters(map, protocolConfig);
        appendParameters(map, this);
        //dubbo:method 方法级配置。对应的配置类： org.apache.dubbo.config.MethodConfig。同时该标签为 <dubbo:service> 或 <dubbo:reference> 的子标签，用于控制到方法级(后续分析)。 
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
                List<ArgumentConfig> arguments = method.getArguments();
                if (arguments != null && arguments.size() > 0) {
                    for (ArgumentConfig argument : arguments) {
                        //类型自动转换.
                        if(argument.getType() != null && argument.getType().length() >0){
                            Method[] methods = interfaceClass.getMethods();
                            //遍历所有方法
                            if(methods != null && methods.length > 0){
                                for (int i = 0; i < methods.length; i++) {
                                    String methodName = methods[i].getName();
                                    //匹配方法名称，获取方法签名.
                                    if(methodName.equals(method.getName())){
                                        Class<?>[] argtypes = methods[i].getParameterTypes();
                                        //一个方法中单个callback
                                        if (argument.getIndex() != -1 ){
                                            if (argtypes[argument.getIndex()].getName().equals(argument.getType())){
                                                appendParameters(map, argument, method.getName() + "." + argument.getIndex());
                                            }else {
                                                throw new IllegalArgumentException("argument config error : the index attribute and type attirbute not match :index :"+argument.getIndex() + ", type:" + argument.getType());
                                            }
                                        } else {
                                            //一个方法中多个callback
                                            for (int j = 0 ;j<argtypes.length ;j++) {
                                                Class<?> argclazz = argtypes[j];
                                                if (argclazz.getName().equals(argument.getType())){
                                                    appendParameters(map, argument, method.getName() + "." + j);
                                                    if (argument.getIndex() != -1 && argument.getIndex() != j){
                                                        throw new IllegalArgumentException("argument config error : the index attribute and type attirbute not match :index :"+argument.getIndex() + ", type:" + argument.getType());
                                                    }
                                                }
                                            }
                                        }
                                    }
                                }
                            }
                        }else if(argument.getIndex() != -1){
                            appendParameters(map, argument, method.getName() + "." + argument.getIndex());
                        }else {
                            throw new IllegalArgumentException("argument config must set index or type attribute.eg: <dubbo:argument index='0' .../> or <dubbo:argument type=xxx .../>");
                        }

                    }
                }
            } // end of methods for
        }

        if (generic) {
            map.put("generic", String.valueOf(true));
            map.put("methods", Constants.ANY_VALUE);
        } else {
            //从MANIFEST.MF规范、jar文件名等获取版本号       
            String revision = Version.getVersion(interfaceClass, version);
            if (revision != null && revision.length() > 0) {
                map.put("revision", revision);
            }
            //获取接口方法名称
            String[] methods = Wrapper.getWrapper(interfaceClass).getMethodNames();
            if(methods.length == 0) {
                logger.warn("NO method found in service interface " + interfaceClass.getName());
                map.put("methods", Constants.ANY_VALUE);
            }
            else {
                //将接口方法,分隔字符串put至map
                map.put("methods", StringUtils.join(new HashSet<String>(Arrays.asList(methods)), ","));
            }
        }
        if (! ConfigUtils.isEmpty(token)) {
             //token 访问令牌
            if (ConfigUtils.isDefault(token)) {
                map.put("token", UUID.randomUUID().toString());
            } else {
                map.put("token", token);
            }
        }
        if ("injvm".equals(protocolConfig.getName())) {
            //protocol为injvm，设置不注册、不通知
            protocolConfig.setRegister(false);
            map.put("notify", "false");
        }
        // 导出服务
        String contextPath = protocolConfig.getContextpath();
        if ((contextPath == null || contextPath.length() == 0) && provider != null) {
            contextPath = provider.getContextpath();
        }
        //基于当前各参数组装url，URL为dubbo自定义实现类，其toString方法返回为全部参数拼装值（dubbo://100.119.69.22:20890/com.aoe.demo.rpc.dubbo.DubboExampleInterf1?anyhost=true&application=rpc-server&default.timeout=1000&dubbo=2.5.3&interface=com.aoe.demo.rpc.dubbo.DubboExampleInterf1&methods=serviceProvider&pid=7576&revision=0.0.1-SNAPSHOT&side=provider&timestamp=1563876209932)
        URL url = new URL(name, host, port, (contextPath == null || contextPath.length() == 0 ? "" : contextPath + "/") + path, map);

        if (ExtensionLoader.getExtensionLoader(ConfiguratorFactory.class)
                .hasExtension(url.getProtocol())) {
            //如若是其他自定义协议（dubbo不会运行至此处）
            url = ExtensionLoader.getExtensionLoader(ConfiguratorFactory.class)
                    .getExtension(url.getProtocol()).getConfigurator(url).configure(url);
        }

        String scope = url.getParameter(Constants.SCOPE_KEY);
        //配置为none不暴露
        if (! Constants.SCOPE_NONE.toString().equalsIgnoreCase(scope)) {

            //配置不是remote的情况下做本地暴露 (配置为remote，则表示只暴露远程服务)
            if (!Constants.SCOPE_REMOTE.toString().equalsIgnoreCase(scope)) {
                //后面单独分析
                exportLocal(url);
            }
            //如果配置不是local则暴露为远程服务.(配置为local，则表示只暴露远程服务)
            if (! Constants.SCOPE_LOCAL.toString().equalsIgnoreCase(scope) ){
                if (logger.isInfoEnabled()) {
                    logger.info("Export dubbo service " + interfaceClass.getName() + " to url " + url);
                }
                if (registryURLs != null && registryURLs.size() > 0
                        && url.getParameter("register", true)) {
                    for (URL registryURL : registryURLs) {
                        url = url.addParameterIfAbsent("dynamic", registryURL.getParameter("dynamic"));
                        URL monitorUrl = loadMonitor(registryURL);
                        if (monitorUrl != null) {
                            url = url.addParameterAndEncoded(Constants.MONITOR_KEY, monitorUrl.toFullString());
                        }
                        if (logger.isInfoEnabled()) {
                            logger.info("Register dubbo service " + interfaceClass.getName() + " url " + url + " to registry " + registryURL);
                        }
                        Invoker<?> invoker = proxyFactory.getInvoker(ref, (Class) interfaceClass, registryURL.addParameterAndEncoded(Constants.EXPORT_KEY, url.toFullString()));

                        Exporter<?> exporter = protocol.export(invoker);
                        exporters.add(exporter);
                    }
                } else {
                    Invoker<?> invoker = proxyFactory.getInvoker(ref, (Class) interfaceClass, url);

                    Exporter<?> exporter = protocol.export(invoker);
                    exporters.add(exporter);
                }
            }
        }
        this.urls.add(url);
    }
```
方法exportLocal(url)其实也


