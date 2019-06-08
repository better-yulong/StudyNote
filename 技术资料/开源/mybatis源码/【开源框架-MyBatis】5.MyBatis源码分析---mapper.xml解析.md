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
此处/mapper/parameterMap解析返回结果为List<XNode> ，即说明可有多个parameterMap，然后遍历解析 
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
从源码看，parameterMap仅有两个属性：id、type.其中id为字符串，而type则需可解析成class类（如若type无法）。细看Class parameterClass = resolveClass(type) 会发现type接受的值为两种：1.别名alais，此时会从MapperConfig.xml的typeAliases显示配置及自动初始化的基本类型、数组及集合类在TypeAliasRegistry去匹配获取class；2.class完整名称，会直接基于Class.forName来获得class
##### 2.4.2 parameter解析
一个parameterMap节点可包含多个parameter节点，其存储于List<ParameterMapping> parameterMappings . 可发现parameter标签可配置属性：property、javaType、jdbcType、resultMap、mode、typeHandler、numericScale.
1. property为必有属性
2. javaType 只允许两种情况：1.不配置javaType即resolveClass返回null；2.若有值则必须可从TypeAliasRegistry中匹配出javaTypeClass或为完整类名且可通过Resources.classForName获取class
3. jdbcType只允许两种情况：1.不配置jdbcType即resolveJdbcType返回null；2.若有值则必须可从JdbcType枚举中匹配出JdbcType类型
4. resultMap获取值，稍后使用
5. mode由resolveParameterMode解析只允许两种情况：1.不配置mode即mode返回null；2.若有值则必须为ParameterMode枚举的三个之一：IN, OUT, INOUT
6. typeHandler只允许两种情况：1.不配置typeHandler即resolveClass返回null；2.若有值则必须可从TypeAliasRegistry中匹配出typeHandler对应的class或为完整类名且可通过Resources.classForName获取class
7. numericScale获取值，未配置则为null

- 获取参数并简单初始化后，会调用
```language
     ParameterMapping parameterMapping = builderAssistant.buildParameterMapping(parameterClass, property, javaTypeClass, jdbcTypeEnum, resultMap, modeEnum, typeHandlerClass, numericScale);

```
```language
  //MapperBuilderAssistant类
  public ParameterMapping buildParameterMapping(
      Class parameterType,
      String property,
      Class javaType,
      JdbcType jdbcType,
      String resultMap,
      ParameterMode parameterMode,
      Class typeHandler,
      Integer numericScale) {
    resultMap = applyCurrentNamespace(resultMap);

    // Class parameterType = parameterMapBuilder.type();
    Class javaTypeClass = resolveParameterJavaType(parameterType, property, javaType, jdbcType);
    TypeHandler typeHandlerInstance = (TypeHandler) resolveInstance(typeHandler);

    ParameterMapping.Builder builder = new ParameterMapping.Builder(configuration, property, javaTypeClass);
    builder.jdbcType(jdbcType);
    builder.resultMapId(resultMap);
    builder.mode(parameterMode);
    builder.numericScale(numericScale);
    builder.typeHandler(typeHandlerInstance);
    return builder.build();
  }
```
resultMap名称通过当parameter标签的resultMap的值基于applyCurrentNamespace生成：null或原始值或namespace拼接原始值
```language
  public String applyCurrentNamespace(String base) {
    if (base == null) return null;
    if (base.contains(".")) return base;
    return currentNamespace + "." + base;
  }

```
javaTypeClass值：javaType不为空则直接返回。若为空,jdbcType为CURSOR时其为ResultSet.class，否则从parameterMap中的type对应的类（resultType传入的为parameterMap中的type对应的类）从判断其对应的get方法的返回值。如若最后仍无法匹配到，则初始化为Object.class
```language
  private Class resolveParameterJavaType(Class resultType, String property, Class javaType, JdbcType jdbcType) {
    if (javaType == null) {
      if (JdbcType.CURSOR.equals(jdbcType)) {
        javaType = java.sql.ResultSet.class;
      } else {
        MetaClass metaResultType = MetaClass.forClass(resultType);
        javaType = metaResultType.getGetterType(property);
      }
    }
    if (javaType == null) {
      javaType = Object.class;
    }
    return javaType;
  }
```
typeHandler若为null则直接返回，否则基于typeHandler调用期newInstance方法.注意此处有强制类型转换为TypeHandler，即说明必须实现TypeHandler接口，但其要在MapperConfig.xml配置也可不配置而在AuthorMapper.xml中用完整类名配置
```language
    TypeHandler typeHandlerInstance = (TypeHandler) resolveInstance(typeHandler);
```

