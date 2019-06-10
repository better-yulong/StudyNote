### 一.运行前转换
#### 1.1 基于Mapper.xml的namespace及id关联 
之前的单元测试方法调用查询源码为：
```language
   //session对应类DefaultSqlSession
  session.selectList("domain.blog.mappers.AuthorMapper.selectAllAuthors");
```
继续深入分析可明白，在解析Mapper.xml文件时，会所有的select、delete、update、insert节点生成对应的MappedStatement对象并存储至configuration对象（其存储为Map结构，key即selectList的参数：由Mapper.xml的namespace值及节点的id拼装；value为MappedStatement对象），而在session的selectList首先则是根据参数去获取MappedStatement实例，然后初始化executro执行后面的逻辑.

#### 1.2 基于Dao文件
```language
  @Test
  public void shouldSelectAuthorsUsingMapperClass() {
    SqlSession session = sqlMapper.openSession();
    try {
      AuthorMapper mapper = session.getMapper(AuthorMapper.class);
      List authors = mapper.selectAllAuthor(101);
      assertEquals(2, authors.size());
    } finally {
      session.close();
    }
  }
```
##### 1.2.1 获取AuthorMapper对象
```language
  //MapperRegistry类
  public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
    if (!knownMappers.contains(type))
      throw new BindingException("Type " + type + " is not known to the MapperRegistry.");
    try {
      return MapperProxy.newMapperProxy(type, sqlSession);
    } catch (Exception e) {
      throw new BindingException("Error getting mapper instance. Cause: " + e, e);
    }
  }
```
session.getMapper最终调用的是MapperRegistry类的getMapper方法，此会先判断type是否可识别（这点在上一篇笔记最后有讲解：在mapper.xml文件解析完成之后，会把对应的namespace对应Dao的class存入knownMappers并解析Dao接口）
```language
  //MapperProxy类
  public static <T> T newMapperProxy(Class<T> mapperInterface, SqlSession sqlSession) {
    ClassLoader classLoader = mapperInterface.getClassLoader();
    Class[] interfaces = new Class[]{mapperInterface};
    MapperProxy proxy = new MapperProxy(sqlSession);
    return (T) Proxy.newProxyInstance(classLoader, interfaces, proxy);
  }

  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    try {
      if (!OBJECT_METHODS.contains(method.getName())) {
        final Class declaringInterface = findDeclaringInterface(proxy, method);
        final MapperMethod mapperMethod = new MapperMethod(declaringInterface, method, sqlSession);
        final Object result = mapperMethod.execute(args);
        if (result == null && method.getReturnType().isPrimitive()) {
          throw new BindingException("Mapper method '" + method.getName() + "' (" + method.getDeclaringClass() + ") attempted to return null from a method with a primitive return type (" + method.getReturnType() + ").");
        }
        return result;
      }
    } catch (SQLException e) {
      e.printStackTrace();
    }
    return null;
  }
```
所以结果可理解为生成代理Proxy0类的实例对象，即Proxy0 etends Proxy implements AuthorMapper；且因Proxy为如下构造函数：
```language
  protected Proxy(InvocationHandler paramInvocationHandler)
  {
    this.h = paramInvocationHandler;
  }
```
即Proxy0类也有同样的构造方法，基于动态代理模式则调用Proxy0类的实例对象的方法均会被重写成调用MapperProxy的invoke方法。
- 总的来说：session.getMapper(AuthorMapper.class)返回是的基于动态代理模式生成的代理对象，运行时实际调用的是其继承自MapperProxy类的invokde方法

