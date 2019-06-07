Mapper.xml文件解析重点分析XMLMapperBuilder，在解析MapperConfig.xml时，XMLMapperConfigBuilder类如下代码：
```language
  //XMLMapperConfigBuilder类代码片段
  private Map<String, XNode> sqlFragments = new HashMap<String, XNode>();

  XMLMapperBuilder mapperParser = new XMLMapperBuilder(reader, configuration, resource, sqlFragments);
  mapperParser.parse();
```

### 一.XMLMapperBuilder实例化
reader是对应第一个单独的业务Mapper.xml文件的IO对象
```language
  public XMLMapperBuilder(Reader reader, Configuration configuration, String resource, Map<String, XNode> sqlFragments) {
    super(configuration);
    this.builderAssistant = new MapperBuilderAssistant(configuration, resource);
    this.parser = new XPathParser(reader, true, new XMLMapperEntityResolver(), configuration.getVariables());
    this.sqlFragments = sqlFragments;
    this.resource = resource;
  }
```
```language
  public XPathParser(Reader reader, boolean validation, EntityResolver entityResolver, Properties variables) {
    this.validation = validation;
    this.entityResolver = entityResolver;
    this.variables = variables;
    this.document = createDocument(reader);
    XPathFactory factory = XPathFactory.newInstance();
    this.xpath = factory.newXPath();
  }
```
解析验证xml文件格式并获取document节点对象，初始化XML文件的Parser对象

### 二.parse解析
基于上面分析已实例化XMLMapperBuilder对象mapperParser，之后即调用parse方法
```language
   mapperParser.parse();
```
```language
  //XMLMapperBuilder类
  public void parse() {
    if (!configuration.isResourceLoaded(resource)) {
      configuration.addLoadedResource(resource);
      configurationElement(parser.evalNode("/mapper"));
      bindMapperForNamespace();
    }
  }
```
后续以AuthorMapper.xml文件为例分析解读： parser.evalNode("/mapper")获取AuthorMapper.xml的mapper节点对象.AuthorMapper.xml格式如下（节选，删除部分）
```language
<mapper namespace="domain.blog.mappers.AuthorMapper">

  <parameterMap id="selectAuthor"
                type="domain.blog.Author">
    <parameter property="id"/>
  </parameterMap>

  <resultMap id="selectAuthor" type="domain.blog.Author">
    <id column="id" property="id"/>
    <result property="username" column="username"/>
    <result property="password" column="password"/>
    <result property="email" column="email"/>
    <result property="bio" column="bio"/>
    <result property="favouriteSection" column="favourite_section"/>
  </resultMap>

  <resultMap id="selectImmutableAuthor" type="domain.blog.ImmutableAuthor">
    <constructor>
      <idArg column="id" javaType="_int"/>
      <arg column="username" javaType="string"/>
      <arg column="password" javaType="string"/>
      <arg column="email" javaType="string"/>
      <arg column="bio" javaType="string"/>
      <arg column="favourite_section" javaType="domain.blog.Section"/>
    </constructor>
  </resultMap>

  <select id="selectAllAuthors"
          resultType="domain.blog.Author">
    select * from author
  </select>
  
    <select id="selectAuthor"
          parameterMap="selectAuthor"
          resultMap="selectAuthor">
    select id, username, password, email, bio, favourite_section
    from author where id = ?
  </select>

  <insert id="insertAuthor"
          parameterType="domain.blog.Author">
    insert into Author (id,username,password,email,bio)
    values (#{id},#{username},#{password},#{email},#{bio})
  </insert>

   <update id="updateAuthor"
          parameterType="domain.blog.Author">
    update Author
    set username=#{username,
                 javaType=String},
        password=#{password},
        email=#{email},
        bio=#{bio}
    where id=#{id}
  </update>

</mapper>
```
而解析mapper.xml节点各子节点标签的源码为：
```language
  //XMLMapperBuilder类
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

#### 2.1.namespace解析
获取mapper节点的namespace值并赋值给MapperBuilderAssistant实例对象

#### 2.2.cache-ref解析
cache-ref从名称可理解为缓存引用，而此处同样是取namespace值并赋值给MapperBuilderAssistant实例对象，从此处来看指二级缓存默认是以namespace来划分

#### 2.3.cache解析
```language
  private void cacheElement(XNode context) throws Exception {
    if (context != null) {
      String type = context.getStringAttribute("type", "PERPETUAL");
      Class typeClass = typeAliasRegistry.resolveAlias(type);
      String eviction = context.getStringAttribute("eviction", "LRU");
      Class evictionClass = typeAliasRegistry.resolveAlias(eviction);
      Long flushInterval = context.getLongAttribute("flushInterval");
      Integer size = context.getIntAttribute("size");
      boolean readWrite = !context.getBooleanAttribute("readOnly", false);
      Properties props = context.getChildrenAsProperties();
      builderAssistant.useNewCache(typeClass, evictionClass, flushInterval, size, readWrite, props);
    }
  }