```language
    public ParameterMapping build() {
      resolveTypeHandler();
      return parameterMapping;
    }

    private void resolveTypeHandler() {
      if (parameterMapping.typeHandler == null) {
        if (parameterMapping.javaType != null) {
          Configuration configuration = parameterMapping.configuration;
          TypeHandlerRegistry typeHandlerRegistry = configuration.getTypeHandlerRegistry();
          parameterMapping.typeHandler = typeHandlerRegistry.getTypeHandler(parameterMapping.javaType, parameterMapping.jdbcType);
        }
      }
    }
```
最后基于ParameterMapping.Builder实例并调用build方法
```language
    //ParameterMapping类
    public ParameterMapping build() {
      resolveTypeHandler();
      return parameterMapping;
    }

    private void resolveTypeHandler() {
      if (parameterMapping.typeHandler == null) {
        if (parameterMapping.javaType != null) {
          Configuration configuration = parameterMapping.configuration;
          TypeHandlerRegistry typeHandlerRegistry = configuration.getTypeHandlerRegistry();
          parameterMapping.typeHandler = typeHandlerRegistry.getTypeHandler(parameterMapping.javaType, parameterMapping.jdbcType);
        }
      }
    }
```
```language
  //TypeHandlerRegistry类
  public TypeHandler getTypeHandler(Class type, JdbcType jdbcType) {
    Map jdbcHandlerMap = TYPE_HANDLER_MAP.get(type);
    TypeHandler handler = null;
    if (jdbcHandlerMap != null) {
      handler = (TypeHandler) jdbcHandlerMap.get(jdbcType);
      if (handler == null) {
        handler = (TypeHandler) jdbcHandlerMap.get(null);
      }
    }
    if (handler == null && type != null && Enum.class.isAssignableFrom(type)) {
      handler = new EnumTypeHandler(type);
    }
    return handler;
  }
 
  private final Map<JdbcType, TypeHandler> JDBC_TYPE_HANDLER_MAP = new HashMap<JdbcType, TypeHandler>();
  private final Map<Class, Map<JdbcType, TypeHandler>> TYPE_HANDLER_MAP = new HashMap<Class, Map<JdbcType, TypeHandler>>();
```
从如上源码来看，即是尝试根据javaType、jdbcType获取typeHanlder

