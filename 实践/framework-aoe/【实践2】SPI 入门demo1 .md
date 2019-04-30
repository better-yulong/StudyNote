- 很多框架都使用了java的SPI机制，如java.sql.Driver的SPI实现（mysql驱动、oracle驱动等）、common-logging的日志接口实现、dubbo的扩展实现等等框架；
- 当前示例以日志接口实现为例。
#### 一、日志框架开发
1. framework-aoe-spi-logger 工程定义日志服务接口 org.framework.aoe.spi.logger.Logger:
```language
public interface Logger {
	
	public void debug(String logger);
	
	public void info(String logger);
	
	public void warn(String logger);
	
	public void error(String logger);
}
```
2. framework-aoe-spi-logger 工程定义日志服务调用API org.framework.aoe.spi.logger.LoggerFactory.LoggerFactory:
```language
public class LoggerFactory {
	
	public LoggerFactory(){
	}
	
	public static Logger getLogger(){
		Logger logger = null ;
		
		ServiceLoader<Logger> serviceLoader = ServiceLoader.load(Logger.class);
		Iterator<Logger> loggers = serviceLoader.iterator();
		if(loggers.hasNext()){
			logger = loggers.next();
		}
		return logger;
	}

}
```
#### 二、Alogger项目（A实现项目）
1. 新建 ALogger-spi-impl（选择maven-archetype-quickstart），pom.xml添加依赖 framework-aoe-spi-logger。
2. 服务具体实现类 ALogger：
```language
public class ALogger implements Logger {
	public ALogger(){
	}

	public void debug(String logger) {
		System.out.println("ALogger-->debug: " + logger);
		
	}

	public void info(String logger) {
		System.out.println("ALogger-->info: " + logger);
		
	}

	public void warn(String logger) {
		System.out.println("ALogger-->warn: " + logger);
		
	}

	public void error(String logger) {
		System.out.println("ALogger-->error: " + logger);
		
	}

}
```
3. ALogger-spi-impl工程右键新建META-INF文件夹及services子文件夹。
4. 在 services文件夹中新建org.framework.aoe.spi.logger.Logger （服务接口全限定名），其内容为 com.aoe.spi.logger.ALogger （服务接口实现类全限定名）
5. 因需将 ALogger-spi-impl 打包成jar,而其依赖 framework-aoe-spi-logger 框架包，故先将该jar提交至本地仓库（因是个人研究，避免提交到公司公共仓库），而其parent又是 framework-aoe-parent，故即在framework-aoe-parent 根目录执行 mvn install .
6. ALogger-spi-impl 根目录执行 mvn install ，同步将该jar 安装到本地maven 仓库，以便后续其他测试项目基于pom.xml引用 ；然而失败。原因？系无法获取 framework-aoe-spi-logger 依赖包。
 - 但第5步明显已安装到本地仓库，而通过构建日志可发现其是尝试从远程仓库去解析并下载依赖包，所以失败。
 - 嚓，各种百度如何优先使用本地仓库获取依赖jar但均未解决。于是重新理清思路，决定先执行下 mvn --help ，然后逐个确认可用参数，发现 -o 参数的offline 模式应该可解决问题。
7. ALogger-spi-impl根目录mvn clean之后，mvn -o install即可成功。

#### 三、Blogger项目（B实现项目） ---- 参考A项目
1. 日志服务接口实现类：
```language
public class BLogger implements Logger {
	public BLogger(){
	}

	public void debug(String logger) {
		System.out.println("BLogger-->debug: " + logger);
		
	}

	public void info(String logger) {
		System.out.println("BLogger-->info: " + logger);
		
	}

	public void warn(String logger) {
		System.out.println("BLogger-->warn: " + logger);
		
	}

	public void error(String logger) {
		System.out.println("BLogger-->error: " + logger);
		
	}

}
```
2. POM.xml文件
```
  groupId>com.aoe.spi</groupId>
  <artifactId>BLogger-spi-impl</artifactId>
  <version>0.0.1-SNAPSHOT</version>
```
3. mvn -o install 部署jar至本地仓库

