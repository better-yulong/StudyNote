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
cache-ref从名称可理解为缓存