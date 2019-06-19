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
因源码为测试各种场景，执行多次后发现结果并未如预期先执行selectKey语句之后再根据key完成insert操作；要通过调试当前操作对应的MappedStatement对应的信息确认是否是执行的当前Mapper.xml中对应的配置（因为多场景测试，许多Mapper文件可能会出现对同一张表的insert操作，使得调用到其他的方法）
### 一.初始化分析
Mapper.xml解析生成MappedStatement时（即XMLStatementBuilder类的parseStatementNode(XNode context)方法该行：List<SqlNode> contents = parseDynamicTags(context);
```language
XMLStatementBuilder类
  private Map<String, NodeHandler> nodeHandlers = new HashMap<String, NodeHandler>() {
    {
      put("include", new IncludeNodeHandler());
      put("trim", new TrimHandler());
      put("where", new WhereHandler());
      put("set", new SetHandler());
      put("foreach", new ForEachHandler());
      put("if", new IfHandler());
      put("choose", new ChooseHandler());
      put("when", new IfHandler());
      put("otherwise", new OtherwiseHandler());
      put("selectKey", new SelectKeyHandler());
    }
  };

    private List<SqlNode> parseDynamicTags(XNode node) {
    List<SqlNode> contents = new ArrayList<SqlNode>();
    NodeList children = node.getNode().getChildNodes();
    for (int i = 0; i < children.getLength(); i++) {
      XNode child = node.newXNode(children.item(i));
      String nodeName = child.getNode().getNodeName();
      if (child.getNode().getNodeType() == Node.CDATA_SECTION_NODE
          || child.getNode().getNodeType() == Node.TEXT_NODE) {
        String data = child.getStringBody("");
        contents.add(new TextSqlNode(data));
      } else {
        NodeHandler handler = nodeHandlers.get(nodeName);
        if (handler == null) {
          throw new BuilderException("Unknown element <" + nodeName + "> in SQL statement.");
        }
        handler.handleNode(child, contents);

      }
    }
    return contents;
  }
  
  private class SelectKeyHandler implements NodeHandler {
    public void handleNode(XNode nodeToHandle, List<SqlNode> targetContents) {
      XNode parent = nodeToHandle.getParent();
      String id = parent.getStringAttribute("id") + SelectKeyGenerator.SELECT_KEY_SUFFIX;
      String resultType = nodeToHandle.getStringAttribute("resultType");
      Class resultTypeClass = resolveClass(resultType);
      StatementType statementType = StatementType.valueOf(nodeToHandle.getStringAttribute("statementType", StatementType.PREPARED.toString()));
      String keyProperty = nodeToHandle.getStringAttribute("keyProperty");
      String parameterType = parent.getStringAttribute("parameterType");
      boolean executeBefore = "BEFORE".equals(nodeToHandle.getStringAttribute("order", "AFTER"));
      Class parameterTypeClass = resolveClass(parameterType);

      //defaults
      boolean useCache = false;
      KeyGenerator keyGenerator = new NoKeyGenerator();
      Integer fetchSize = null;
      Integer timeout = null;
      boolean flushCache = false;
      String parameterMap = null;
      String resultMap = null;
      ResultSetType resultSetTypeEnum = null;

      List<SqlNode> contents = parseDynamicTags(nodeToHandle);
      MixedSqlNode rootSqlNode = new MixedSqlNode(contents);
      SqlSource sqlSource = new DynamicSqlSource(configuration, rootSqlNode);
      SqlCommandType sqlCommandType = SqlCommandType.SELECT;

      builderAssistant.addMappedStatement(id, sqlSource, statementType, sqlCommandType,
          fetchSize, timeout, parameterMap, parameterTypeClass, resultMap, resultTypeClass,
          resultSetTypeEnum, flushCache, useCache, keyGenerator, keyProperty);

      id = builderAssistant.applyCurrentNamespace(id);

      MappedStatement keyStatement = configuration.getMappedStatement(id);
      configuration.addKeyGenerator(id, new SelectKeyGenerator(keyStatement, executeBefore));
    }
  }
```





