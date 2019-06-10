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
##### 2.1.1 queryStack判断当前的SQL执行栈，可能会连续执行多条sql语（之前讲解resultMap时有学习），即同一个线程query方法可能会嵌套执行，即每执行一次+1，而结束一次查询减1。简单的记数器，当结果为0意味着执行结束
##### 2.1.2 createCacheKey为根据MappedStatement的Id、rowBounds参数、parameter（对应mapper.selectAuthor(101)；此处为普通参数，未使用注解且只有1个参数，故为value为101的Integer对象）、sql、ParameterMappings参数对象（基于parameterMap的parameterMapping所有或者直接parameterType的parameterObject）。此处可能涉及到参数的特殊处理，针对性分析一下：
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
###### 2.1.2.1 MapperMethod的setupMethodSignature方法
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
```
