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

### 