##### 1.2.2 AuthorMapper方法调用
根据上面分析，最终执行的是MapperProxy类的invoke方法
```language
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    try {
      if (!OBJECT_METHODS.contains(method.getName())) {
        final Class declaringInterface = findDeclaringInterface(proxy, method);
        final MapperMethod mapperMethod = new MapperMethod(declaringInterface, method, sqlSession);
        final Object result = mapperMethod.execute(args);
        if (result == null && method.getReturnType().isPrimitive()) {
          throw new BindingException("Mapper method '" + method.getName() + "' (" + method.getDeclaringClass() + ") attempted to return null from a method with a primitive return type (" + method.getReturnType() + ").");
        }
        return result;
      }
    } catch (SQLException e) {
      e.printStackTrace();
    }
    return null;
  }
```
##### 1.2.2.1 MapperMethod对象创建及初始化
```language
  public MapperMethod(Class declaringInterface, Method method, SqlSession sqlSession) {
    paramNames = new ArrayList<String>();
    paramPositions = new ArrayList<Integer>();
    this.sqlSession = sqlSession;
    this.method = method;
    this.config = sqlSession.getConfiguration();
    this.hasNamedParameters = false;
    this.declaringInterface = declaringInterface;
    setupFields();
    setupMethodSignature();
    setupCommandType();
    validateStatement();
  }
```
setupFields()方法里即根据Proxy0实例的接口名称（AuthorMapper）及当前执行的方法selectAuthor拼接设置commandName；setupMethodSignature则是判断返回值是否为List及方法参数注解等处理；setupCommandType则是根据commandName调用config.getMappedStatement(commandName)方法获取MappedStatement对象及CommandType（获取结果为SqlCommandType.SELECT），validateStatement方法其实有些多余。
> 此处做了一个小实验，即将Dao中的selectAuthor重命名（使得与Mapper.xml的select的名称不匹配），发现执行到如上方法时会因无法获取到对应的MappedStatement而抛出异常。
##### 1.2.2.2 MapperMethod.execute方法
```language
  public Object execute(Object[] args) throws SQLException {
    Object result;
    if (SqlCommandType.INSERT == type) {
      Object param = getParam(args);
      result = sqlSession.insert(commandName, param);
    } else if (SqlCommandType.UPDATE == type) {
      Object param = getParam(args);
      result = sqlSession.update(commandName, param);
    } else if (SqlCommandType.DELETE == type) {
      Object param = getParam(args);
      result = sqlSession.delete(commandName, param);
    } else if (SqlCommandType.SELECT == type) {
      if (returnsList) {
        result = executeForList(args);
      } else {
        Object param = getParam(args);
        result = sqlSession.selectOne(commandName, param);
      }
    } else {
      throw new BindingException("Unkown execution method for: " + commandName);
    }
    return result;
  }
```
基于如上的动态代理的转换，最终运行sqlSession的select方法。

### 二.sqlSession的select执行分析
#### 2.1 List结果集查询
其实查询单个selectOne方法底层仍是调用selectList，唯一区别是RowBounds是设置的默认对象
```language
  //MapperMethod类
  private Object executeForList(Object[] args) throws SQLException {
    Object result;
    if (rowBoundsIndex != null) {
      Object param = getParam(args);
      RowBounds rowBounds = (RowBounds) args[rowBoundsIndex];
      result = sqlSession.selectList(commandName, param, rowBounds);
    } else {
      Object param = getParam(args);
      result = sqlSession.selectList(commandName, param);
    }
    return result;
  }
```
```language
  //DefaultSqlSession类
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
```language
  //BaseExecutor类
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
BaseExecutor类query方法：
##### 2.1.1 queryStack
queryStack判断当前的SQL执行栈，可能会连续执行多条sql语（之前讲解resultMap时有学习），即同一个线程query方法可能会嵌套执行，即每执行一次+1，而结束一次查询减1。简单的记数器，当结果为0意味着执行结束
##### 2.1.2 createCacheKey（用于一级缓存）
```language
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
- createCacheKey为根据MappedStatement的Id、rowBounds参数、parameter（对应mapper.selectAuthor(101)；此处为普通参数，未使用注解且只有1个参数，故为value为101的Integer对象）、sql、ParameterMappings参数对象（如若parameterObject为HashMap则直接cacheKey.update(parameterObject)；否则则根据parameterMappings从metaObject（包装parameterObject）获取参数值（1、3、5）添加至cacheKey。
- 关于参数param涉及到原参数的特殊处理，针对性分析一下：
```language
  //BoundAuthorMapper类
  List<Post> findThreeSpecificPosts(@Param("one") int one,
                                    RowBounds rowBounds,
                                    @Param("two") int two,
                                    int three);
