启动实例化，通过如下代码实例化多个数据源并作为dynamicDataSource 注入到Sring容器；同时实例化SqlSessionFactory 时使用包含多个数据源的动态数据源实例，以便于之后运行时getConnection时可动态从多个数据源中动态选择。
```
	 */
	@Bean(name = "dynamicDataSource")
	@Primary
	@DependsOn({ "dataSource1", "dataSource3", "dataSource2" })
	public DynamicDataSource dataSource() {
		DynamicDataSource multipleDataSource = new DynamicDataSource();
		Map<Object, Object> targetDataSources = new HashMap<Object, Object>();
		DataSource dataSource1 = dataSource1();
		DataSource dataSource3 = dataSource3();
		DataSource dataSource2 = dataSource2();
		if (null != dataSource1) {
			targetDataSources.put(ConstantPool.NG_DATASOURCE, dataSource1);
			//multipleDataSource.setDefaultTargetDataSource(dataSource1);
		}
		if (null != dataSource3) {
			targetDataSources.put(ConstantPool.TZ_DATASOURCE, dataSource3);
			//multipleDataSource.setDefaultTargetDataSource(dataSource3);
		}
		if (null != dataSource2) {
			targetDataSources.put(ConstantPool.GH_DATASOURCE, dataSource2);
			//multipleDataSource.setDefaultTargetDataSource(dataSource2);
		}
		multipleDataSource.setTargetDataSources(targetDataSources);
		return multipleDataSource;
	}

	/**
	 * 根据数据源创建SqlSessionFactory
	 */
	@Bean
	public SqlSessionFactory sqlSessionFactory()
			throws Exception {
		SqlSessionFactoryBean fb = new SqlSessionFactoryBean();
		fb.setDataSource(this.dataSource());// 指定数据源(这个必须有，否则报错)
		// 下边两句仅仅用于*.xml文件，如果整个持久层操作不需要使用到xml文件的话（只用注解就可以搞定），则不加
		//fb.setTypeAliasesPackage(typeAliasesPackage);//指定基包
		// String[] mybatisMapperLocations = {"classpath*:/orm/*.xml", "classpath*:/orm/**/*.xml"};
		fb.setMapperLocations(new PathMatchingResourcePatternResolver().getResources(mapperLocations));//xml

		return fb.getObject();
	}


```
即SqlSessionFactory对应的dataSource为DynamicDataSource实例，而无论为何种数据源，最终都是调用dataSource的getConnection方法，那么DynamicDataSource类对应的getConnection则是正好会调用 如下手
运行时，某个mybatis的select查询栈信息:
```
    DynamicDataSource(AbstractRoutingDataSource).determineTargetDataSource() line: 204	--- Spring提供AbstractRoutingDataSource 动态根据determineCurrentLookupKey值获取数据源
	DynamicDataSource.determineCurrentLookupKey() line: 17	 -- 获取自定义数据源key
	DynamicDataSource(AbstractRoutingDataSource).determineTargetDataSource() line: 196	
	DynamicDataSource(AbstractRoutingDataSource).getConnection() line: 164	
	DataSourceUtils.doGetConnection(DataSource) line: 111	
	DataSourceUtils.getConnection(DataSource) line: 77	
	SpringManagedTransaction.openConnection() line: 82	
	SpringManagedTransaction.getConnection() line: 68	
	SimpleExecutor(BaseExecutor).getConnection(Log) line: 336	
	SimpleExecutor.prepareStatement(StatementHandler, Log) line: 84	
	SimpleExecutor.doQuery(MappedStatement, Object, RowBounds, ResultHandler, BoundSql) line: 62	
	SimpleExecutor(BaseExecutor).queryFromDatabase(MappedStatement, Object, RowBounds, ResultHandler, CacheKey, BoundSql) line: 324	
	SimpleExecutor(BaseExecutor).query(MappedStatement, Object, RowBounds, ResultHandler, CacheKey, BoundSql) line: 156	
	CachingExecutor.query(MappedStatement, Object, RowBounds, ResultHandler, CacheKey, BoundSql) line: 109	
	PageInterceptor.intercept(Invocation) line: 141	
	Plugin.invoke(Object, Method, Object[]) line: 61	
	$Proxy208.query(MappedStatement, Object, RowBounds, ResultHandler) line: not available	
	DefaultSqlSession.selectList(String, Object, RowBounds) line: 148	
	DefaultSqlSession.selectList(String, Object) line: 141	
	DefaultSqlSession.selectOne(String, Object) line: 77	
	NativeMethodAccessorImpl.invoke0(Method, Object, Object[]) line: not available [native method]	
	NativeMethodAccessorImpl.invoke(Object, Object[]) line: 62	
	DelegatingMethodAccessorImpl.invoke(Object, Object[]) line: 43	
	Method.invoke(Object, Object...) line: 498	
	SqlSessionTemplate$SqlSessionInterceptor.invoke(Object, Method, Object[]) line: 433	
	$Proxy129.selectOne(String, Object) line: not available	
	SqlSessionTemplate.selectOne(String, Object) line: 166	
	MapperMethod.execute(SqlSession, Object[]) line: 82	
	MapperProxy<T>.invoke(Object, Method, Object[]) line: 59	
	$Proxy133.merchantLogin(String, String) line: not available	
	LoginServiceImpl.merchantLogin(LoginDto) line: 59

```
getConnection时根据各方法动态指定的index下标来动态获取数据源对应的connection
```
public class DynamicDataSource extends AbstractRoutingDataSource {
	
	@Override
	protected Object determineCurrentLookupKey() {
		return "DataSource" + DBHolder.getIndex;
	}
}

Spring提供AbstractRoutingDataSource 用于数据源路由扩展determineTargetDataSource，即根据上面determineCurrentLookupKey返回值从resolvedDataSources 集合中获取对应的数据源。
	protected DataSource determineTargetDataSource() {
		Assert.notNull(this.resolvedDataSources, "DataSource router not initialized");
		Object lookupKey = determineCurrentLookupKey();
		DataSource dataSource = this.resolvedDataSources.get(lookupKey);
		if (dataSource == null && (this.lenientFallback || lookupKey == null)) {
			dataSource = this.resolvedDefaultDataSource;
		}
		if (dataSource == null) {
			throw new IllegalStateException("Cannot determine target DataSource for lookup key [" + lookupKey + "]");
		}
		return dataSource;
	}

```



