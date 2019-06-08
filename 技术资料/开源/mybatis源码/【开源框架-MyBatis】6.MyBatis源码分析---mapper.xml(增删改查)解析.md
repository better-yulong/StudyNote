XMLMapperBuilder类
```language
  buildStatementFromContext(context.evalNodes("select|insert|update|delete"));
```
```language
  private void buildStatementFromContext(List<XNode> list) {
    for (XNode context : list) {
      final XMLStatementBuilder statementParser = new XMLStatementBuilder(configuration, builderAssistant, this);
      statementParser.parseStatementNode(context);
    }
  }
```
从new XMLStatementBuilder可理解为前面的均是configuration、builderAssistant对象的初始化，是为最后解析select|insert|update|delete做准备.
```language
  public XMLStatementBuilder(Configuration configuration, MapperBuilderAssistant builderAssistant, XMLMapperBuilder xmlMapperParser) {
    super(configuration);
    this.builderAssistant = builderAssistant;
    this.xmlMapperParser = xmlMapperParser;
  }
  public void parseStatementNode(XNode context) {
    String id = context.getStringAttribute("id");
    Integer fetchSize = context.getIntAttribute("fetchSize", null);
    Integer timeout = context.getIntAttribute("timeout", null);
    String parameterMap = context.getStringAttribute("parameterMap");
    String parameterType = context.getStringAttribute("parameterType");
    Class parameterTypeClass = resolveClass(parameterType);
    String resultMap = context.getStringAttribute("resultMap");
    String resultType = context.getStringAttribute("resultType");

    Class resultTypeClass = resolveClass(resultType);
    String resultSetType = context.getStringAttribute("resultSetType");
    StatementType statementType = StatementType.valueOf(context.getStringAttribute("statementType", StatementType.PREPARED.toString()));
    ResultSetType resultSetTypeEnum = resolveResultSetType(resultSetType);

    List<SqlNode> contents = parseDynamicTags(context);
    MixedSqlNode rootSqlNode = new MixedSqlNode(contents);
    SqlSource sqlSource = new DynamicSqlSource(configuration, rootSqlNode);
    String nodeName = context.getNode().getNodeName();
    SqlCommandType sqlCommandType = SqlCommandType.valueOf(nodeName.toUpperCase(Locale.ENGLISH));
    boolean isSelect = sqlCommandType == SqlCommandType.SELECT;
    boolean flushCache = context.getBooleanAttribute("flushCache", !isSelect);
    boolean useCache = context.getBooleanAttribute("useCache", isSelect);

    String keyProperty = context.getStringAttribute("keyProperty");
    KeyGenerator keyGenerator;
    String keyStatementId = id + SelectKeyGenerator.SELECT_KEY_SUFFIX;
    keyStatementId = builderAssistant.applyCurrentNamespace(keyStatementId);
    if (configuration.hasKeyGenerator(keyStatementId)) {
      keyGenerator = configuration.getKeyGenerator(keyStatementId);
    } else {
      keyGenerator = context.getBooleanAttribute("useGeneratedKeys",
          configuration.isUseGeneratedKeys() && SqlCommandType.INSERT.equals(sqlCommandType))
          ? new Jdbc3KeyGenerator() : new NoKeyGenerator();
    }

    builderAssistant.addMappedStatement(id, sqlSource, statementType, sqlCommandType,
        fetchSize, timeout, parameterMap, parameterTypeClass, resultMap, resultTypeClass,
        resultSetTypeEnum, flushCache, useCache, keyGenerator, keyProperty);
  }
```
- 首先是各参数id、fetchSize、timeout、parameterMap、parameterType（为获取parameterTypeClass）、resultMap、resultType（为获取resultTypeClass）、resultSetType（为获取resultSetTypeEnum）、statementType、flushCache、useCache、keyProperty的并初始化；同时获取nodeName确认isSelect值。

### 1.1 keyProperty（主键处理）
除了上述大多属性的简单处理，相对复杂的可就是常见的插入时基于数据库序列 、sql函数等先获取主键id再完成insert操作,xml的写法有两种：
```language
  <insert id="insert" parameterType="Person" useGeneratedKeys="true" keyProperty="id">
    	INSERT INTO Person(firstName, lastName)
    	VALUES(#{firstName}, #{lastName})
  </insert>

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
```language
    String keyProperty = context.getStringAttribute("keyProperty");
    KeyGenerator keyGenerator;
    String keyStatementId = id + SelectKeyGenerator.SELECT_KEY_SUFFIX;
    keyStatementId = builderAssistant.applyCurrentNamespace(keyStatementId);
    if (configuration.hasKeyGenerator(keyStatementId)) {
      keyGenerator = configuration.getKeyGenerator(keyStatementId);
    } else {
      keyGenerator = context.getBooleanAttribute("useGeneratedKeys",
          configuration.isUseGeneratedKeys() && SqlCommandType.INSERT.equals(sqlCommandType))
          ? new Jdbc3KeyGenerator() : new NoKeyGenerator();
    }
```
而在分析主键逻辑之前，需先关注 List<SqlNode> contents = parseDynamicTags(context);及 configuration.addKeyGenerator(id, new SelectKeyGenerator(keyStatement, executeBefore));即可理解configuration.getKeyGenerator(keyStatementId).
```language
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

```
```language
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
```
```language
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
其实其最终仍是创建一个新的MappedStatement至configuration，只是id是根据当前namespace、父insert的ide


1. 显示指定useGeneratedKeys为true或者是insert语句且MapperConfig.xml中显示指定useGeneratedKeys为true(默认为false),则即说明可使用key生成器，且默认使用Jdbc3KeyGenerator（其implements KeyGenerator）
2. 对于selectKey方式即返回其初始化时生成的keyGenerator（找了下，暂时没有找到