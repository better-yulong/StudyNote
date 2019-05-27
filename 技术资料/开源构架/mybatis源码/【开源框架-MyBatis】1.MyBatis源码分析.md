### 准备
- 基于首次阅读框架源码尽量选择早期版本，因为更接近到原始设计思路，易于理解，支持mybatis3.0.1版本（mybatis与ibatis有较大差别，做了比较大的重写）。近期版本的mybatis源码可在github中搜索mybatis-3查找，但前期的已无github可用，在mybatis官网可找到早期mybatis的源码zip包。（已下载并上传至github:https://github.com/better-yulong/StudyNote-Resource/blob/master/StudyNote-Resource/source-zip/mybatis-3-mybatis-3.0.1.zip）。
- 解压并import maven project至eclipse，工程各种报错，发现该工程导入build path关联的jre1.5，初步怀疑与jdk版本有关系，查看mybatis.iml文件其中提到LANGUAGE_LEVEL="JDK_1_5"，尝试将JDK切换至1.6，再次构建工程错误消失。

### 一. Mybatis 源码单元测试
- 工程导入之后其实是一筹莫展，根本无从下手，考虑到工程自带 src/test/java中各种代码，比如数据表建表脚本、数据insert脚本，初步分析这应该就是用来初始化单元测试依赖环境的。但继续查看，类过多且并未找到真实可运行的测试类。无奈之举（后面发现误打误撞对了），在 src/test/java 中搜索Junit关键字，查找可运行的单元测试方法，一搜还果然不少，再结合junit关键字及select关键字，确实找到了@Test注解标识可可执行单元测试类：SqlSessionTest。
1. 初步过了下SqlSessionTest代码尝试直接执行单元测试方法: shouldSelectAllAuthors（不是运行该类所有单元测试方法），发现直接报错：
```language
java.lang.NullPointerException
	at org.eclipse.jdt.internal.junit4.runner.SubForestFilter.shouldRun(SubForestFilter.java:81)
	at org.junit.internal.runners.TestClassMethodsRunner.filter(TestClassMethodsRunner.java:84)
	at org.junit.runner.manipulation.Filter.apply(Filter.java:47)
	at org.junit.internal.runners.TestClassRunner.filter(TestClassRunner.java:64)
	at org.junit.runner.manipulation.Filter.apply(Filter.java:47)
	at org.junit.internal.requests.FilterRequest.getRunner(FilterRequest.java:34)
```
- 尝试通过debug但并未找到原因，无奈借助度娘，发现与junit jar有关系。因为junit单元测试依赖eclipse的Junit插件（自带Junit jar包），但该工程为方便测试其pom.xml有依赖Junit jar包，因工程中自带jar而buildpath指定顺序会使得test时默认优先加载工程中自带的 junit 包，而其恰好又与版本新的eclipse的Junit插件不兼容，故根据网上提示，修改BuildPath：引入默认的Junit jar包（右击项目名：config buildpath-->Libraries-->add library-->JUnit）并调整排序至最上面，优先加载：

![1-1](https://github.com/better-yulong/StudyNote-Resource/blob/master/StudyNote-Resource/tech/framework/mybatis/mybatis-1-1.PNG)
![1-2](https://github.com/better-yulong/StudyNote-Resource/blob/master/StudyNote-Resource/tech/framework/mybatis/mybatis-1-2.PNG)
- 单独运行 shouldSelectAllAuthors 单元测试方法，结果：
```language
Error executing: CREATE TABLE node (
id  INT NOT NULL,
parent_id INT,
PRIMARY KEY(id)
)
.  Cause: java.sql.SQLException: Table/View 'NODE' already exists in Schema 'APP'.
Error executing: INSERT INTO node (id, parent_id) VALUES (1,null)
.  Cause: java.sql.SQLIntegrityConstraintViolationException: The statement was aborted because it would have caused a duplicate key value in a unique or primary key constraint or unique index identified by 'SQL190527132731780' defined on 'NODE'.
	DEBUG [main] - ooo Connection Opened
	DEBUG [main] - ==>  Executing: select * from author 
	DEBUG [main] - ==> Parameters: 
	DEBUG [main] - <==    Columns: ID, USERNAME, PASSWORD, EMAIL, BIO, FAVOURITE_SECTION
	DEBUG [main] - <==        Row: 101, jim, ********, jim@ibatis.apache.org, , NEWS
	DEBUG [main] - <==        Row: 102, sally, ********, sally@ibatis.apache.org, null, VIDEOS
	DEBUG [main] - xxx Connection Closed
```
从运行结果来看，select测试通过；
### 二. 单元测试源码分析
基于测试通过的单元测试方法：shouldSelectAllAuthors，结合源码分析：
```language
public class SqlSessionTest extends BaseDataTest {
  private static SqlSessionFactory sqlMapper;

  @BeforeClass
  public static void setup() throws Exception {
    createBlogDataSource();
    final String resource = "org/apache/ibatis/builder/MapperConfig.xml";
    final Reader reader = Resources.getResourceAsReader(resource);
    sqlMapper = new SqlSessionFactoryBuilder().build(reader);
  }

   //引得省略......
   
  @Test
  public void shouldSelectAllAuthors() throws Exception {
    SqlSession session = sqlMapper.openSession(TransactionIsolationLevel.SERIALIZABLE);
    try {
      List<Author> authors = session.selectList("domain.blog.mappers.AuthorMapper.selectAllAuthors");
      assertEquals(2, authors.size());
    } finally {
      session.close();
    }
  }
```
SqlSessionTest父类：
```language
public class BaseDataTest {

  public static final String BLOG_PROPERTIES = "databases/blog/blog-derby.properties";
  public static final String BLOG_DDL = "databases/blog/blog-derby-schema.sql";
  public static final String BLOG_DATA = "databases/blog/blog-derby-dataload.sql";

  public static final String JPETSTORE_PROPERTIES = "databases/jpetstore/jpetstore-hsqldb.properties";
  public static final String JPETSTORE_DDL = "databases/jpetstore/jpetstore-hsqldb-schema.sql";
  public static final String JPETSTORE_DATA = "databases/jpetstore/jpetstore-hsqldb-dataload.sql";

  public static UnpooledDataSource createUnpooledDataSource(String resource) throws IOException {
    Properties props = Resources.getResourceAsProperties(resource);
    UnpooledDataSource ds = new UnpooledDataSource();
    ds.setDriver(props.getProperty("driver"));
    ds.setUrl(props.getProperty("url"));
    ds.setUsername(props.getProperty("username"));
    ds.setPassword(props.getProperty("password"));
    return ds;
  }

  public static PooledDataSource createPooledDataSource(String resource) throws IOException {
    Properties props = Resources.getResourceAsProperties(resource);
    PooledDataSource ds = new PooledDataSource();
    ds.setDriver(props.getProperty("driver"));
    ds.setUrl(props.getProperty("url"));
    ds.setUsername(props.getProperty("username"));
    ds.setPassword(props.getProperty("password"));
    return ds;
  }

  public static void runScript(DataSource ds, String resource) throws IOException, SQLException {
    Connection connection = ds.getConnection();
    try {
      ScriptRunner runner = new ScriptRunner(connection);
      runner.setAutoCommit(true);
      runner.setStopOnError(false);
      runner.setLogWriter(null);
      runScript(runner, resource);
    } finally {
      connection.close();
    }
  }

  public static void runScript(ScriptRunner runner, String resource) throws IOException, SQLException {
    Reader reader = Resources.getResourceAsReader(resource);
    try {
      runner.runScript(reader);
    } finally {
      reader.close();
    }
  }

  public static DataSource createBlogDataSource() throws IOException, SQLException {
    DataSource ds = createUnpooledDataSource(BLOG_PROPERTIES);
    runScript(ds, BLOG_DDL);
    runScript(ds, BLOG_DATA);
    return ds;
  }

  public static DataSource createJPetstoreDataSource() throws IOException, SQLException {
    DataSource ds = createUnpooledDataSource(JPETSTORE_PROPERTIES);
    runScript(ds, JPETSTORE_DDL);
    runScript(ds, JPETSTORE_DATA);
    return ds;
  }

  @Test
  public void dummy() {
  }

}
```
#### 2.1 BeforeClass分析
```language
  @BeforeClass
  public static void setup() throws Exception {
    createBlogDataSource();
    final String resource = "org/apache/ibatis/builder/MapperConfig.xml";
    final Reader reader = Resources.getResourceAsReader(resource);
    sqlMapper = new SqlSessionFactoryBuilder().build(reader);
  }
```
##### 2.1.1 BeforeClass --> createBlogDataSource() 分析
```language
  public static DataSource createBlogDataSource() throws IOException, SQLException {
    DataSource ds = createUnpooledDataSource(BLOG_PROPERTIES);
    runScript(ds, BLOG_DDL);
    runScript(ds, BLOG_DATA);
    return ds;
  }
```
从方法名可知道该方法用于创建数据源，涉及两个子方法：createUnpooledDataSource、runScript
###### 2.1.1.1 BeforeClass --> createBlogDataSource() -->createUnpooledDataSource
```language
  public static UnpooledDataSource createUnpooledDataSource(String resource) throws IOException {
    Properties props = Resources.getResourceAsProperties(resource);
    UnpooledDataSource ds = new UnpooledDataSource();
    ds.setDriver(props.getProperty("driver"));
    ds.setUrl(props.getProperty("url"));
    ds.setUsername(props.getProperty("username"));
    ds.setPassword(props.getProperty("password"));
    return ds;
  }
```
其中数据库连接配置文件：
```language
driver=org.apache.derby.jdbc.EmbeddedDriver
url=jdbc:derby:ibderby;create=true
username=
password=
```
1. derby是apache的一个开源数据库产品，有丰富的特性。它支持client/server模式外，也支持embedded模式.
2. MyBatis内置了两个DataSource的实现：UnpooledDataSource，该数据源对于每次获取请求都简单的打开和关闭连接。PooledDataSource，该数据源在Unpooled的基础上构建了连接池。
###### 2.1.1.2 BeforeClass --> createBlogDataSource() -->runScript
```language
  //BaseDataTest类：
  public static void runScript(DataSource ds, String resource) throws IOException, SQLException {
    Connection connection = ds.getConnection();
    try {
      ScriptRunner runner = new ScriptRunner(connection);
      runner.setAutoCommit(true);
      runner.setStopOnError(false);
      runner.setLogWriter(null);
      runScript(runner, resource);
    } finally {
      connection.close();
    }
  }

  public static void runScript(ScriptRunner runner, String resource) throws IOException, SQLException {
    Reader reader = Resources.getResourceAsReader(resource);
    try {
      runner.runScript(reader);
    } finally {
      reader.close();
    }
  }

  //ScriptRunner类：
  public void runScript(Reader reader) {
    setAutoCommit();

    try {
      if (sendFullScript) {
        executeFullScript(reader);
      } else {
        executeLineByLine(reader);
      }
    } finally {
      rollbackConnection();
    }
  }
```
