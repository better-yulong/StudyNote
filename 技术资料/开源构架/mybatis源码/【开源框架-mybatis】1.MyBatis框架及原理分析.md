转自：https://www.cnblogs.com/luoxn28/p/6417892.html
### MyBatis简介
- MyBatis 是支持定制化 SQL、存储过程以及高级映射的优秀的持久层框架，其主要就完成2件事情：
1. 封装JDBC操作
2. 利用反射打通Java类与SQL语句之间的相互转换
- MyBatis的主要设计目的就是让我们对执行SQL语句时对输入输出的数据管理更加方便，所以方便地写出SQL和方便地获取SQL的执行结果才是MyBatis的核心竞争力。

### MyBatis的配置
MyBatis框架和其他绝大部分框架一样，需要一个配置文件，其配置文件大致如下：
```language
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE configuration PUBLIC "-//mybatis.org//DTD Config 3.0//EN" "http://mybatis.org/dtd/mybatis-3-config.dtd">

<configuration>
    <settings>
        <setting name="cacheEnabled" value="true"/>
        <setting name="lazyLoadingEnabled" value="false"/>
        <!--<setting name="logImpl" value="STDOUT_LOGGING"/> &lt;!&ndash; 打印日志信息 &ndash;&gt;-->
    </settings>

    <typeAliases>
        <typeAlias type="com.luo.dao.UserDao" alias="User"/>
    </typeAliases>

    <environments default="development">
        <environment id="development">
            <transactionManager type="JDBC"/> <!--事务管理类型-->
            <dataSource type="POOLED">
                <property name="username" value="name"/>
                <property name="password" value="pwd"/>
                <property name="driver" value="com.mysql.jdbc.Driver"/>
                <property name="url" value="jdbc:mysql://127.0.0.1/ssh_study"/>
            </dataSource>
        </environment>
    </environments>

    <mappers>
        <mapper resource="userMapper.xml"/>
    </mappers>

</configuration>
```
