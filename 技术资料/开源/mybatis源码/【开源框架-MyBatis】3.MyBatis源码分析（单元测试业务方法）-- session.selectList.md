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
#### 2.1.1  cache处理
首先获取MappedStatement的缓存Cache对象，若不为空则根据flushCache值确认是否清空缓存（当前为false即为不清空），此处既然要使用Cache实例，那就需同步了解下cache实例初始化的逻辑：
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
原来虽然默认开启开级缓存，但如若没有显示配置cacheModelName

