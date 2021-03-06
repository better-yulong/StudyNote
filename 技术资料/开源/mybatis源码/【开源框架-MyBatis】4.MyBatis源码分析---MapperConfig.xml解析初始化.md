结合单元测试初始化方法源码：
```language
  private static SqlSessionFactory sqlMapper;

  @BeforeClass
  public static void setup() throws Exception {
    createBlogDataSource();
    final String resource = "org/apache/ibatis/builder/MapperConfig.xml";
    final Reader reader = Resources.getResourceAsReader(resource);
    sqlMapper = new SqlSessionFactoryBuilder().build(reader);
  }
```
其中createBlogDataSource()用于初始化内存数据库；resour、reader用于获取指向MyBatis全局Config文件的IO对象，核对均是通过new SqlSessionFactoryBuilder().build(reader)创建SqlSessionFactory实例：
```language
  public SqlSessionFactory build(Reader reader, String environment, Properties props) {
    try {
      XMLConfigBuilder parser = new XMLConfigBuilder(reader, environment, props);
      Configuration config = parser.parse();
      return build(config);
    } catch (Exception e) {
      throw ExceptionFactory.wrapException("Error building SqlSession.", e);
    } finally {
      ErrorContext.instance().reset();
      try {
        reader.close();
      } catch (IOException e) {
        // Intentionally ignore.  Prefer previous error.
      }
    }
  }
```
MapperConfig.xml文件解析重点分析XMLConfigBuilder
### 一. 创建解析器对象parse实例
```language
 	XMLConfigBuilder parser = new XMLConfigBuilder(reader, environment, props);
```
```language
  public XMLConfigBuilder(Reader reader, String environment, Properties props) {
    super(new Configuration());
    ErrorContext.instance().resource("SQL Mapper Configuration");
    this.configuration.setVariables(props);
    this.parsed = false;
    this.environment = environment;
    this.parser = new XPathParser(reader, true, new XMLMapperEntityResolver(), props);
  }
```
1. 全局配置Configuration实例初始化：super(new Configuration());
```language
  public Configuration() {
    typeAliasRegistry.registerAlias("JDBC", JdbcTransactionFactory.class.getName());
    typeAliasRegistry.registerAlias("MANAGED", ManagedTransactionFactory.class.getName());
    typeAliasRegistry.registerAlias("JNDI", JndiDataSourceFactory.class.getName());
    typeAliasRegistry.registerAlias("POOLED", PooledDataSourceFactory.class.getName());
    typeAliasRegistry.registerAlias("UNPOOLED", UnpooledDataSourceFactory.class.getName());

    typeAliasRegistry.registerAlias("PERPETUAL", PerpetualCache.class.getName());
    typeAliasRegistry.registerAlias("FIFO", FifoCache.class.getName());
    typeAliasRegistry.registerAlias("LRU", LruCache.class.getName());
    typeAliasRegistry.registerAlias("SOFT", SoftCache.class.getName());
    typeAliasRegistry.registerAlias("WEAK", WeakCache.class.getName());
  }
```
即完成如事务、数据源、缓存策略别名的注册
2. 解析器对象初始化：new XPathParser(reader, true, new XMLMapperEntityResolver(), props);
重点XMLMapperEntityResolver，源码如下用于xml格式及文件头dtd文件路径初始化（获取dtd文件用于验证xml文件格式合法性）
```language
public class XMLMapperEntityResolver implements EntityResolver {

  private static final Map<String, String> doctypeMap = new HashMap<String, String>();

  private static final String IBATIS_CONFIG_DOCTYPE = "-//ibatis.apache.org//DTD Config 3.0//EN".toUpperCase(Locale.ENGLISH);
  private static final String IBATIS_CONFIG_URL = "http://ibatis.apache.org/dtd/ibatis-3-config.dtd".toUpperCase(Locale.ENGLISH);

  private static final String IBATIS_MAPPER_DOCTYPE = "-//ibatis.apache.org//DTD Mapper 3.0//EN".toUpperCase(Locale.ENGLISH);
  private static final String IBATIS_MAPPER_URL = "http://ibatis.apache.org/dtd/ibatis-3-mapper.dtd".toUpperCase(Locale.ENGLISH);

  private static final String MYBATIS_CONFIG_DOCTYPE = "-//mybatis.org//DTD Config 3.0//EN".toUpperCase(Locale.ENGLISH);
  private static final String MYBATIS_CONFIG_URL = "http://mybatis.org/dtd/mybatis-3-config.dtd".toUpperCase(Locale.ENGLISH);

  private static final String MYBATIS_MAPPER_DOCTYPE = "-//mybatis.org//DTD Mapper 3.0//EN".toUpperCase(Locale.ENGLISH);
  private static final String MYBATIS_MAPPER_URL = "http://mybatis.org/dtd/mybatis-3-mapper.dtd".toUpperCase(Locale.ENGLISH);

  private static final String IBATIS_CONFIG_DTD = "org/apache/ibatis/builder/xml/mybatis-3-config.dtd";
  private static final String IBATIS_MAPPER_DTD = "org/apache/ibatis/builder/xml/mybatis-3-mapper.dtd";

  static {
    doctypeMap.put(IBATIS_CONFIG_URL, IBATIS_CONFIG_DTD);
    doctypeMap.put(IBATIS_CONFIG_DOCTYPE, IBATIS_CONFIG_DTD);

    doctypeMap.put(IBATIS_MAPPER_URL, IBATIS_MAPPER_DTD);
    doctypeMap.put(IBATIS_MAPPER_DOCTYPE, IBATIS_MAPPER_DTD);

    doctypeMap.put(MYBATIS_CONFIG_URL, IBATIS_CONFIG_DTD);
    doctypeMap.put(MYBATIS_CONFIG_DOCTYPE, IBATIS_CONFIG_DTD);

    doctypeMap.put(MYBATIS_MAPPER_URL, IBATIS_MAPPER_DTD);
    doctypeMap.put(MYBATIS_MAPPER_DOCTYPE, IBATIS_MAPPER_DTD);
  }
```

