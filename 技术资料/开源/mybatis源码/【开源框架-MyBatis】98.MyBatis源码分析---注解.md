BoundAuthorMapper


 该部分代码可结合 DefaultParameterHandler的setParameters方法源码来理解
```language
    //DefaultParameterHandler的setParameters方法
      public void setParameters(PreparedStatement ps)
      throws SQLException {
    ErrorContext.instance().activity("setting parameters").object(mappedStatement.getParameterMap().getId());
    List<ParameterMapping> parameterMappings = boundSql.getParameterMappings();
    if (parameterMappings != null) {
      MetaObject metaObject = parameterObject == null ? null : configuration.newMetaObject(parameterObject);
      for (int i = 0; i < parameterMappings.size(); i++) {
        ParameterMapping parameterMapping = parameterMappings.get(i);
        if (parameterMapping.getMode() != ParameterMode.OUT) {
          Object value;
          String propertyName = parameterMapping.getProperty();
          PropertyTokenizer prop = new PropertyTokenizer(propertyName);
          if (parameterObject == null) {
            value = null;
          } else if (typeHandlerRegistry.hasTypeHandler(parameterObject.getClass())) {
            value = parameterObject;
          } else if (boundSql.hasAdditionalParameter(propertyName)) {
            value = boundSql.getAdditionalParameter(propertyName);
          } else if (propertyName.startsWith(ForEachSqlNode.ITEM_PREFIX)
              && boundSql.hasAdditionalParameter(prop.getName())) {
            value = boundSql.getAdditionalParameter(prop.getName());
            if (value != null) {
              value = configuration.newMetaObject(value).getValue(propertyName.substring(prop.getName().length()));
            }
          } else {
            value = metaObject == null ? null : metaObject.getValue(propertyName);
          }
          TypeHandler typeHandler = parameterMapping.getTypeHandler();
          if (typeHandler == null) {
            throw new ExecutorException("There was no TypeHandler found for parameter " + propertyName + " of statement " + mappedStatement.getId());
          }
          typeHandler.setParameter(ps, i + 1, value, parameterMapping.getJdbcType());
        }
      }
    }
  }
```
```language
  <select id="selectAuthor"
          parameterMap="selectAuthor"
          resultMap="selectAuthor">
    select id, username, password, email, bio, favourite_section
    from author where id = ?
  </select>
```
- 解析select语句时（其他insert、udpate、delete语句类似）创建MappedStatement实例时，获取boundSql关联的parameterMappings(之前已经根据select标签的parameterMap将parameterMapping关联到boundSql)
- 获取业务调用query方法是传入的parameterOjbect，若不为空则将其包装为metaObject对象
- 稍后遍历parameterMapping列表：
1. 判断ParameterMode是否不为OUT（即为IN、INOUT其中之一），说明该paramter可作为入参；否则直接结束不处理---正好对应AuthorMapper.xml的parameter的mode属性




