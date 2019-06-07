Mapper.xml文件解析重点分析XMLMapperBuilder，在解析MapperConfig.xml时，XMLMapperConfigBuilder类如下代码：
```language
 	XMLMapperBuilder mapperParser = new XMLMapperBuilder(reader, configuration, resource, sqlFragments);
	mapperParser.parse();
```