### 二. 创建解析器对象parse实例
```language
 	Configuration config = parser.parse();
```
```language
    //XMLConfigBuilder类
    public Configuration parse() {
    if (parsed) {
      throw new BuilderException("Each MapperConfigParser can only be used once.");
    }
    parsed = true;
    parseConfiguration(parser.evalNode("/configuration"));
    return configuration;
  }
```
parser.evalNode("/configuration")返回MapperConfig.xml文件configuration对应的Node对象（其包含所有的子节点信息），后续解析以该对象为root节点解析
```language
  //XMLConfigBuilder类
  private void parseConfiguration(XNode root) {
    try {
      typeAliasesElement(root.evalNode("typeAliases"));
      pluginElement(root.evalNode("plugins"));
      objectFactoryElement(root.evalNode("objectFactory"));
      objectWrapperFactoryElement(root.evalNode("objectWrapperFactory"));
      propertiesElement(root.evalNode("properties"));
      settingsElement(root.evalNode("settings"));
      environmentsElement(root.evalNode("environments"));
      typeHandlerElement(root.evalNode("typeHandlers"));
      mapperElement(root.evalNode("mappers"));
    } catch (Exception e) {
      throw new BuilderException("Error parsing SQL Mapper Configuration. Cause: " + e, e);
    }
  }
```
#### 2.1 typeAliases解析
```language
  <typeAliases>
    <typeAlias alias="Author" type="domain.blog.Author"/>
    <typeAlias alias="Blog" type="domain.blog.Blog"/>
  </typeAliases>  
```
通过root.evalNode("typeAliases") 获取typeAliases节点对象,之后调用typeAliasesElement()方法解析交存储alias信息：底层为Map存储，以alias的String为key，type对应的class为value。alias可如上显示指定，也可不指定则默认会取类名（如上面的Author、Blog）,如对应的Mapper.xml可类似使用：
```language
 <resultMap id="blogWithPosts" type="Blog">
 </resultMap>		
```

