根据上一篇笔记，已经正常获取sqlSession对象，该篇则开始分析查询的执行源码：
```language
 List<Author> authors = session.selectList("domain.blog.mappers.AuthorMapper.selectAllAuthors");
```
```language
   //DefaultSqlSession类：
   //其中parameter为null，rowBounds值为默认RowBounds.DEFAULT(行边界；可用于分页)
  public List selectList(String statement, Object parameter, RowBounds rowBounds) {
    try {
      MappedStatement ms = configuration.getMappedStatement(statement);
      return executor.query(ms, wrapCollection(parameter), rowBounds, Executor.NO_RESULT_HANDLER);
    } catch (Exception e) {
      throw ExceptionFactory.wrapException("Error querying database.  Cause: " + e, e);
    } finally {
      ErrorContext.instance().reset();
    }
  }
```
### 1. MappedStatement 获取
```language
       	MappedStatement ms = configuration.getMappedStatement(statement);
```
```language
    //Configuration类
    public MappedStatement getMappedStatement(String id) {
    	return mappedStatements.get(id);
    }
```
从如上代码来看，暂时不太确定其中的逻辑，需要具体了解mappedStatements的设置值的逻辑，针对性查看Configuration源码，发现如下示例：
```
protected final Map<String, MappedStatement> mappedStatements = new StrictMap<String, MappedStatement>("Mapped Statements collection");

  public void addMappedStatement(MappedStatement ms) {
    mappedStatements.put(ms.getId(), ms);
  }

```
于是乎通过断点确认，Mybatis在初始化SqlSessionFactory实例时（ sqlMapper = new SqlSessionFactoryBuilder().build(reader);）会解析MapperConfig.xml以及其关联的Mapper文件：
```language
  MapperConfig.xml文件：
  <mappers>
    <mapper resource="org/apache/ibatis/builder/AuthorMapper.xml"/>
    <mapper resource="org/apache/ibatis/builder/BlogMapper.xml"/>
    <mapper resource="org/apache/ibatis/builder/CachedAuthorMapper.xml"/>
    <mapper resource="org/apache/ibatis/builder/PostMapper.xml"/>
    <mapper resource="org/apache/ibatis/builder/NestedBlogMapper.xml"/>
  </mappers>
```
核心代码片段：
```
    //XMLMapperBuilder类解析MapperConfig.xml
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

    private void configurationElement(XNode context) {
    try {
      String namespace = context.getStringAttribute("namespace");
      builderAssistant.setCurrentNamespace(namespace);
      cacheRefElement(context.evalNode("cache-ref"));
      cacheElement(context.evalNode("cache"));
      parameterMapElement(context.evalNodes("/mapper/parameterMap"));
      resultMapElements(context.evalNodes("/mapper/resultMap"));
      sqlElement(context.evalNodes("/mapper/sql"));
      buildStatementFromContext(context.evalNodes("select|insert|update|delete"));
    } catch (Exception e) {
      throw new RuntimeException("Error parsing Mapper XML. Cause: " + e, e);
    }

  }
```
Mapper文件解析及MappedStatement创建示例：
```language
    public void parseStatementNode(XNode context) {
    String id = context.getStringAttribute("id");
    Integer fetchSize = context.getIntAttribute("fetchSize", null);
    Integer timeout = context.getIntAttribute("timeout", null);
    String parameterMap = context.getStringAttribute("parameterMap");
    String parameterType = context.getStringAttribute("parameterType");
    Class parameterTypeClass = resolveClass(parameterType);
    String resultMap = context.getStringAttribute("resultMap");
    String resultType = context.getStringAttribute("resultType");

    Class resultTypeClass = resolveClass(resultType);
    String resultSetType = context.getStringAttribute("resultSetType");
    StatementType statementType = StatementType.valueOf(context.getStringAttribute("statementType", StatementType.PREPARED.toString()));
    ResultSetType resultSetTypeEnum = resolveResultSetType(resultSetType);

    List<SqlNode> contents = parseDynamicTags(context);
    MixedSqlNode rootSqlNode = new MixedSqlNode(contents);
    SqlSource sqlSource = new DynamicSqlSource(configuration, rootSqlNode);
    String nodeName = context.getNode().getNodeName();
    SqlCommandType sqlCommandType = SqlCommandType.valueOf(nodeName.toUpperCase(Locale.ENGLISH));
    boolean isSelect = sqlCommandType == SqlCommandType.SELECT;
    boolean flushCache = context.getBooleanAttribute("flushCache", !isSelect);
    boolean useCache = context.getBooleanAttribute("useCache", isSelect);

    String keyProperty = context.getStringAttribute("keyProperty");
    KeyGenerator keyGenerator;
    String keyStatementId = id + SelectKeyGenerator.SELECT_KEY_SUFFIX;
    keyStatementId = builderAssistant.applyCurrentNamespace(keyStatementId);
    if (configuration.hasKeyGenerator(keyStatementId)) {
      keyGenerator = configuration.getKeyGenerator(keyStatementId);
    } else {
      keyGenerator = context.getBooleanAttribute("useGeneratedKeys",
          configuration.isUseGeneratedKeys() && SqlCommandType.INSERT.equals(sqlCommandType))
          ? new Jdbc3KeyGenerator() : new NoKeyGenerator();
    }

    builderAssistant.addMappedStatement(id, sqlSource, statementType, sqlCommandType,
        fetchSize, timeout, parameterMap, parameterTypeClass, resultMap, resultTypeClass,
        resultSetTypeEnum, flushCache, useCache, keyGenerator, keyProperty);
  }
```
- select * from author where id = #{id}
> id:selectAuthorWithInlineParams ; sqlSource:Configuration实例和rootSqlNode实例（对应mapper的sql节点,其中就有完整的sql）； statementType：PREPARED（STATEMENT, PREPARED, CALLABLE ； sqlCommandType:SELECT(UNKNOWN, INSERT, UPDATE, DELETE, SELECT) ; fetchSize:null ; timeout:null ; parameterMap:null ; parameterTypeClass:class java.lang.Integer ; resultMap:null ; resultTypeClass:class domain.blog.Author ; resultSetTypeEnum:null ; flushCache:false ; useCache:true ; keyGenerator:NoKeyGenerator ; keyProperty:null .
- select * from Blog where id = #{id}
> id:selectBlogWithPostsUsingSubSelect ; sqlSource:Configuration实例和rootSqlNode实例（对应mapper的sql节点,其中就有完整的sql）； statementType：PREPARED（STATEMENT, PREPARED, CALLABLE ； sqlCommandType:SELECT(UNKNOWN, INSERT, UPDATE, DELETE, SELECT) ; fetchSize:null ; timeout:null ; parameterMap:null ; parameterTypeClass:class java.lang.Integer ; resultMap:blogWithPosts ; resultTypeClass:null ; resultSetTypeEnum:null ; flushCache:false ; useCache:true ; keyGenerator:NoKeyGenerator ; keyProperty:null .

