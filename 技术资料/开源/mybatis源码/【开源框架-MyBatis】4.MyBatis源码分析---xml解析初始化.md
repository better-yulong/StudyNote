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
重点XMLMapperEntityResolver，源码如下用于xml格式及文件头dtd文件路径初始化
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
通过root.evalNode("typeAliases") 获取typeAliases节点对象