#### 2.2 plugins解
```language
<plugins>
    <plugin interceptor="org.apache.ibatis.builder.ExamplePlugin">
      <property name="pluginProperty" value="100"/>
    </plugin>
  </plugins>
```
所有的自定义Plugin均需实现org.apache.ibatis.plugin.Interceptor
```language
@Intercepts({})
public class ExamplePlugin implements Interceptor {

  public Object intercept(Invocation invocation) throws Throwable {
    return invocation.proceed();
  }

  public Object plugin(Object target) {
    return Plugin.wrap(target, this);
  }

  public void setProperties(Properties properties) {

  }

}
```
- 之前有分析过，在创建Executor的实例时，会执行 executor = (Executor) interceptorChain.pluginAll(executor) 代码，用于对原始的executro实例拦截并基于动态代理生成其代理对象。
每个Interceptor的实现类可使用注解@Intercepts({})根据需要配置拦截信息（Signature可支持多个），如：
> @Intercepts({@Signature(type = Map.class, method = "get", args = {Object.class})})
- Plugin类：先解析Interceptor的实现类的Intercepts配置并解析后存储至signatureMap，然后基于
```language
  // public class Plugin implements InvocationHandler 
  public static Object wrap(Object target, Interceptor interceptor) {
    Map<Class, Set<Method>> signatureMap = getSignatureMap(interceptor);
    Class type = target.getClass();
    Class[] interfaces = getAllInterfaces(type, signatureMap);
    if (interfaces.length > 0) {
      return Proxy.newProxyInstance(
          type.getClassLoader(),
          interfaces,
          new Plugin(target, interceptor, signatureMap));
    }
    return target;
  }

  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    try {
      Set<Method> methods = signatureMap.get(method.getDeclaringClass());
      if (methods != null && methods.contains(method)) {
        return interceptor.intercept(new Invocation(target, method, args));
      }
      return method.invoke(target, args);
    } catch (Exception e) {
      throw ExceptionUtil.unwrapThrowable(e);
    }
  }

  private static Map<Class, Set<Method>> getSignatureMap(Interceptor interceptor) {
    Signature[] sigs = interceptor.getClass().getAnnotation(Intercepts.class).value();
    Map<Class, Set<Method>> signatureMap = new HashMap<Class, Set<Method>>();
    for (Signature sig : sigs) {
      Set<Method> methods = signatureMap.get(sig.type());
      if (methods == null) {
        methods = new HashSet<Method>();
        signatureMap.put(sig.type(), methods);
      }
      try {
        Method method = sig.type().getMethod(sig.method(), sig.args());
        methods.add(method);
      } catch (NoSuchMethodException e) {
        throw new PluginException("Could not find method on " + sig.type() + " named " + sig.method() + ". Cause: " + e, e);
      }
    }
    return signatureMap;
  }

```
- 之前了解过动态代理实现的原理，即所有原始executor方法的调用都会调用Plugin的invoke方法，而invode方法则是根据当前方法名匹配该Plugin对象关联的Inteceptor实例的@Intercepts、Signature，来确定是否先行执行Interceptor实例的intercept方法（此处如若配置有多个Interceptor，则会对executor的实例多次动态代理）。在执行完所有Interceptor的intercetp方法之后，才会调用原始executor的对应方法（method.invoke(target, args);）
- 开始针对Signature的配置不太理解，具体type、method、args参数的意义不及明白，实际发现signatureMap的定义为Map<Class, Set<Method>>，即每个class对应其的Method集合，比如如下的配置即可理解为拦截Map的get方法
```language
  {@Signature(type = Map.class, method = "get", args = {Object.class})}
```
而通过实际源码分析，除了Executor实例外，也会对ParameterHandler、ResultSetHandler、StatementHandler的方法调用生成动态代理对象；但同时也有局限性（个人理解，稍后验证：type对应的应该是当前执行的对象的实际类型，而如若Signature配置的是当前拦截对象的父类或接口是无法拦截的）
- 经验证，发现如此理解是基于type的实际类型拦截理解错误，因为漏看Plugin类wrap方法中生成动态对象的判断逻辑： 
```language
  //Plugin类
  public static Object wrap(Object target, Interceptor interceptor) {
    Map<Class, Set<Method>> signatureMap = getSignatureMap(interceptor);
    Class type = target.getClass();
    Class[] interfaces = getAllInterfaces(type, signatureMap);
    if (interfaces.length > 0) {
      return Proxy.newProxyInstance(
          type.getClassLoader(),
          interfaces,
          new Plugin(target, interceptor, signatureMap));
    }
    return target;
  }

  private static Class[] getAllInterfaces(Class type, Map<Class, Set<Method>> signatureMap) {
    Set<Class> interfaces = new HashSet<Class>();
    while (type != null) {
      for (Class c : type.getInterfaces()) {
        if (signatureMap.containsKey(c)) {
          interfaces.add(c);
        }
      }
      type = type.getSuperclass();
    }
    return interfaces.toArray(new Class[interfaces.size()]);
  }
```
重新分析后新理解：
1. 动态代理是基于接口，对其实现类的对象进行动态代理封装；
2. Plugin的war封装会获取当前运行对象的接口列表并与拦截器Interceptor的Signature的type值匹配，只有匹配上才会生成动态代理对象；若未匹配成功则返回原对象（此处正好解决了之后的误解：未注意到这段匹配逻辑时认为所有的对象都会生成动态代理对象，只是在运行时再根据Interceptor判断是否执行默认逻辑还是先执行拦截器逻辑；当时还怀疑此种会导致过多动态代理对象生成影响性能，看来是想多了）
- 验证通过示例
```language

@Intercepts({
    @Signature(type = Executor.class, method = "query", args = {MappedStatement.class, Object.class, RowBounds.class, ResultHandler.class})})
public class ExamplePlugin implements Interceptor {

  public Object intercept(Invocation invocation) throws Throwable {
	System.out.println("ExamplePlugin intercept...........");
    return invocation.proceed();
  }

```
之前分析时发现拦截对应ParameterHandler、ResultSetHandler、StatementHandler均可生效，故做进一步的验证
```language
@Intercepts({
    @Signature(type = Executor.class, method = "query", args = {MappedStatement.class, Object.class, RowBounds.class, ResultHandler.class})
    ,@Signature(type = ParameterHandler.class, method = "setParameters", args = {PreparedStatement.class})
    ,@Signature(type = ResultSetHandler.class, method = "handleResultSets", args = {Statement.class})
    ,@Signature(type = StatementHandler.class, method = "query", args = {Statement.class,ResultHandler.class})
    })
public class ExamplePlugin implements Interceptor { 

  public Object intercept(Invocation invocation) throws Throwable {
	System.out.println("ExamplePlugin intercept:" + invocation.getTarget().getClass().getName() + ":" + invocation.getMethod().getName());
    return invocation.proceed();
  }
   ....
}

//运行结果：
DEBUG [main] - ooo Connection Opened
ExamplePlugin intercept:org.apache.ibatis.executor.CachingExecutor:query
ExamplePlugin intercept:org.apache.ibatis.executor.parameter.DefaultParameterHandler:setParameters
ExamplePlugin intercept:org.apache.ibatis.executor.statement.RoutingStatementHandler:query
DEBUG [main] - ==>  Executing: select id, username, password, email, bio, favourite_section from author where id = ? 
DEBUG [main] - ==> Parameters: 101(Integer)
ExamplePlugin intercept:org.apache.ibatis.executor.resultset.FastResultSetHandler:handleResultSets
DEBUG [main] - <==    Columns: ID, USERNAME, PASSWORD, EMAIL, BIO, FAVOURITE_SECTION
DEBUG [main] - <==        Row: 101, jim, ********, jim@ibatis.apache.org, , NEWS
DEBUG [main] - xxx Connection Closed
```

