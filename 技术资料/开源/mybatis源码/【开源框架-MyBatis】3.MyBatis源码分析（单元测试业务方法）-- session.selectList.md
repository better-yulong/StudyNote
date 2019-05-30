根据上一篇笔记，已经正常获取sqlSession对象，该篇则开始分析查询的执行源码：
```language
 List<Author> authors = session.selectList("domain.blog.mappers.AuthorMapper.selectAllAuthors");
```
```language
   //其中parameter为null，rowBounds值为默认
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