#### 2.5.resultMap解析
```language
    resultMapElements(context.evalNodes("/mapper/resultMap"));
```
```language
  <resultMap id="selectAuthor" type="domain.blog.Author">
    <id column="id" property="id"/>
    <result property="username" column="username"/>
    <result property="password" column="password"/>
    <result property="email" column="email"/>
    <result property="bio" column="bio"/>
    <result property="favouriteSection" column="favourite_section"/>
  </resultMap>
```
同样中存在多个resultMap标签，遍历解析
```language
  private ResultMap resultMapElement(XNode resultMapNode) throws Exception {
    return resultMapElement(resultMapNode, Collections.EMPTY_LIST);
  }

  private ResultMap resultMapElement(XNode resultMapNode, List<ResultMapping> additionalResultMappings) throws Exception {
    ErrorContext.instance().activity("processing " + resultMapNode.getValueBasedIdentifier());
    String id = resultMapNode.getStringAttribute("id",
        resultMapNode.getValueBasedIdentifier());
    String type = resultMapNode.getStringAttribute("type",
        resultMapNode.getStringAttribute("ofType",
            resultMapNode.getStringAttribute("resultType",
                resultMapNode.getStringAttribute("javaType"))));
    String extend = resultMapNode.getStringAttribute("extends");
    Class typeClass = resolveClass(type);
    Discriminator discriminator = null;
    List<ResultMapping> resultMappings = new ArrayList<ResultMapping>();
    resultMappings.addAll(additionalResultMappings);
    List<XNode> resultChildren = resultMapNode.getChildren();
    for (XNode resultChild : resultChildren) {
      if ("constructor".equals(resultChild.getName())) {
        processConstructorElement(resultChild, typeClass, resultMappings);
      } else if ("discriminator".equals(resultChild.getName())) {
        discriminator = processDiscriminatorElement(resultChild, typeClass, resultMappings);
      } else {
        ArrayList<ResultFlag> flags = new ArrayList<ResultFlag>();
        if ("id".equals(resultChild.getName())) {
          flags.add(ResultFlag.ID);
        }
        resultMappings.add(buildResultMappingFromContext(resultChild, typeClass, flags));
      }
    }
    return builderAssistant.addResultMap(id, typeClass, extend, discriminator, resultMappings);
  }
```
1. id获取
```language
   String id = resultMapNode.getStringAttribute("id",
        resultMapNode.getValueBasedIdentifier());
```
即是获取resultMap的id属性，若没有则根据当前resultMap的其他标签或父节点等生成默认值（个人感觉这块儿代码逻辑不合理，无论id是否有值均会运行resultMapNode.getValueBasedIdentifier()，而基本使用时都会配置id值）
2. type获取
```language
 String type = resultMapNode.getStringAttribute("type",
        resultMapNode.getStringAttribute("ofType",
            resultMapNode.getStringAttribute("resultType",
                resultMapNode.getStringAttribute("javaType"))));
```
即是获取resultMap的type属性，若没有则根据当前resultMap的其他标签或父节点等生成默认值（这块儿逻辑和id获取一样，会导致过多多余的getStringAttribute调用）
3. extends获取
直接获取extends标签值
4. type解析获取class
获取type对应的class：null、基本类型则从TypeHanlderRegister获取、抑或通过forName获取