```
从代码来看，<cache> 标签可配置属性为：type(不配置默认PERPETUAL)、eviction（不配置默认LRU)、flushInterval(不配置默认返回nul)、size(不配置默认返回nul)、readOnly(不配置默认false);同时会根据type、eviction匹配对应的class类。稍后则调用MapperBuilderAssistant实例的useNewCache方法初始化缓存对象
```language
  //MapperBuilderAssistant类
  public Cache useNewCache(Class typeClass,
                           Class evictionClass,
                           Long flushInterval,
                           Integer size,
                           boolean readWrite,
                           Properties props) {
    typeClass = valueOrDefault(typeClass, PerpetualCache.class);
    evictionClass = valueOrDefault(evictionClass, LruCache.class);
    Cache cache = new CacheBuilder(currentNamespace)
        .implementation(typeClass)
        .addDecorator(evictionClass)
        .clearInterval(flushInterval)
        .size(size)
        .readWrite(readWrite)
        .properties(props)
        .build();
    configuration.addCache(cache);
    currentCache = cache;
    return cache;
  }
```
useNewCache会完成cache实例化，并将其存入configuration实例；该实例用HashMap存储所有的cache对象，key为cache的ID（即currentNamespace）,value为cache对象。之后将currentCache指向初始化的cache实例并返回

#### 2.4.parameterMap及parameter解析
```language
   parameterMapElement(context.evalNodes("/mapper/parameterMap"));
```
此处
```language
  private void parameterMapElement(List<XNode> list) throws Exception {
    for (XNode parameterMapNode : list) {
      String id = parameterMapNode.getStringAttribute("id");
      String type = parameterMapNode.getStringAttribute("type");
      Class parameterClass = resolveClass(type);
      List<XNode> parameterNodes = parameterMapNode.evalNodes("parameter");
      List<ParameterMapping> parameterMappings = new ArrayList<ParameterMapping>();
      for (XNode parameterNode : parameterNodes) {
        String property = parameterNode.getStringAttribute("property");
        String javaType = parameterNode.getStringAttribute("javaType");
        String jdbcType = parameterNode.getStringAttribute("jdbcType");
        String resultMap = parameterNode.getStringAttribute("resultMap");
        String mode = parameterNode.getStringAttribute("mode");
        String typeHandler = parameterNode.getStringAttribute("typeHandler");
        Integer numericScale = parameterNode.getIntAttribute("numericScale", null);
        ParameterMode modeEnum = resolveParameterMode(mode);
        Class javaTypeClass = resolveClass(javaType);
        JdbcType jdbcTypeEnum = resolveJdbcType(jdbcType);
        Class typeHandlerClass = resolveClass(typeHandler);
        ParameterMapping parameterMapping = builderAssistant.buildParameterMapping(parameterClass, property, javaTypeClass, jdbcTypeEnum, resultMap, modeEnum, typeHandlerClass, numericScale);
        parameterMappings.add(parameterMapping);
      }
      builderAssistant.addParameterMap(id, parameterClass, parameterMappings);
    }
  }
```
##### 2.4.1 parameterMap解析
从源码看，parameterMap仅有两个属性：id、type