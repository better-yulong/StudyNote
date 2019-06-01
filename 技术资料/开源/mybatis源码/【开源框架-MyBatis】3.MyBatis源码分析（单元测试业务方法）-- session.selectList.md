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
根据之前笔记分析，

CachingExecutor(executor)