##### 2.5.1 resultMap/result解析
从源码看，resultMap支持三种子节点：constructor、discriminator、id及默认result
###### 2.5.1.1 constructor方式处理
```language
   processConstructorElement(resultChild, typeClass, resultMappings);
```
```language
  private void processConstructorElement(XNode resultChild, Class resultType, List<ResultMapping> resultMappings) throws Exception {
    List<XNode> argChildren = resultChild.getChildren();
    for (XNode argChild : argChildren) {
      ArrayList<ResultFlag> flags = new ArrayList<ResultFlag>();
      flags.add(ResultFlag.CONSTRUCTOR);
      if ("idArg".equals(argChild.getName())) {
        flags.add(ResultFlag.ID);
      }
      resultMappings.add(buildResultMappingFromContext(argChild, resultType, flags));
    }
  }
```
```language
  private ResultMapping buildResultMappingFromContext(XNode context, Class resultType, ArrayList<ResultFlag> flags) throws Exception {
    String property = context.getStringAttribute("property");
    String column = context.getStringAttribute("column");
    String javaType = context.getStringAttribute("javaType");
    String jdbcType = context.getStringAttribute("jdbcType");
    String nestedSelect = context.getStringAttribute("select");
    String nestedResultMap = context.getStringAttribute("resultMap",
        processNestedResultMappings(context, Collections.EMPTY_LIST));
    String typeHandler = context.getStringAttribute("typeHandler");
    Class javaTypeClass = resolveClass(javaType);
    Class typeHandlerClass = resolveClass(typeHandler);
    JdbcType jdbcTypeEnum = resolveJdbcType(jdbcType);
    return builderAssistant.buildResultMapping(resultType, property, column, javaTypeClass, jdbcTypeEnum, nestedSelect, nestedResultMap, typeHandlerClass, flags);
  }
```
处理与普通的reslt相似，区别在于RequestMapping的属性ResultFlag的区别（普通为null；否则构造方法则会标识为ID或CONSTRUCTOR
```language
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
```
运行时区别在普通的result节点，是在初始化resultMap其Type对应的对象后，调用其的setXXX方法完成参数赋值；而节点产constructor则是要求在创建resultMap其Type对应的对象时，将这部分作为构造方法的参数传入（比如ImmutableAuthor只提供构造该方法和公有的get方法，并不提供set方法。初步看来确保对象创建后属性即不可修改）
###### 2.5.1.2 discriminator方式处理（主要用于相对复杂的数据结构）
Document是Book 、Magazine 的父类，而子类分别添加不同的属性：
```language
public class Document {
  private int id;
  private String title;
  private DocType type;
  private List attributes; //此处为List
}
```
```language
public class Book extends Document {
  private Integer pages;
}
```
```language
public class Magazine extends Document {
  private String city;
}
```
数据库数据：
```language
select * from DOCUMENTS
Columns: DOCUMENT_ID, DOCUMENT_TITLE, DOCUMENT_TYPE, DOCUMENT_PAGENUMBER, DOCUMENT_CITY
DEBUG [main] - <==        Row: 1, The World of Null-A, BOOK, 55, null
DEBUG [main] - <==        Row: 2, Le Progres de Lyon, NEWSPAPER, null, Lyon
DEBUG [main] - <==        Row: 3, Lord of the Rings, BOOK, 3587, null
DEBUG [main] - <==        Row: 4, Le Canard enchaine, NEWSPAPER, null, Paris
DEBUG [main] - <==        Row: 5, Le Monde, BROADSHEET, null, Paris
DEBUG [main] - <==        Row: 6, Foundation, MONOGRAPH, 557, null
-------------------------------------------------------------------------------
select a.*, b.attribute from Documents a left join Document_Attributes b on a.document_id = b.document_id order by a.document_id
DEBUG [main] - <==    Columns: DOCUMENT_ID, DOCUMENT_TITLE, DOCUMENT_TYPE, DOCUMENT_PAGENUMBER, DOCUMENT_CITY, ATTRIBUTE
DEBUG [main] - <==        Row: 1, The World of Null-A, BOOK, 55, null, English
DEBUG [main] - <==        Row: 1, The World of Null-A, BOOK, 55, null, Sci-Fi
DEBUG [main] - <==        Row: 2, Le Progres de Lyon, NEWSPAPER, null, Lyon, French
DEBUG [main] - <==        Row: 3, Lord of the Rings, BOOK, 3587, null, Fantasy
DEBUG [main] - <==        Row: 3, Lord of the Rings, BOOK, 3587, null, English
DEBUG [main] - <==        Row: 4, Le Canard enchaine, NEWSPAPER, null, Paris, null
DEBUG [main] - <==        Row: 5, Le Monde, BROADSHEET, null, Paris, null
DEBUG [main] - <==        Row: 6, Foundation, MONOGRAPH, 557, null, null
即document为主表，Document_Attributes 为属性扩展表，document与Document_Attributes 通过主键关联，document一对多Document_Attributes 
```
xml配置：
```language
  <resultMap id="documentWithAttributes" class="com.testdomain.Document" groupBy="id">
    <result property="id" column="DOCUMENT_ID"/>
    <result property="title" column="DOCUMENT_TITLE"/>
    <result property="type" column="DOCUMENT_TYPE"/>
    <result property="attributes" resultMap="Documents.documentAttributes"/>
    <discriminator column="DOCUMENT_TYPE" javaType="string">
      <subMap value="BOOK" resultMap="bookWithAttributes"/>
      <subMap value="NEWSPAPER" resultMap="newsWithAttributes"/>
    </discriminator>
  </resultMap>

    <resultMap id="bookWithAttributes" class="com.testdomain.Book" extends="documentWithAttributes">
    <result property="pages" column="DOCUMENT_PAGENUMBER"/>
  </resultMap>

  <resultMap id="newsWithAttributes" class="com.testdomain.Magazine" extends="documentWithAttributes">
    <result property="city" column="DOCUMENT_CITY"/>
  </resultMap>

   <resultMap id="documentAttributes" class="java.lang.String">
    <result property="value" column="attribute"/>
  </resultMap>
```
通过如下查询分析， <result property="attributes" resultMap="Documents.documentAttributes"/>可用于实现resultMap嵌套（即Object中的List属性）；另外discriminator 可实现简单的业务逻辑判断，即DOCUMENT_TYPE的alue为BOOK时按bookWithAttributes赋值，而value为NEWSPAPER时按newsWithAttributes赋值
```language
    <discriminator column="DOCUMENT_TYPE" javaType="string">
      <subMap value="BOOK" resultMap="bookWithAttributes"/>
      <subMap value="NEWSPAPER" resultMap="newsWithAttributes"/>
    </discriminator>
```
- 看完示例的结果的分析，再来分析源码.如若discriminator 子节点除了subMap还，还可是association、
collection、case.
1. discriminator子节点subMap时，即以value的值为key、resultMap的String值为value保存至discriminatorMap。
2. discriminator子节点association、collection、case，
```language
  <resultMap id="blogWithPosts" type="Blog">
    <id property="id" column="id"/>
    <result property="title" column="title"/>
    <association property="author" column="author_id"
                 select="selectAuthorWithInlineParams"/>
    <collection property="posts" column="id" select="selectPostsForBlog"/>
  </resultMap>
```
```language
public class Blog {

  private int id;
  private String title;
  private Author author;
  private List<Post> posts;
}
```
对于属性select，则递归解析完成初始化并返回resultMap的Id转换成类似如下的关联
```language
    <result property="favoriteDocument" resultMap="Documents.document"/>
```

