### 一.背景
之前虽然会频繁使用Junit进行单元测试，但从未较为深入的理解过，而在分析Mybatis源码时，发现同一单元测试方法多次执行时，上次执行的部分Derby缓存数据后续执行仍然可用。原本单纯的认为每次Junit执行会开启一个新的JVM进程，但验证发现并非如此；故此处针对性理解下。

### 二.分析
#### 1. 调试代码，查看进程信息：
```language
 @Test
  public void shouldSelectAllAuthors() throws Exception {
    SqlSession session = sqlMapper.openSession(TransactionIsolationLevel.SERIALIZABLE);
    try {
      List<Author> authors = session.selectList("domain.blog.mappers.AuthorMapper.selectAllAuthors");
      assertEquals(2, authors.size()); //断点
    } finally {
      session.close();
    }
  }
```
在如上assertEquals行添加断点，以Debugg方式运行Junit测试，运行至断点处后，执行jps -lvm:

