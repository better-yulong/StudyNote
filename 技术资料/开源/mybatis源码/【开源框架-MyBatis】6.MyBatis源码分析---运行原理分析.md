### 一.运行前准备
#### 1.1 基于Mapper.xml的namespace及id关联 
之前的单元测试方法调用查询源码为：
```language
   //session对应类DefaultSqlSession
  session.selectList("domain.blog.mappers.AuthorMapper.selectAllAuthors");
```
继续深入分析可明白，在解析Mapper.xml文件时，会所有的select、delete、update、insert节点生成对应的MappedStatement对象并存储至configuration对象（其存储为Map结构，key即selectList的参数：由Mapper.xml的namespace值及节点的id拼装；value为MappedStatement对象），而在session的selectList首先则是根据参数去获取MappedStatement实例，然后初始化executro执行后面的逻辑.

#### 1.2 基于Dao文件
```language
  @Test
  public void shouldSelectAuthorsUsingMapperClass() {
    SqlSession session = sqlMapper.openSession();
    try {
      AuthorMapper mapper = session.getMapper(AuthorMapper.class);
      List authors = mapper.selectAllAuthors();
      assertEquals(2, authors.size());
    } finally {
      session.close();
    }
  }
```
##### 1.2.1 获取AuthorMapper对象
```language
  //MapperRegistry类
  public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
    if (!knownMappers.contains(type))
      throw new BindingException("Type " + type + " is not known to the MapperRegistry.");
    try {
      return MapperProxy.newMapperProxy(type, sqlSession);
    } catch (Exception e) {
      throw new BindingException("Error getting mapper instance. Cause: " + e, e);
    }
  }
```
session.getMapper最终调用的是MapperRegistry类的getMapper方法，此会先判断type是否可识别（这点在上一篇笔记最后有讲解：在mapper.xml文件解析完成之后，会把对应的namespace对应Dao的class存入knownMappers并解析Dao接口）
```language
  //MapperProxy类
  public static <T> T newMapperProxy(Class<T> mapperInterface, SqlSession sqlSession) {
    ClassLoader classLoader = mapperInterface.getClassLoader();
    Class[] interfaces = new Class[]{mapperInterface};
    MapperProxy proxy = new MapperProxy(sqlSession);
    return (T) Proxy.newProxyInstance(classLoader, interfaces, proxy);
  }

  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    try {
      if (!OBJECT_METHODS.contains(method.getName())) {
        final Class declaringInterface = findDeclaringInterface(proxy, method);
        final MapperMethod mapperMethod = new MapperMethod(declaringInterface, method, sqlSession);
        final Object result = mapperMethod.execute(args);
        if (result == null && method.getReturnType().isPrimitive()) {
          throw new BindingException("Mapper method '" + method.getName() + "' (" + method.getDeclaringClass() + ") attempted to return null from a method with a primitive return type (" + method.getReturnType() + ").");
        }
        return result;
      }
    } catch (SQLException e) {
      e.printStackTrace();
    }
    return null;
  }
```
所以结果可理解为生成代理Proxy0类的实例对象，即Proxy0 etends Proxy implements AuthorMapper；且因Proxy为如下构造函数：
```language
  protected Proxy(InvocationHandler paramInvocationHandler)
  {
    this.h = paramInvocationHandler;
  }
```
即Proxy0类也有同样的构造方法，基于动态代理模式则调用Proxy0类的实例对象的方法均会被重写成调用MapperProxy的invoke方法。
- 总的来说：session.getMapper(AuthorMapper.class)返回是的基于动态代理模式生成的代理对象，其