#### 四、META-INF及services
- 核对 ALogger-spi-impl及BLogger-spi-impl jar包，发现新增的META-INF/services/ org.framework.aoe.spi.logger.Logger 文件都未打包到jar中。
- 经确认，系原在项目根目录创建 META-INF/services方式不对，而是在 src/main/resources新建src/main/resources，并创建 org.framework.aoe.spi.logger.Logger 文件，之后重新打包即可。

#### 五、日志SPI测试
1. 新建 Logger-Test（选择maven-archetype-quickstart），pom.xml同时引入两套日志SPI实现：
```language
		<dependency>
			<groupId>com.aoe.spi</groupId>
			<artifactId>ALogger-spi-impl</artifactId>
			<version>0.0.1-SNAPSHOT</version>
		</dependency>
		<dependency>
			<groupId>com.aoe.spi</groupId>
			<artifactId>BLogger-spi-impl</artifactId>
			<version>0.0.1-SNAPSHOT</version>
		</dependency>
```
2. 创建测试类：
```language
import org.framework.aoe.spi.logger.Logger;
import org.framework.aoe.spi.logger.LoggerFactory;

public class LoggerTest {

	private final static Logger logger = LoggerFactory.getLogger();
	
	public static void main(String[] args) {
		logger.debug("this is use debug... ");
		logger.info("this is use info... ");
		logger.warn("this is use warn... ");
		logger.error("this is use error... ");
	}

}
```
3. 运行
```language
ALogger-->debug: this is use debug... 
ALogger-->info: this is use info... 
ALogger-->warn: this is use warn... 
ALogger-->error: this is use error... 
```
4. pom.xml中同时引入两套日志实现，但当前默认是按pom.xml中顺序加载，即先加载ALogger后加载BLogger，而服务访问API LoggerFactory.getLogger()方法默认返回第1个，所以调用ALogger的方法日志中打印为ALogger；若注释掉pom.xml中 ALogger jar的依赖或者调整先后顺序，由会打印BLogger。
5. 如若有多个实现，可在META-INF/services/的对应服务接口文件中配置多行，每行为一个服务接口实现类的全限定名。对于该文件中配置的多个服务接口实现类，基于ServiceLoader会依次实例化类。部分涉及到我分支或者组合情况，若基于类责任模式：
```language
public class LoggerFactory {
	
	private static ArrayList<Logger> list ;
    static {
        list = new ArrayList<Logger>();

        ServiceLoader loader = ServiceLoader.load(Logger.class);
        Iterator<Logger> it = loader.iterator();
        while (it.hasNext()) {
            list.add(it.next());
        }
    }
    
	public LoggerFactory(){
	}
	
	public static Logger getLogger(){
		System.out.println(new Date());
		return list!=null?list.get(0):null;
	}
	
	public static void logAllInfo(String info){
		for(Logger logger:list){
			logger.info("logAllInfo:" + info);
		}
	}
}
```
应用测试方法：
```language
	public static void main(String[] args) {
		logger.debug("this is use debug... ");
		logger.info("this is use info... ");
		logger.warn("this is use warn... ");
		logger.error("this is use error... ");
		LoggerFactory.logAllInfo("logger list test");
	}
```
运行结果：
```language
Tue Apr 30 17:42:57 CST 2019
ALogger-->debug: this is use debug... 
ALogger-->info: this is use info... 
ALogger-->warn: this is use warn... 
ALogger-->error: this is use error... 
ALogger-->info: logAllInfo:logger list test
BLogger-->info: logAllInfo:logger list test

```
6. 对于服务接口实现类实例化及注册，可依赖于如上的META-INF/services及ServiceLoader自动完成；其实也可参考数据库


 