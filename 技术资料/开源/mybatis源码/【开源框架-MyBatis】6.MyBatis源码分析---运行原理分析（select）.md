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
1. 初始化multipleResults
2. resultMaps对象对应解析Mapper.xml时生成的resultMap列表，当前示例的ResultMap对象对应class值为class domain.blog.Post。（此得注意到resultMaps.get(resultSetCount)，可发现可支持配置多个，如resultMap="account-result, account-result"）
3. stmt.getResultSet()返回ResultSet （而此外即是对应各数据库驱动对应ResultSet的实现类的实例，此处为：org.apache.derby.impl.jdbc.EmbedResultSet40）
4. validateResultMapsCount相对比较简单，则是确认st对象不为空且resultMaps有ResultMap对象可用于处理结果（否则抛异常）
5. 循环获取resultMap，并调用handleResultSet方法
```language
  //FastResultSetHandler类
  protected void handleResultSet(ResultSet rs, ResultMap resultMap, List multipleResults) throws SQLException {
    if (resultHandler == null) {
      DefaultResultHandler defaultResultHandler = new DefaultResultHandler();
      handleRowValues(rs, resultMap, defaultResultHandler, rowBounds);
      multipleResults.add(defaultResultHandler.getResultList());
    } else {
      handleRowValues(rs, resultMap, resultHandler, rowBounds);
    }
  }

    protected void handleRowValues(ResultSet rs, ResultMap resultMap, ResultHandler resultHandler, RowBounds rowBounds) throws SQLException {
    final DefaultResultContext resultContext = new DefaultResultContext();
    skipRows(rs, rowBounds);
    while (shouldProcessMoreRows(rs, resultContext, rowBounds)) {
      final ResultMap discriminatedResultMap = resolveDiscriminatedResultMap(rs, resultMap);
      Object rowValue = getRowValue(rs, discriminatedResultMap, null);
      resultContext.nextResultObject(rowValue);
      resultHandler.handleResult(resultContext);
    }
  }
 
    protected Object getRowValue(ResultSet rs, ResultMap resultMap, CacheKey rowKey) throws SQLException {
    final List<String> mappedColumnNames = new ArrayList<String>();
    final List<String> unmappedColumnNames = new ArrayList<String>();
    final ResultLoaderMap lazyLoader = instantiateResultLoaderMap();
    Object resultObject = createResultObject(rs, resultMap, lazyLoader);
    if (resultObject != null && !typeHandlerRegistry.hasTypeHandler(resultMap.getType())) {
      final MetaObject metaObject = configuration.newMetaObject(resultObject);
      loadMappedAndUnmappedColumnNames(rs, resultMap, mappedColumnNames, unmappedColumnNames);
      boolean foundValues = resultMap.getConstructorResultMappings().size() > 0;
      if (!AutoMappingBehavior.NONE.equals(configuration.getAutoMappingBehavior())) {
        foundValues = applyAutomaticMappings(rs, unmappedColumnNames, metaObject) || foundValues;
      }
      foundValues = applyPropertyMappings(rs, resultMap, mappedColumnNames, metaObject, lazyLoader) || foundValues;
      resultObject = foundValues ? resultObject : null;
      return resultObject;
    }
    return resultObject;
  }
```
- handleResultSet方法：resultHandler为null，即初始化defaultResultHandler 调用handleRowValues方法；
- handleRowValues方法：skipRows主要是用于有指定rowBounds参数时跳至指定行，一般可忽略；shouldProcessMoreRows则会判断rs.next()并移动指针。
- handleResultSet方法：resolveDiscriminatedResultMap用于处理ResultMap标识中的discriminator标签逻辑（discriminator翻译过来为鉴别器；之前的示例有讲：discriminator可指定根据某列的值做逻辑判断，可关联不同的SubMap）获取当行前对应的SubMap。如若ResultMap无discriminator则直接返回ResultMap；若ResultMap有discriminator标签，则根据discriminator从rs获取对应列值并取得SubMap并返回SubMap
- getRowValue方法（getRowValue(rs, discriminatedResultMap, null)如下核心方法）
  1.createResultObject()方法用于创建结果对象，但参数中的ResultMap是上一步获取的discriminator对应的SubMap，即创建的是子ResultMap对应的实例对象（其中会判断ResultMap的type对应的class；如若class在TypeHandler中则根据ResultMap的Type创建对应对象（如Integer、Date，抑或自定义对象）；如若有constructor标签，则调用获取rs对应的值调用有参构造方法实例化对象；如若classg不在TypeHandler也无constructor标签则直接调用objectFactory.create(resultType)创建对象，就对应于resultType="domain.blog.Author"）---此处疑问？ResultMap如果返回的是discriminator标签的SubMap，此处仅是创建子对象；那怎么实例化RsultMap对象的呢？
