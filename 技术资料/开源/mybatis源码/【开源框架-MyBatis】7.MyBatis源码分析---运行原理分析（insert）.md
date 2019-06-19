基于seelctKey的插入源代码分析
BindingTest测试类shouldInsertAuthorWithSelectKey
```language
  @Test
  public void shouldInsertAuthorWithSelectKey() {
    SqlSession session = sqlSessionFactory.openSession();
    try {
      BoundAuthorMapper mapper = session.getMapper(BoundAuthorMapper.class);
      Author author = new Author(-1, "cbegin", "******", "cbegin@nowhere.com", "N/A", Section.NEWS);
      int rows = mapper.insertAuthor(author);
      assertEquals(1, rows);
      session.rollback();
    } finally {
      session.close();
    }
  }
```
```language
  BoundAuthorMapper.xml文件
  <insert id="insertAuthor" parameterType="domain.blog.Author">
    <selectKey keyProperty="id" resultType="int" order="BEFORE">
      select CAST(RANDOM()*1000000 as INTEGER) a from SYSIBM.SYSDUMMY1
    </selectKey>
    insert into Author (id,username,password,email,bio,favourite_section)
    values(
    #{id}, #{username}, #{password}, #{email}, #{bio}, #{favouriteSection:VARCHAR}
    )
  </insert>
```
因源码为测试各种场景，执行多次后发现结果并未如预期先执行selectKey语句之后再根据key完成insert操作；要通过调试当前操作对应的MappedStatement对应的信息确认是否是执行的当前Mapper.xml中对应的配置（因为多场景测试，许多Mapper文件可能会出现对同一张表的insert操作，使得调用的）