#### 2.3 objectFactory解析
```language
      objectFactoryElement(root.evalNode("objectFactory"));
```
```language
  <objectFactory type="org.apache.ibatis.builder.ExampleObjectFactory">
    <property name="objectFactoryProperty" value="100"/>
  </objectFactory>
```
所有的自定义ObjectFactory均需继承 org.apache.ibatis.builder.DefaultObjectFactory
```language
  private void objectFactoryElement(XNode context) throws Exception {
    if (context != null) {
      String type = context.getStringAttribute("type");
      Properties properties = context.getChildrenAsProperties();
      ObjectFactory factory = (ObjectFactory) resolveClass(type).newInstance();
      factory.setProperties(properties);
      configuration.setObjectFactory(factory);
    }
  }
```
从代码来看比较简单，即实例化对象工厂实例，设置Properties并赋值给全局的configuration实例;但从代码看怎么感觉只能配置了个ObjectFactory呢？经验证，原来还真是。至于ObjectFactory何时被使用，适用场景后面再理解分析.

#### 2.4 objectWrapperFactory解析
```language
   objectWrapperFactoryElement(root.evalNode("objectWrapperFactory"));
```
```language
  private void objectWrapperFactoryElement(XNode context) throws Exception {
    if (context != null) {
      String type = context.getStringAttribute("type");
      ObjectWrapperFactory factory = (ObjectWrapperFactory) resolveClass(type).newInstance();
      configuration.setObjectWrapperFactory(factory);
    }
  }
```
从代码来看，解析也相对简单，即初始化objectWrapperFactor实例并赋值给全局的configuration实例，从源码来看也只能配置一个。其何时被使用，适用场景后面再理解分析.

#### 2.5 properties解析
```language
	propertiesElement(root.evalNode("properties"));
```
```language
  <properties resource="org/apache/ibatis/builder/mapper.properties">
       <property name="driver" value="org.apache.derby.jdbc.EmbeddedDriver"/>
  </properties>
```
```language
  private void propertiesElement(XNode context) throws Exception {
    if (context != null) {
      Properties defaults = context.getChildrenAsProperties();
      String resource = context.getStringAttribute("resource");
      String url = context.getStringAttribute("url");
      if (resource != null && url != null) {
        throw new BuilderException("The properties element cannot specify both a URL and a resource based property file reference.  Please specify one or the other.");
      }
      if (resource != null) {
        defaults.putAll(Resources.getResourceAsProperties(resource));
      } else if (url != null) {
        defaults.putAll(Resources.getUrlAsProperties(url));
      }
      Properties vars = configuration.getVariables();
      if (vars != null) {
        defaults.putAll(vars);
      }
      parser.setVariables(defaults);
      configuration.setVariables(defaults);
    }
  }
```
```language
  //mapper.properties内容
  url=jdbc:derby:ibderby;create=true
  username=
  password=
```
properties标签只有两个属性:resource、url，且只能二选一. 其实该方法也比较简单，即是把url或resource指向的properties文件和MapperConfig.xml中properties里的properties属性全部解析合并到Properties并设置到parse对象、configuration对象

#### 2.6 settings解析
```language
  settingsElement(root.evalNode("settings"));
```
```language
  <settings>
    <setting name="cacheEnabled" value="true"/>
    <setting name="lazyLoadingEnabled" value="false"/>
    <setting name="multipleResultSetsEnabled" value="true"/>
    <setting name="useColumnLabel" value="true"/>
    <setting name="useGeneratedKeys" value="false"/>
    <setting name="defaultExecutorType" value="SIMPLE"/>
    <setting name="defaultStatementTimeout" value="25000"/>
  </settings>
```
```language
 private void settingsElement(XNode context) throws Exception {
    if (context != null) {
      Properties props = context.getChildrenAsProperties();
      // Check that all settings are known to the configuration class
      for (Map.Entry entry : props.entrySet()) {
        MetaClass metaConfig = MetaClass.forClass(Configuration.class);
        if (!metaConfig.hasSetter((String) entry.getKey())) {
          throw new BuilderException("The setting " + entry.getKey() + " is not known.  Make sure you spelled it correctly (case sensitive).");
        }
      }
      configuration.setAutoMappingBehavior(AutoMappingBehavior.valueOf(stringValueOf(props.getProperty("autoMappingBehavior"), "PARTIAL")));
      configuration.setCacheEnabled(booleanValueOf(props.getProperty("cacheEnabled"), true));
      configuration.setLazyLoadingEnabled(booleanValueOf(props.getProperty("lazyLoadingEnabled"), safeCglibCheck()));
      configuration.setAggressiveLazyLoading(booleanValueOf(props.getProperty("aggressiveLazyLoading"), true));
      configuration.setMultipleResultSetsEnabled(booleanValueOf(props.getProperty("multipleResultSetsEnabled"), true));
      configuration.setUseColumnLabel(booleanValueOf(props.getProperty("useColumnLabel"), true));
      configuration.setUseGeneratedKeys(booleanValueOf(props.getProperty("useGeneratedKeys"), false));
      configuration.setDefaultExecutorType(ExecutorType.valueOf(stringValueOf(props.getProperty("defaultExecutorType"), "SIMPLE")));
      configuration.setDefaultStatementTimeout(integerValueOf(props.getProperty("defaultStatementTimeout"), null));
    }
  }
```
即逐个解析setting配置并设置给configuration对象的对应字段

