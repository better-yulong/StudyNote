Mapper.xml文件解析重点分析XMLMapperBuilder，在解析MapperConfig.xml时，XMLMapperConfigBuilder类如下代码：
```language
  //XMLMapperConfigBuilder类代码片段
  private Map<String, XNode> sqlFragments = new HashMap<String, XNode>();

  XMLMapperBuilder mapperParser = new XMLMapperBuilder(reader, configuration, resource, sqlFragments);
  mapperParser.parse();
```
```language
  public XMLMapperBuilder(Reader reader, Configuration configuration, String resource, Map<String, XNode> sqlFragments) {
    super(configuration);
    this.builderAssistant = new MapperBuilderAssistant(configuration, resource);
    this.parser = new XPathParser(reader, true, new XMLMapperEntityResolver(), configuration.getVariables());
    this.sqlFragments = sqlFragments;
    this.resource = resource;
  }
```