> 之所以理解错也是对discriminator标签的SubMap与ResultMap的Class没有理解清楚？SubMap对应的Class必须是ReusltMap对应的Class的子类，如Document是Book、Magazine父类 ，只是各子类有添加各自私有属性，那么上面的问题就不难理解，即根据Book或者 Magazine对象的属性与rs匹配后赋值
```language
  <resultMap id="document" class="com.testdomain.Document">
    <result property="id" column="DOCUMENT_ID"/>
    <result property="title" column="DOCUMENT_TITLE"/>
    <result property="type" column="DOCUMENT_TYPE"/>
    <discriminator column="DOCUMENT_TYPE" javaType="string">
      <subMap value="BOOK" resultMap="book"/>
      <subMap value="NEWSPAPER" resultMap="news"/>
    </discriminator>
  </resultMap>

    <resultMap id="book" class="com.testdomain.Book" extends="document">
    <result property="pages" column="DOCUMENT_PAGENUMBER"/>
  </resultMap>

  <resultMap id="news" class="com.testdomain.Magazine" extends="document">
    <result property="city" column="DOCUMENT_CITY"/>
  </resultMap>
```
  2.applyAutomaticMappings(rs, unmappedColumnNames, metaObject)方法用于处理结果集对对象的映射（即类似于：resultType="domain.blog.Author"；其参数为unmappedColumnNames），会根据metaObject（其是结果类的包装类实例，即Author的包装类实例对象）的属性property，从rs中获取值并用对应的TypeHandler处理，最后赋值给metaObject并标记foundValues为true;(之前有分析，通过当前代码也可看出，结果集到对象的映射支持可通过配置关闭) 
  3.applyPropertyMappings(rs, resultMap, mappedColumnNames, metaObject, lazyLoader)可看出参数为mappedColumnNames，即可在ResultMap中映射到的列名；区别同第一个方法在于先根据column获取TypeHandler处理后的value，再根据property赋值给metaObjet对象。