#### 2.5.sql解析
```language
   sqlElement(context.evalNodes("/mapper/sql"));
```
```language
  private void sqlElement(List<XNode> list) throws Exception {
    for (XNode context : list) {
      String id = context.getStringAttribute("id");
      id = builderAssistant.applyCurrentNamespace(id);
      sqlFragments.put(id, context);
    }
  }
```
builderAssistant.applyCurrentNamespace主要用于获取sql标签对象的完整id(namespace.id),而sqlFragments定义为Map<String, XNode> sqlFragments。整体来说并没有作处理，只是保存

#### 2.6.select|insert|update|delete解析
```language
  buildStatementFromContext(context.evalNodes("select|insert|update|delete"));
```
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
- 而上面的parseDynamicTags方法除了解析selectKey之外，其实是将select、insert、update、delete内的所有内容全部解析为Node节点对象，区别在于普通针对其嵌套的如include、trim、where、set、foreach、if、choose、when、otherwise、selectKey均会有对应的Handler针对处理（这点儿其实就类似于编译原理的语法树分析）；同时标签内原始的sql或者CDATA也会被解析为Node存储到parseDynamicTags的返回结果List<SqlNode>。
- 针对keyProperty说明：在完成全部解析之后，其最终仍是创建一个新的MappedStatement至configuration，只是id是根据当前namespace、父insert的ide及固定后缀生成。区别在于KeyGenerator对象差异，具体如下：
1. 显示指定useGeneratedKeys为true或者是insert语句且MapperConfig.xml中显示指定useGeneratedKeys为true(默认为false),则即说明可使用key生成器，且默认使用Jdbc3KeyGenerator（其implements KeyGenerator）
2. 对于selectKey方式即返回其初始化时生成的keyGenerator（如SelectKeyGenerator）

#### 最终，所有的select、insert、update、delete、selectKey都会对应一个MappedStatement对象存储在configuration.