#### 2.7 environments解析
```language
      environmentsElement(root.evalNode("environments"));
```
```language
  <environments default="development">
    <environment id="development">
      <transactionManager type="JDBC">
        <property name="" value=""/>
      </transactionManager>
      <dataSource type="UNPOOLED">
        <property name="driver" value="${driver}"/>
        <property name="url" value="${url}"/>
        <property name="username" value="${username}"/>
        <property name="password" value="${password}"/>
      </dataSource>
    </environment>
  </environments>
```
```language
  //XMLConfigBuilder类
  private void environmentsElement(XNode context) throws Exception {
    if (context != null) {
      if (environment == null) {
        environment = context.getStringAttribute("default");
      }
      for (XNode child : context.getChildren()) {
        String id = child.getStringAttribute("id");
        if (isSpecifiedEnvironment(id)) {
          TransactionFactory txFactory = transactionManagerElement(child.evalNode("transactionManager"));
          DataSourceFactory dsFactory = dataSourceElement(child.evalNode("dataSource"));
          Environment.Builder environmentBuilder = new Environment.Builder(id, txFactory, dsFactory.getDataSource());
          configuration.setEnvironment(environmentBuilder.build());
        }
      }
    }
  }
```
- 即将environment属性指向default指向的id(development);循环获得environments的所有子environment节点，但仅对匹配default指向的id的environment进行解析并创建txFactory、dsFactory，之后基于Environment.Builder- 及其build方法生成Environment实例并赋值给configuration对象。
- 两点：1.此处可在xml通过{url}表达式直接使用之前解析获得的properties属性;2.environment可配置多个

#### 2.8 mappers解析
```language
   mapperElement(root.evalNode("mappers"));
```
```language
  <mappers>
    <mapper resource="org/apache/ibatis/builder/AuthorMapper.xml"/>
    <mapper resource="org/apache/ibatis/builder/BlogMapper.xml"/>
    <mapper resource="org/apache/ibatis/builder/CachedAuthorMapper.xml"/>
    <mapper resource="org/apache/ibatis/builder/PostMapper.xml"/>
    <mapper resource="org/apache/ibatis/builder/NestedBlogMapper.xml"/>
  </mappers>
```
```language
  private void mapperElement(XNode parent) throws Exception {
    if (parent != null) {
      for (XNode child : parent.getChildren()) {
        String resource = child.getStringAttribute("resource");
        String url = child.getStringAttribute("url");
        Reader reader;
        if (resource != null && url == null) {
          ErrorContext.instance().resource(resource);
          reader = Resources.getResourceAsReader(resource);
          XMLMapperBuilder mapperParser = new XMLMapperBuilder(reader, configuration, resource, sqlFragments);
          mapperParser.parse();
        } else if (url != null && resource == null) {
          ErrorContext.instance().resource(url);
          reader = Resources.getUrlAsReader(url);
          XMLMapperBuilder mapperParser = new XMLMapperBuilder(reader, configuration, url, sqlFragments);
          mapperParser.parse();
        } else {
          throw new BuilderException("A mapper element may only specify a url or resource, but not both.");
        }
      }
    }
  }
```
- mapper.xml文件可使用resource相对于类路径的资源引用；亦可采用资源定位符资源定位符（URL）。（据悉，后续也支持映射器接口实现类的完全限定类名，即Dao名称（class="org.mybatis.builder.AuthorMapper),还可使用诸如<package name="org.mybatis.builder"/>包名批量引用。
- 而Mapper.xml的解析则重点在XMLMapperBuilder 类，该部分下一篇单独分析。