##### 2.2.5 resultSetHandler.handleResultSets方法
###### 2.2.5.1 FastResultSetHandler:handleResultSets
```language
  Mapper.xml文件
  <resultMap id="blogWithPosts" type="Blog">
    <id property="id" column="id"/>
    <result property="title" column="title"/>
    <association property="author" column="author_id"
                 select="selectAuthorWithInlineParams"/>
    <collection property="posts" column="id" select="selectPostsForBlog"/>
  </resultMap>

  <select id="selectBlogWithPostsUsingSubSelect" parameterType="int" resultMap="blogWithPosts">
    select * from Blog where id = #{id}
  </select>

  <select id="selectAuthorWithInlineParams"
          parameterType="int"
          resultType="domain.blog.Author">
    select * from author where id = #{id}
  </select>

  <select id="selectPostsForBlog" parameterType="int" resultType="Post">
    select * from Post where blog_id = #{blog_id}
  </select>
```
- 查询三次
1. select * from Blog where id = ?
2. select * from author where id = ?
3. select * from Post where blog_id = ?
```

DEBUG [main] - ooo Connection Opened
ExamplePlugin intercept:org.apache.ibatis.executor.CachingExecutor:query
ExamplePlugin intercept:org.apache.ibatis.executor.parameter.DefaultParameterHandler:setParameters
ExamplePlugin intercept:org.apache.ibatis.executor.statement.RoutingStatementHandler:query
DEBUG [main] - ==>  Executing: select * from Blog where id = ? 
DEBUG [main] - ==> Parameters: 1(Integer)
ExamplePlugin intercept:org.apache.ibatis.executor.resultset.FastResultSetHandler:handleResultSets
DEBUG [main] - <==    Columns: ID, AUTHOR_ID, TITLE
DEBUG [main] - <==        Row: 1, 101, Jim Business
ExamplePlugin intercept:org.apache.ibatis.executor.parameter.DefaultParameterHandler:setParameters
ExamplePlugin intercept:org.apache.ibatis.executor.statement.RoutingStatementHandler:query
DEBUG [main] - ==>  Executing: select * from author where id = ? 
DEBUG [main] - ==> Parameters: 101(Integer)
ExamplePlugin intercept:org.apache.ibatis.executor.resultset.FastResultSetHandler:handleResultSets
DEBUG [main] - <==    Columns: ID, USERNAME, PASSWORD, EMAIL, BIO, FAVOURITE_SECTION
DEBUG [main] - <==        Row: 101, jim, ********, jim@ibatis.apache.org, , NEWS
ExamplePlugin intercept:org.apache.ibatis.executor.parameter.DefaultParameterHandler:setParameters
ExamplePlugin intercept:org.apache.ibatis.executor.statement.RoutingStatementHandler:query
DEBUG [main] - ==>  Executing: select * from Post where blog_id = ? 
DEBUG [main] - ==> Parameters: 1(Integer)
ExamplePlugin intercept:org.apache.ibatis.executor.resultset.FastResultSetHandler:handleResultSets
DEBUG [main] - <==    Columns: ID, BLOG_ID, AUTHOR_ID, CREATED_ON, SECTION, SUBJECT, BODY, DRAFT
DEBUG [main] - <==        Row: 1, 1, 101, 2007-12-05 00:00:00.0, NEWS, Corn nuts, I think if I never smelled another corn nut it would be too soon..., 1
DEBUG [main] - <==        Row: 2, 1, 101, 2008-01-12 00:00:00.0, VIDEOS, Paul Hogan on Toy Dogs, That's not a dog.  THAT's a dog!, 0
DEBUG [main] - xxx Connection Closed
```
如上在resultMap 中通过select 关联查询仍使用FastResultSetHandler:handleResultSets处理结果集；且后两次关联查询会依赖第一次查询返回结果集中的数据。
###### 2.2.5.1 NestedResultSetHandler:handleResultSets
```language
  Mapper.xml
  <select id="getDocumentsWithAttributes" resultMap="documentWithAttributes">
    select a.*, b.attribute
    from Documents a left join Document_Attributes b
    on a.document_id = b.document_id
    order by a.document_id
  </select>
 <resultMap id="documentWithAttributes" class="com.testdomain.Document" groupBy="id">
    <result property="id" column="DOCUMENT_ID"/>
    <result property="title" column="DOCUMENT_TITLE"/>
    <result property="type" column="DOCUMENT_TYPE"/>
    <result property="attributes" resultMap="Documents.documentAttributes"/>
    <discriminator column="DOCUMENT_TYPE" javaType="string">
      <subMap value="BOOK" resultMap="bookWithAttributes"/>
      <subMap value="NEWSPAPER" resultMap="newsWithAttributes"/>
    </discriminator>
  </resultMap>

  <resultMap id="book" class="com.testdomain.Book" extends="document">
    <result property="pages" column="DOCUMENT_PAGENUMBER"/>
  </resultMap>

  <resultMap id="news" class="com.testdomain.Magazine" extends="document">
    <result property="city" column="DOCUMENT_CITY"/>
  </resultMap>
```
- 查询一次：
1. select a.*, b.attribute     from Documents a left join Document_Attributes b     on a.document_id = b.document_id     order by a.document_id   
```
DEBUG [main] - Checked out connection 22859697 from pool.
DEBUG [main] - ooo Connection Opened
DEBUG [main] - ==>  Executing: select a.*, b.attribute from Documents a left join Document_Attributes b on a.document_id = b.document_id order by a.document_id 
DEBUG [main] - ==> Parameters: 
DEBUG [main] - <==    Columns: DOCUMENT_ID, DOCUMENT_TITLE, DOCUMENT_TYPE, DOCUMENT_PAGENUMBER, DOCUMENT_CITY, ATTRIBUTE
DEBUG [main] - <==        Row: 1, The World of Null-A, BOOK, 55, null, English
DEBUG [main] - <==        Row: 1, The World of Null-A, BOOK, 55, null, Sci-Fi
DEBUG [main] - <==        Row: 2, Le Progres de Lyon, NEWSPAPER, null, Lyon, French
DEBUG [main] - <==        Row: 3, Lord of the Rings, BOOK, 3587, null, Fantasy
DEBUG [main] - <==        Row: 3, Lord of the Rings, BOOK, 3587, null, English
DEBUG [main] - <==        Row: 4, Le Canard enchaine, NEWSPAPER, null, Paris, null
DEBUG [main] - <==        Row: 5, Le Monde, BROADSHEET, null, Paris, null
DEBUG [main] - <==        Row: 6, Foundation, MONOGRAPH, 557, null, null
DEBUG [main] - xxx Connection Closed
DEBUG [main] - Returned connection 22859697 to pool.

```
基于如上分析及如下源码，可初步认为如若resultMap中依赖其他resultMap的id，则会使用NestedResultSetHandler处理结果集；若resultMap中依赖其他的select，则是多次查询仍使用FastResultSetHandler处理结果集。
```
 resultMap.hasNestedResultMaps = resultMap.hasNestedResultMaps || resultMapping.getNestedResultMapId() != null;

  public ResultSetHandler newResultSetHandler(Executor executor, MappedStatement mappedStatement, RowBounds rowBounds, ParameterHandler parameterHandler, ResultHandler resultHandler, BoundSql boundSql) {
    ResultSetHandler resultSetHandler = mappedStatement.hasNestedResultMaps() ?
        new NestedResultSetHandler(executor, mappedStatement, parameterHandler, resultHandler, boundSql, rowBounds)
        : new FastResultSetHandler(executor, mappedStatement, parameterHandler, resultHandler, boundSql, rowBounds);
    resultSetHandler = (ResultSetHandler) interceptorChain.pluginAll(resultSetHandler);
    return resultSetHandler;
  }

```

