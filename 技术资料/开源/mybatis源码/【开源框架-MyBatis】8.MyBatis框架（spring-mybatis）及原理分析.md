转自：https://www.cnblogs.com/luoxn28/p/6417892.html
https://my.oschina.net/zudajun/blog/666223

### MyBatis简介
- MyBatis 是支持定制化 SQL、存储过程以及高级映射的优秀的持久层框架，其主要就完成2件事情：
1. 封装JDBC操作
2. 利用反射打通Java类与SQL语句之间的相互转换
- MyBatis的主要设计目的就是让我们对执行SQL语句时对输入输出的数据管理更加方便，所以方便地写出SQL和方便地获取SQL的执行结果才是MyBatis的核心竞争力。

### MyBatis的配置
MyBatis框架和其他绝大部分框架一样，需要一个配置文件，其配置文件大致如下：
```language
   	<!-- 自动扫描了所有的XxxxMapper.java，这样就不用一个一个手动配置Mpper的映射了，只要Mapper接口类和Mapper映射文件对应起来就可以了。 -->
	<bean class="org.mybatis.spring.mapper.MapperScannerConfigurer">
		<property name="basePackage" value="com.sfpay.sypay.**.dao" />
	</bean>

	<bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean"
		p:dataSource-ref="dataSource" p:configLocation="classpath:config/mybatis-config.xml"
		p:mapperLocations="classpath*:mapper/*.xml" />
```
```language
--mybatis-config.xml文件
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE configuration PUBLIC "-//mybatis.org//DTD Config 3.0//EN" "http://mybatis.org/dtd/mybatis-3-config.dtd">

<configuration>
    <settings>
        <setting name="cacheEnabled" value="true"/>
        <setting name="lazyLoadingEnabled" value="false"/>
        <!--<setting name="logImpl" value="STDOUT_LOGGING"/>
    </settings>

    <typeAliases>
        <typeAlias type="com.test.dao.UserDao" alias="User"/>
    </typeAliases>

     <mappers>
        <mapper resource="userMapper.xml"/>
    </mappers>

</configuration>
```
前面已经对MyBatis的源码有了深入的分析，但是平时我们使用更多是基于spring-mybatis整合无缝使用，即直接注入Dao实例并调用对应方法，那具体又是怎么实现的呢？

### MyBatis&Spring简介
- 参考：http://www.mybatis.org/spring/scm.html  https://github.com/mybatis/spring
#### 说明
参考mybatis在 github 的说明：Mybatis-Spring 适配器是一易于使用的Spring连接Mybati sql映射框架，即可理解其为适配层

