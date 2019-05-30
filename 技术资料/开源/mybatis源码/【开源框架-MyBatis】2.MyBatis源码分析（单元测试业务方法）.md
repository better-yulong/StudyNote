基于笔记"【开源框架-MyBatis】1.MyBatis源码分析（单元测试before初始化）"已经知道了数据库表、数据初始化及基于MapperConfig.xml文件初始化SqlSessionFactory实例；下面就正式进入单元测试业务方法执行分析：
```language
  @Test
  public void shouldSelectAllAuthors() throws Exception {
    SqlSession session = sqlMapper.openSession(TransactionIsolationLevel.SERIALIZABLE);
    try {
      List<Author> authors = session.selectList("domain.blog.mappers.AuthorMapper.selectAllAuthors");
      assertEquals(2, authors.size());
    } finally {
      session.close();
    }
  }
```

#### 一.基于SqlSessionFactory实例sqlMapper获取SqlSession
默认 SqlSession session = sqlMapper.openSession(TransactionIsolationLevel.SERIALIZABLE)；指定事务级别：序列化隔离级别（实际生产关系数据库极少使用该级别）。个人理解是因为全量单元测试时，不同的测试执行互相有影响（比如删表、建表、查询等均依赖特定的测试表、数据），故指定序列化隔离级别。
- DefaultSqlSessionFactory类创建session源码：
```language
  private SqlSession openSessionFromDataSource(ExecutorType execType, TransactionIsolationLevel level, boolean autoCommit) {
    try {
      final Environment environment = configuration.getEnvironment();
      final DataSource dataSource = getDataSourceFromEnvironment(environment);
      TransactionFactory transactionFactory = getTransactionFactoryFromEnvironment(environment);
      Connection connection = dataSource.getConnection();
      if (level != null) {
        connection.setTransactionIsolation(level.getLevel());
      }
      connection = wrapConnection(connection);
      Transaction tx = transactionFactory.newTransaction(connection, autoCommit);
      Executor executor = configuration.newExecutor(tx, execType);
      return new DefaultSqlSession(configuration, executor, autoCommit);
    } catch (SQLException e) {
      throw ExceptionFactory.wrapException("Error opening session.  Cause: " + e, e);
    } finally {
      ErrorContext.instance().reset();
    }
  }
```
```language
public class Environment {
  private String id;
  private TransactionFactory transactionFactory;
  private DataSource dataSource;
}
```
##### 1. 初始化Environment实例，同步获取数据源datasource、事务管理器工厂transactionFactory及
 获取Connection,并设置事务级别；
```language
final Environment environment = configuration.getEnvironment();
      final DataSource dataSource = getDataSourceFromEnvironment(environment);
      TransactionFactory transactionFactory = getTransactionFactoryFromEnvironment(environment);
      Connection connection = dataSource.getConnection();
      if (level != null) {
        connection.setTransactionIsolation(level.getLevel());
      }
```
##### 2. 包装Connection实例
主要时根据对默认的Connection包装，即如若日志级别是debug模式则返回动态代理实现的Connection包装类：针对Connection获取、关闭连接等添加debug级别日志（底层为JDK Proxy动态代理）
##### 3. 获取事务管理器
Transaction tx = transactionFactory.newTransaction(connection, autoCommit);
##### 4. 获取执行器
```language
	Executor executor = configuration.newExecutor(tx, execType);
```
```language
  //Configuration类：
  public Executor newExecutor(Transaction transaction, ExecutorType executorType) {
    executorType = executorType == null ? defaultExecutorType : executorType;
    executorType = executorType == null ? ExecutorType.SIMPLE : executorType;
    Executor executor;
    if (ExecutorType.BATCH == executorType) {
      executor = new BatchExecutor(this,transaction);
    } else if (ExecutorType.REUSE == executorType) {
      executor = new ReuseExecutor(this,transaction);
    } else {
      executor = new SimpleExecutor(this,transaction);
    }
    if (cacheEnabled) {
      executor = new CachingExecutor(executor);
    }
    executor = (Executor) interceptorChain.pluginAll(executor);
    return executor;
  }

```
基于executorType为ExecutorType.SIMPLE，MappingConfigure.xml中配置 <setting name="cacheEnabled" value="true"/>，上面方法返回的是一个CachingExecutor实例（CachingExecutor实例主要是在正常的Executor对数据库的更新操作之前先刷新缓存；查询时会优先根据查询有效缓存数据，源码后续分析）
```language
  <plugins>
    <plugin interceptor="org.apache.ibatis.builder.ExamplePlugin">
      <property name="pluginProperty" value="100"/>
    </plugin>
  </plugins>
```
```language
  //InterceptorChain类：
  public Object pluginAll(Object target) {
    for (Interceptor interceptor : interceptors) {
      target = interceptor.plugin(target); //底层实现仍然是JDK动态代理:Proxy
    }
    return target;
  }
```
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
```language
  // Plugin 类（继承自InvocationHandler
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
```

通过代码码比较好理解，即是针对拦截器Inteceptor列表（而默认MapperConfig.xml仅配置了一个Inteceptor:ExamplePlugin,




