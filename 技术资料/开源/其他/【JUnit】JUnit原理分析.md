### 一.背景
之前虽然会频繁使用Junit进行单元测试，但从未较为深入的理解过，而在分析Mybatis源码时，发现同一单元测试方法多次执行时，上次执行的部分Derby缓存数据后续执行仍然可用。原本单纯的认为每次Junit执行会开启一个新的JVM进程，但验证发现并非如此；故此处针对性理解下。

### 二.分析
#### 1. 调试代码，查看进程信息：
```language
 @Test
  public void shouldSelectAllAuthors() throws Exception {
    SqlSession session = sqlMapper.openSession(TransactionIsolationLevel.SERIALIZABLE);
    try {
      List<Author> authors = session.selectList("domain.blog.mappers.AuthorMapper.selectAllAuthors");
      assertEquals(2, authors.size()); //断点
    } finally {
      session.close();
    }
  }
```
在如上assertEquals行添加断点，以Debugg方式运行Junit测试，运行至断点处后，执行jps -lvm:
```language
8308 sun.tools.jps.Jps -lvm -Denv.class.path=.;D:\profiles;D:\work\java\jdklib;D:\work\java\jdklib\tools.jar;E:\study\apache-jmeter-2.9\lib\ext\ApacheJMeter_core.jar;E:\study\apa
che-jmeter-2.9\lib\jorphan.jar; -Dapplication.home=D:\work\java\jdk -Xms8m
7772 org.eclipse.jdt.internal.junit.runner.RemoteTestRunner -version 3 -port 53059 -testLoaderClass org.eclipse.jdt.internal.junit4.runner.JUnit4TestLoader -loaderpluginname org.
eclipse.jdt.junit4.runtime -test org.apache.ibatis.session.SqlSessionTest:shouldSelectAllAuthors -agentlib:jdwp=transport=dt_socket,suspend=y,address=localhost:53061 -ea -Dfile.e
ncoding=UTF-8
7132  -Dosgi.requiredJavaVersion=1.7 -XX:+UseStringDeduplication -Dosgi.requiredJavaVersion=1.7 -Xverify:none -XX:+DisableExplicitGC -Xnoclassgc -XX:+UseParNewGC -XX:+UseConcMark
SweepGC -XX:CMSInitiatingOccupancyFraction=85 -XX:-UseGCOverheadLimit -Dhttps.protocols=TLSv1,TLSv1.1,TLSv1.2 -Djava.io.tmpdir=D:\tmp -Dfile.encoding=UTF-8 -Duser.home=D:\tmp -Xm
s256m -Xmx1024m -XX:MaxPermSize=256M -XX:PermSize=128m -verbose:gc -XX:+PrintGCDetails -XX:+PrintGCDateStamps -Xloggc:D:/tmp/eclipse-jee-gc.log
```
- 从日志来看：
1. PID 8308 为jps运行时进程，可忽略。
2. PID 7772 为RemoteTestRunner，即是单元测试进程。
3. PID 7312 为Eclipse 的JVM进程，具体怎么确认可根据笔记"【Eclipse】eclipse.ini 文件理解及eclipse GC分析示例"。

- 如果单纯从上面的日志来看，仍然无法解释多次运行缓存可复用的问题，于是乎度娘对RemoteTestRunner 做下了解，但没有太多可用信息，于是乎尝试分析RemoteTestRunner 源码。正好，发现了部分有用的信息：
```language
源码：
	/**
	 * Host to connect to, default is the localhost
	 */
	private String fHost= ""; //$NON-NLS-1$
	/**
	 * Port to connect to.
	 */
	private int fPort= -1;

		/**
	 * The main entry point.
	 *
	 * @param args Parameters:
	 * <pre>-classnames: the name of the test suite class
	 * -testfilename: the name of a file containing classnames of test suites
	 * -test: the test method name (format classname testname)
	 * -host: the host to connect to default local host
	 * -port: the port to connect to, mandatory argument
	 * -keepalive: keep the process alive after a test run
     * </pre>
     */
	public static void main(String[] args) {
		try {
			RemoteTestRunner testRunServer= new RemoteTestRunner();
			testRunServer.init(args);
			testRunServer.run();
		} catch (Throwable e) {
			e.printStackTrace(); // don't allow System.exit(0) to swallow exceptions
		} finally {
			// fix for 14434
			System.exit(0);
		}
	}

	protected void run() {
		if (!connect())
			return;
		if (fRerunTest != null) {
			rerunTest(new RerunRequest(Integer.parseInt(fRerunTest), fTestClassNames[0], fTestName));
			return;
		}

		FirstRunExecutionListener listener= firstRunExecutionListener();
		fExecution= new TestExecution(listener, getClassifier());
		runTests(fExecution);
		if (fKeepAlive)
			waitForReruns();

		shutDown();

	}

```
```language
API接口文档：
public static void main(String[] args)
The main entry point. Parameters
 -classnames: the name of the test suite class
 -testfilename: the name of a file containing classnames of test suites
 -test: the test method name (format classname testname) 
 -host: the host to connect to default local host 
 -port: the port to connect to, mandatory argument 
 -keepalive: keep the process alive after a test run
 
```
Eclipse 调试调用栈：
```language
Thread [main] (Suspended (breakpoint at line 97 in SqlSessionTest))	
	SqlSessionTest.shouldSelectAllAuthors() line: 97	
	NativeMethodAccessorImpl.invoke0(Method, Object, Object[]) line: not available [native method]	
	NativeMethodAccessorImpl.invoke(Object, Object[]) line: not available	
	FrameworkMethod$1(ReflectiveCallable).run() line: 12	
	FrameworkMethod.invokeExplosively(Object, Object...) line: 47	
	InvokeMethod.evaluate() line: 17	
	BlockJUnit4ClassRunner(ParentRunner<T>).runLeaf(Statement, Description, RunNotifier) line: 325	
	
	ParentRunner$2.evaluate() line: 268	
	RunBefores.evaluate() line: 26	
	BlockJUnit4ClassRunner(ParentRunner<T>).run(RunNotifier) line: 363	
	JUnit4TestReference.run(TestExecution) line: 86	
	TestExecution.run(ITestReference[]) line: 38	
	RemoteTestRunner.runTests(String[], String, TestExecution) line: 459	
	RemoteTestRunner.runTests(TestExecution) line: 678	
	RemoteTestRunner.run() line: 382	
	RemoteTestRunner.main(String[]) line: 192	
```