```
```language
  //BindingTest类
  BoundAuthorMapper mapper = session.getMapper(BoundAuthorMapper.class);
  List<Post> posts = mapper.findThreeSpecificPosts(1, new RowBounds(1, 1), 3, 5);
```
- BoundSql boundSql = ms.getBoundSql(parameterObject）
```language
    //原始SQL	
    select * from post
    where id in (#{one},#{two},#{2})
```
```language
  //DynamicSqlSource类
  public BoundSql getBoundSql(Object parameterObject) {
    DynamicContext context = new DynamicContext(configuration, parameterObject);
    rootSqlNode.apply(context);
    SqlSourceBuilder sqlSourceParser = new SqlSourceBuilder(configuration);
    Class parameterType = parameterObject == null ? Object.class : parameterObject.getClass();
    SqlSource sqlSource = sqlSourceParser.parse(context.getSql(), parameterType);
    BoundSql boundSql = sqlSource.getBoundSql(parameterObject);
    for (Map.Entry<String, Object> entry : context.getBindings().entrySet()) {
      boundSql.setAdditionalParameter(entry.getKey(), entry.getValue());
    }
    return boundSql;
  }
```
ms.getBoundSql最终会调用DynamicSqlSource类的getBoundSql方法；context.getSql()对应Mapper.xml文件中的原始SQL；而sqlSourceParser.parse(context.getSql(), parameterType)处理后sqlSource对象的sql属性为（替换占位符）：
```language
select * from post where id in (?,?,?)
```
且sqlSource的ParameterMappings属性则会存储#{one},#{two},#{2}对应的paramName值。

###### 2.1.2.1 MapperMethod的setupMethodSignature方法
mapper.findThreeSpecificPosts(1, new RowBounds(1, 1), 3, 5)有4个参数，而会被如的ParameterMappings仅3个参数，为何呢？问题MapperMethod的setupMethodSignature方法的处理逻辑
```language
  private void setupMethodSignature() {
    if (List.class.isAssignableFrom(method.getReturnType())) {
      returnsList = true;
    }
    final Class[] argTypes = method.getParameterTypes();
    for (int i = 0; i < argTypes.length; i++) {
      if (RowBounds.class.isAssignableFrom(argTypes[i])) {
        rowBoundsIndex = i;
      } else {
        String paramName = String.valueOf(paramPositions.size());
        paramName = getParamNameFromAnnotation(i, paramName);
        paramNames.add(paramName);
        paramPositions.add(i);
      }
    }
  }
  
    private Object getParam(Object[] args) {
    final int paramCount = paramPositions.size();
    if (args == null || paramCount == 0) {
      return null;
    } else if (!hasNamedParameters && paramCount == 1) {
      return args[paramPositions.get(0)];
    } else {
      Map param = new HashMap();
      for (int i = 0; i < paramCount; i++) {
        param.put(paramNames.get(i), args[paramPositions.get(i)]);
      }
      return param;
    }
  }
```
> args为findThreeSpecificPosts参数数组
1. 运行结果：paramNames（List）：[one, two, 2]；paramPositions（List）：[0, 2, 3];rowBoundsIndex=1。其实此处的很简单：1.判断出rowBoundsIndex在参数中的下标；2.解析其他非rowBoundsIndex参数，记录其他参数的paramName(如果有@Param注解则即为注解指定的参数名称；否则就简单粗暴的用下标作为paraName.）
2. 之后的真实查询会基于如上getParam方法根据paramNames、paramPositions转换成param对象（HashMap：{2=5, two=3, one=1}）（只参数只有一个则直接使用普通对象如Integer返回，而不会使用Map）
##### 2.1.3 BaseExecutor的List query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler) 方法
ms为MappedSatement对象；parameter即为非RowBounds对应的Map；RowBounds即为rowBounds对象；resultHandler即为结果ResultHandler对象（默认为Executor.NO_RESULT_HANDLER）

#### 2.2 SimpleExecutor的doQuery方法
```language
  //SimpleExecutor类
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
##### 2.2.1 ms.getConfiguration()
获取当前MappedStatement对象的configuration对象
##### 2.2.2 StatementHandler封装（RoutingStatementHandler实际是该实现类）
之前有分析过，即根据interceptor链及各拦截器配置基于动态代理Proxy.newProxyInstance()生成 resultHandler的代理对象
##### 2.2.3 stmt = prepareStatement(handler);
```language
  //SimpleExecutor类
  private Statement prepareStatement(StatementHandler handler) throws SQLException {
    Statement stmt;
    Connection connection = transaction.getConnection();
    stmt = handler.prepare(connection);
    handler.parameterize(stmt);
    return stmt;
  }
```
```language
  //BaseStatementHandler类
  public Statement prepare(Connection connection)
      throws SQLException {
    ErrorContext.instance().sql(boundSql.getSql());
    Statement statement = null;
    try {
      statement = instantiateStatement(connection);
      setStatementTimeout(statement);
      setFetchSize(statement);
      return statement;
    } catch (SQLException e) {
      closeStatement(statement);
      throw e;
    } catch (Exception e) {
      closeStatement(statement);
      throw new ExecutorException("Error preparing statement.  Cause: " + e, e);
    }
  }

```
- 初始化准备工作：1.设置connection对象prepareStatement对应的sql（如：select * from post where id in (?,?,?)）；2.设置statement的读超时时间（默认为0）；3.设置statment的fetchSize（默认为0）
-  handler.parameterize(stmt)方法则主要针对于sql中有selectKey的场景（insert偏多），会在该方法里面根据keyGenerator完成key值对应的sql的执行及参数绑定，下一篇另行分析。select是该方法可先忽略。

##### 2.2.4 handler.query(stmt, resultHandler)
handler是RoutingStatementHandler的实例，但query方法最终调用的是PreparedStatementHandler的query方法
```language
  public List query(Statement statement, ResultHandler resultHandler)
      throws SQLException {
    PreparedStatement ps = (PreparedStatement) statement;
    ps.execute();
    return resultSetHandler.handleResultSets(ps);
  }
```
###### 2.2.4.1 ps.execute方法执行
其实该方法底层执行依赖数据库恭驱动，即调用数据库驱动对应的java.sql.PreparedStatement的实现类的execute方法。即通过method.invoke(statement, params)最终调用EmbedPreparedStatement动态代理类的executeStatement方法，并将结果等信息赋值给ps对象。具体是哪些信息呢？可通过resultSetHandler.handleResultSets来反向分析
###### 2.2.4.2 resultSetHandler.handleResultSets方法
resultSetHandler为FastResultSetHandler的实现：
```language
  public List handleResultSets(Statement stmt) throws SQLException {
    final List multipleResults = new ArrayList();
    final List<ResultMap> resultMaps = mappedStatement.getResultMaps();
    int resultMapCount = resultMaps.size();
    int resultSetCount = 0;
    ResultSet rs = stmt.getResultSet();
    validateResultMapsCount(rs,resultMapCount);
    while (rs != null && resultMapCount > resultSetCount) {
      final ResultMap resultMap = resultMaps.get(resultSetCount);
      handleResultSet(rs, resultMap, multipleResults);
      rs = getNextResultSet(stmt);
      cleanUpAfterHandlingResultSet();
      resultSetCount++;
    }
    return collapseSingleResultList(multipleResults);
  }
```
resultMaps对象对应解析Mapper.xml时生成的resultMap列表，当前示例的ResultMap对象对应class值为class domain.blog.Post

