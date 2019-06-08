### 一.运行前准备
#### 1.1 基于Mapper.xml的namespace及id关联 
之前的单元测试方法调用查询源码为：
```language
   //session对应类DefaultSqlSession
  session.selectList("domain.blog.mappers.AuthorMapper.selectAllAuthors");
```
继续深入分析可明白，在解析Mapper.xml文件时，会所有的select、delete、update、insert节点生成对应的MappedStatement对象并存储至configuration对象（其存储为Map结构，key即selectList的参数：由Mapper.xml的namespace值及节点的id拼装；value为MappedStatement对象），而在session的selectList首先则是根据参数去获取MappedStatement实例，然后初始化executro执行后面的逻辑.




