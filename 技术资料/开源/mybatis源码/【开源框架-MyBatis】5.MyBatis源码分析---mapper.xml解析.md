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
后续以AuthorMapper.xml文件为例分析解读： parser.evalNode("/mapper")获取Mapper