### 2. 数据查询
```language
return executor.query(ms, wrapCollection(parameter), rowBounds, Executor.NO_RESULT_HANDLER);
```
根据之前笔记分析，创建SelSession时即已初始化executor实例(实际类型为CachingExecutor(BaseExecutor)），参数：ms为上一步获取的MappedStatement实例；wrapCollection(parameter)最终结果为null（因为当前select没有参数）；rowBounds为默认值（默认查询所有行）；最后一个通过名称先暂时理解为查询无结果处理器。
#### 2.1 executor.query分析（CachingExecutor类）
```language
  public List query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException {
    if (ms != null) {
      Cache cache = ms.getCache();
      if (cache != null) {
        flushCacheIfRequired(ms);
        cache.getReadWriteLock().readLock().lock();
        try {
          if (ms.isUseCache()) {
            CacheKey key = createCacheKey(ms, parameterObject, rowBounds);
            final List cachedList = (List) cache.getObject(key);
            if (cachedList != null) {
              return cachedList;
            } else {
              List list = delegate.query(ms, parameterObject, rowBounds, resultHandler);
              tcm.putObject(cache, key, list);
              return list;
            }
          } else {
            return delegate.query(ms, parameterObject, rowBounds, resultHandler);
          }
        } finally {
          cache.getReadWriteLock().readLock().unlock();
        }
      }
    }
    return delegate.query(ms, parameterObject, rowBounds, resultHandler);
  }
```
#### 2.1.1  cache生成及获取（ms对象不为null）
首先获取MappedStatement的缓存Cache对象，此处既然要使用Cache实例，那就需同步了解下cache实例初始化的逻辑：
```language

  public void parseGeneralStatement(XNode context) {
    // get attributes
    String id = context.getStringAttribute("id");
    String parameterMapName = context.getStringAttribute("parameterMap");
    String parameterClassName = context.getStringAttribute("parameterClass");
    String resultMapName = context.getStringAttribute("resultMap");
    String resultClassName = context.getStringAttribute("resultClass");
    String cacheModelName = context.getStringAttribute("cacheModel");
    String resultSetType = context.getStringAttribute("resultSetType");
    String fetchSize = context.getStringAttribute("fetchSize");
    String timeout = context.getStringAttribute("timeout");
    // 2.x -- String allowRemapping = context.getStringAttribute("remapResults");

    if (context.getStringAttribute("xmlResultName") != null) {
      throw new UnsupportedOperationException("xmlResultName is not supported by iBATIS 3");
    }

    if (mapParser.getConfigParser().isUseStatementNamespaces()) {
      id = mapParser.applyNamespace(id);
    }

    String[] additionalResultMapNames = null;
    if (resultMapName != null) {
      additionalResultMapNames = getAllButFirstToken(resultMapName);
      resultMapName = getFirstToken(resultMapName);
      resultMapName = mapParser.applyNamespace(resultMapName);
      for (int i = 0; i < additionalResultMapNames.length; i++) {
        additionalResultMapNames[i] = mapParser.applyNamespace(additionalResultMapNames[i]);
      }
    }

    String[] additionalResultClassNames = null;
    if (resultClassName != null) {
      additionalResultClassNames = getAllButFirstToken(resultClassName);
      resultClassName = getFirstToken(resultClassName);
    }
    Class[] additionalResultClasses = null;
    if (additionalResultClassNames != null) {
      additionalResultClasses = new Class[additionalResultClassNames.length];
      for (int i = 0; i < additionalResultClassNames.length; i++) {
        additionalResultClasses[i] = resolveClass(additionalResultClassNames[i]);
      }
    }

    Integer timeoutInt = timeout == null ? null : new Integer(timeout);
    Integer fetchSizeInt = fetchSize == null ? null : new Integer(fetchSize);

    // 2.x -- boolean allowRemappingBool = "true".equals(allowRemapping);

    SqlSource sqlSource = new SqlSourceFactory(mapParser).newSqlSourceIntance(mapParser, context);

    String nodeName = context.getNode().getNodeName();
    SqlCommandType sqlCommandType;
    try {
      sqlCommandType = SqlCommandType.valueOf(nodeName.toUpperCase());
    } catch (Exception e) {
      sqlCommandType = SqlCommandType.UNKNOWN; 
    }


    MappedStatement.Builder builder = new MappedStatement.Builder(configuration, id, sqlSource,sqlCommandType);

    builder.useCache(true);//为什么直接设置为true，而不是获取Configuration的cache配置
    if (!"select".equals(context.getNode().getNodeName())) {
      builder.flushCacheRequired(true);
    }

    if (parameterMapName != null) {
      parameterMapName = mapParser.applyNamespace(parameterMapName);
      builder.parameterMap(configuration.getParameterMap(parameterMapName));
    } else if (parameterClassName != null) {
      Class parameterClass = resolveClass(parameterClassName);
      List<ParameterMapping> parameterMappings = new ArrayList<ParameterMapping>();
      if (sqlSource instanceof SimpleSqlSource) {
        parameterMappings = sqlSource.getBoundSql(null).getParameterMappings();
      }
      ParameterMap.Builder parameterMapBuilder = new ParameterMap.Builder(configuration, id + "-ParameterMap", parameterClass, parameterMappings);
      builder.parameterMap(parameterMapBuilder.build());
    }

    List<ResultMap> resultMaps = new ArrayList<ResultMap>();
    if (resultMapName != null) {
      resultMaps.add(configuration.getResultMap(resultMapName));
      if (additionalResultMapNames != null) {
        for (String additionalResultMapName : additionalResultMapNames) {
          resultMaps.add(configuration.getResultMap(additionalResultMapName));
        }
      }
    } else if (resultClassName != null) {
      Class resultClass = resolveClass(resultClassName);
      ResultMap.Builder resultMapBuilder = new ResultMap.Builder(configuration, id + "-ResultMap", resultClass, new ArrayList<ResultMapping>());
      resultMaps.add(resultMapBuilder.build());
      if (additionalResultClasses != null) {
        for (Class additionalResultClass : additionalResultClasses) {
          resultMapBuilder = new ResultMap.Builder(configuration, id + "-ResultMap", additionalResultClass, new ArrayList<ResultMapping>());
          resultMaps.add(resultMapBuilder.build());
        }
      }
    }
    builder.resultMaps(resultMaps);

    builder.fetchSize(fetchSizeInt);

    builder.timeout(timeoutInt);

    if (cacheModelName != null) {
      cacheModelName = mapParser.applyNamespace(cacheModelName);
      Cache cache = configuration.getCache(cacheModelName);
      builder.cache(cache);
    }

    if (resultSetType != null) {
      builder.resultSetType(ResultSetType.valueOf(resultSetType));
    }

    // allowRemappingBool -- silently ignored

    findAndParseSelectKey(id, context);

    configuration.addMappedStatement(builder.build());
  }
```
```language
configuration.setCacheEnabled(booleanValueOf(props.getProperty("cacheEnabled"), true));
```
从如上代码来看，当前版本的Mybatis默认开启二级缓存，即cacheEnabled除非显示指定的false否则只会为true; 然而Builder时却并没有获取 cacheEnabled的值，在整个代码中搜索也未找到；无论是否配置cacheEnabled或者说cacheEnabled无论为true或false，在构建MappedStatement时均默认使用二级缓存。但是实际在调试 ms.getCache()时却发现Cache为null，那是为什么呢？
```language
    //XmlSqlStatementParser类：
    String cacheModelName = context.getStringAttribute("cacheModel");
    if (cacheModelName != null) {
      cacheModelName = mapParser.applyNamespace(cacheModelName);
      Cache cache = configuration.getCache(cacheModelName);
      builder.cache(cache);
    }
```
原来虽然默认开启开级缓存，但如若没有显示配置cacheModelName那么也不会为其创建Cache对象，于是乎在该单元测试的Mapper.xml中添加下cacheModel属性看看是否可初始化缓存对象。然而并没有什么用，因为尝试添加cacheModel时xml直接显示属性非法，故感觉不太对。debug发现并未运行上面的方法，于是乎重新分析。发现如下代码：
```language

  private void configurationElement(XNode context) {
    try {
      String namespace = context.getStringAttribute("namespace");
      builderAssistant.setCurrentNamespace(namespace);
      cacheRefElement(context.evalNode("cache-ref"));
      cacheElement(context.evalNode("cache"));
      parameterMapElement(context.evalNodes("/mapper/parameterMap"));
      resultMapElements(context.evalNodes("/mapper/resultMap"));
      sqlElement(context.evalNodes("/mapper/sql"));
      buildStatementFromContext(context.evalNodes("select|insert|update|delete"));
    } catch (Exception e) {
      throw new RuntimeException("Error parsing Mapper XML. Cause: " + e, e);
    }

  }
```
```language

  public Cache useNewCache(Class typeClass,
                           Class evictionClass,
                           Long flushInterval,
                           Integer size,
                           boolean readWrite,
                           Properties props) {
    typeClass = valueOrDefault(typeClass, PerpetualCache.class);
    evictionClass = valueOrDefault(evictionClass, LruCache.class);
    Cache cache = new CacheBuilder(currentNamespace)
        .implementation(typeClass)
        .addDecorator(evictionClass)
        .clearInterval(flushInterval)
        .size(size)
        .readWrite(readWrite)
        .properties(props)
        .build();
    configuration.addCache(cache);
    currentCache = cache;
    return cache;
  }
```
发现需有对应Mappe.xml中添加 <cache/> 即可。而cache对象是以每个Mapper文件的 Namespace为 id存放在全局公用对象Configuration。
```language
 <mapper namespace="domain.blog.mappers.AuthorMapper">
 	<parameterMap id="selectAuthor"  type="domain.blog.Author">
            <parameter property="id"/>
        </parameterMap>
  
  	<cache/>
```
- 至此重新梳理全局MapperConfig.xml中的 <setting name="cacheEnabled" value="true"/> 与业务Mapper.xml的<cache/>的关系，终于理解了但其实也发现了框架的问题。
- 业务Mapper.xml中的<cache/>标签决定了有解析xml时否根据namespace创建Cache对象；而全局MapperConfig.xml中的 <setting name="cacheEnabled" value="false"/>则决定了创建Executor是否创建CachingExecutor还是SimpleExecutor。
#### 2.1.2  cache处理逻辑
#### 2.1.2.1 cache不为空
1. 获取Cache实例后，判断若cahe不为空则根据flushCache值确认是否清空缓存（当前为false即为不清空），同时获取获取cache的读锁。
2. 根据MappedStatement、parameterObject、rowBounds参数获取CacheKey实例，从caceh根据CacheKey获取cachedList，如若获取到缓存数据则直接返回cachedList（不真实查询数据库）；若cachedList为null,则真实调用delegate.query查询，并调用TransactionalCacheManager的putObject(cache, key, list);更新cache数据，然后返回查询结果list
#### 2.1.2.2 cache为空
则直接查询数据库并返回list（缓存为空，只做数据库查询不作任何处理）
#### 2.1.3  锁释放
finally方法释放锁：cache.getReadWriteLock().readLock().unlock();
#### 2.1.4  ms（MappedStatement对象）为null处理----此处有点质疑
此处有分支，若ms不为null则如上面cache逻辑；而ms为null，并未做大家常做的非空判断，而是调用delegate.query(ms, parameterObject, rowBounds, resultHandler);而通过继续深入了解，发现所有方法均未对ms做非空判断。验证：尝试传入ms为null，则发现为抛出null指针异常

#### 2.2 BaseExecutor.query分析
如上真实查询数据库代码： delegate.query(ms, parameterObject, rowBounds, resultHandler);即对应
BaseExecutor.query方法：
```language
  //BaseExecutor.query方法
  public List query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException {
    ErrorContext.instance().resource(ms.getResource()).activity("executing a query").object(ms.getId());
    if (closed) throw new ExecutorException("Executor was closed.");
    List list;
    try {
      queryStack++;
      CacheKey key = createCacheKey(ms, parameter, rowBounds);
      final List cachedList = (List) localCache.getObject(key);
      if (cachedList != null) {
        list = cachedList;
      } else {
        localCache.putObject(key, EXECUTION_PLACEHOLDER);
        try {
          list = doQuery(ms, parameter, rowBounds, resultHandler);
        } finally {
          localCache.removeObject(key);
        }
        localCache.putObject(key, list);
      }
    } finally {
      queryStack--;
    }
    if (queryStack == 0) {
      for (DeferredLoad deferredLoad : deferredLoads) {
        deferredLoad.load();
      }
    }
    return list;
  }
```
##### 2.2.1 closed用于标识单个executor执行状态，初始为false，而在executor的close方法中会更新为true; queryStack默认为0，稍后分析。
##### 2.2.2 缓存：一开始没想起来此处为何会有缓存？当注意到调用的是localCache是突然明白此处应该是二级缓存，而之前基于ms对象（MappedStatement）则是二级缓存.
```language
      CacheKey key = createCacheKey(ms, parameter, rowBounds);
      final List cachedList = (List) localCache.getObject(key);
    
   //以及createCacheKey源码：
	 public CacheKey createCacheKey(MappedStatement ms, Object parameterObject, RowBounds rowBounds) {
    if (closed) throw new ExecutorException("Executor was closed.");
    BoundSql boundSql = ms.getBoundSql(parameterObject);
    CacheKey cacheKey = new CacheKey();
    cacheKey.update(ms.getId());
    cacheKey.update(rowBounds.getOffset());
    cacheKey.update(rowBounds.getLimit());
    cacheKey.update(boundSql.getSql());
    List<ParameterMapping> parameterMappings = boundSql.getParameterMappings();
    if (parameterMappings.size() > 0 && parameterObject != null) {
      TypeHandlerRegistry typeHandlerRegistry = ms.getConfiguration().getTypeHandlerRegistry();
      if (typeHandlerRegistry.hasTypeHandler(parameterObject.getClass())) {
        cacheKey.update(parameterObject);
      } else {
        MetaObject metaObject = configuration.newMetaObject(parameterObject);
        for (ParameterMapping parameterMapping : parameterMappings) {
          String propertyName = parameterMapping.getProperty();
          if (metaObject.hasGetter(propertyName)) {
            cacheKey.update(metaObject.getValue(propertyName));
          } else if (boundSql.hasAdditionalParameter(propertyName)) {
            cacheKey.update(boundSql.getAdditionalParameter(propertyName));
          }
        }
      }
    }
    return cacheKey;
  }
```
整体来看，是根据当前sql的参数封装cachekey实例，并从当前Exeutor实例变量使用cacheKey从localCache获取缓存信息：
1. 如获得缓存数据，则获取缓存数据，后续不再查询数据库
2. 如未获得缓存数据，则将localCache中对应cacheKey的value值设置为临时占位符（EXECUTION_PLACEHOLDER），然后调用底层查询方法

##### 2.2.3 数据查询（核心查询）
```language
  public List doQuery(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException {
    Statement stmt = null;
    try {
      Configuration configuration = ms.getConfiguration();
      StatementHandler handler = configuration.newStatementHandler(this, ms, parameter, rowBounds, resultHandler);
      stmt = prepareStatement(handler);
      return handler.query(stmt, resultHandler);
    } finally {
      closeStatement(stmt);
    }
  }
```
###### 2.2.3.1 数据查询
1. 获取全局的Configuration
```language
  Configuration configuration = ms.getConfiguration();
```
2. 获取处理器
```language
   StatementHandler handler = configuration.newStatementHandler(this, ms, parameter, rowBounds, resultHandler);
```
```language
  //Configuration类
  public StatementHandler newStatementHandler(Executor executor, MappedStatement mappedStatement, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler) {
    StatementHandler statementHandler = new RoutingStatementHandler(executor, mappedStatement, parameterObject, rowBounds, resultHandler);
    statementHandler = (StatementHandler) interceptorChain.pluginAll(statementHandler);
    return statementHandler;
  }
```
```language

  public RoutingStatementHandler(Executor executor, MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler) {

    switch (ms.getStatementType()) {
      case STATEMENT:
        delegate = new SimpleStatementHandler(executor, ms, parameter, rowBounds, resultHandler);
        break;
      case PREPARED:
        delegate = new PreparedStatementHandler(executor, ms, parameter, rowBounds, resultHandler);
        break;
      case CALLABLE:
        delegate = new CallableStatementHandler(executor, ms, parameter, rowBounds, resultHandler);
        break;
      default:
        throw new ExecutorException("Unknown statement type: " + ms.getStatementType());
    }

  }
```
```language
  protected BaseStatementHandler(Executor executor, MappedStatement mappedStatement, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler) {
    this.configuration = mappedStatement.getConfiguration();
    this.executor = executor;
    this.mappedStatement = mappedStatement;
    this.rowBounds = rowBounds;

    this.typeHandlerRegistry = configuration.getTypeHandlerRegistry();
    this.objectFactory = configuration.getObjectFactory();

    this.boundSql = mappedStatement.getBoundSql(parameterObject);

    this.parameterHandler = configuration.newParameterHandler(mappedStatement, parameterObject, boundSql);
    this.resultSetHandler = configuration.newResultSetHandler(executor, mappedStatement, rowBounds, parameterHandler, resultHandler, boundSql);
  }
```
其中就比较关注的是请求参数、响应结果处理逻辑，即解析全局MpaaerConfig.xml时如若有配置
```language
  全局MpaaerConfig.xml
  <typeHandlers>
    <typeHandler javaType="String" jdbcType="VARCHAR" handler="org.apache.ibatis.builder.ExampleTypeHandler"/>
  </typeHandlers>
```
3. ParametersHandler
```language
//Configuration类：
public ParameterHandler newParameterHandler(MappedStatement mappedStatement, Object parameterObject, BoundSql boundSql) {
    ParameterHandler parameterHandler = new DefaultParameterHandler(mappedStatement, parameterObject, boundSql);
    parameterHandler = (ParameterHandler) interceptorChain.pluginAll(parameterHandler);
    return parameterHandler;
  }
```
源码：
```language
public class DefaultParameterHandler implements ParameterHandler {

  private final TypeHandlerRegistry typeHandlerRegistry;

  private final MappedStatement mappedStatement;
  private final Object parameterObject;
  private BoundSql boundSql;
  private Configuration configuration;


  public DefaultParameterHandler(MappedStatement mappedStatement, Object parameterObject, BoundSql boundSql) {
    this.mappedStatement = mappedStatement;
    this.configuration = mappedStatement.getConfiguration();
    this.typeHandlerRegistry = mappedStatement.getConfiguration().getTypeHandlerRegistry();
    this.parameterObject = parameterObject;
    this.boundSql = boundSql;
  }

  public Object getParameterObject() {
    return parameterObject;
  }

  public void setParameters(PreparedStatement ps)
      throws SQLException {
    ErrorContext.instance().activity("setting parameters").object(mappedStatement.getParameterMap().getId());
    List<ParameterMapping> parameterMappings = boundSql.getParameterMappings();
    if (parameterMappings != null) {
      MetaObject metaObject = parameterObject == null ? null : configuration.newMetaObject(parameterObject);
      for (int i = 0; i < parameterMappings.size(); i++) {
        ParameterMapping parameterMapping = parameterMappings.get(i);
        if (parameterMapping.getMode() != ParameterMode.OUT) {
          Object value;
          String propertyName = parameterMapping.getProperty();
          PropertyTokenizer prop = new PropertyTokenizer(propertyName);
          if (parameterObject == null) {
            value = null;
          } else if (typeHandlerRegistry.hasTypeHandler(parameterObject.getClass())) {
            value = parameterObject;
          } else if (boundSql.hasAdditionalParameter(propertyName)) {
            value = boundSql.getAdditionalParameter(propertyName);
          } else if (propertyName.startsWith(ForEachSqlNode.ITEM_PREFIX)
              && boundSql.hasAdditionalParameter(prop.getName())) {
            value = boundSql.getAdditionalParameter(prop.getName());
            if (value != null) {
              value = configuration.newMetaObject(value).getValue(propertyName.substring(prop.getName().length()));
            }
          } else {
            value = metaObject == null ? null : metaObject.getValue(propertyName);
          }
          TypeHandler typeHandler = parameterMapping.getTypeHandler();
          if (typeHandler == null) {
            throw new ExecutorException("There was no TypeHandler found for parameter " + propertyName + " of statement " + mappedStatement.getId());
          }
          typeHandler.setParameter(ps, i + 1, value, parameterMapping.getJdbcType());
        }
      }
    }
  }

}
```
ResultSetHandler
```language
  public ResultSetHandler newResultSetHandler(Executor executor, MappedStatement mappedStatement, RowBounds rowBounds, ParameterHandler parameterHandler, ResultHandler resultHandler, BoundSql boundSql) {
    ResultSetHandler resultSetHandler = mappedStatement.hasNestedResultMaps() ?
        new NestedResultSetHandler(executor, mappedStatement, parameterHandler, resultHandler, boundSql, rowBounds)
        : new FastResultSetHandler(executor, mappedStatement, parameterHandler, resultHandler, boundSql, rowBounds);
    resultSetHandler = (ResultSetHandler) interceptorChain.pluginAll(resultSetHandler);
    return resultSetHandler;
  }
```



