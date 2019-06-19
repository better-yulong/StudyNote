BindingTest   shouldInsertAuthorWithSelectKey




```language
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


