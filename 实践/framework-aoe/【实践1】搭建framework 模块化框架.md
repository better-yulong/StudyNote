- 目标：搭建模块化的业务开发框架
####  一、搭建framework 框架
##### 1、创建框架父项目
基于eclipse 创建 Maven Project （maven-archetype-quickstart）工程framework-aoe-parent ,之后把pom文件中的packing标签改成pom.,并右键右键项目->maven->update maven Configuration...  
```language
  <groupId>com.aoe.framework</groupId>
  <artifactId>framework-aoe-parent</artifactId>
  <version>0.0.1-SNAPSHOT</version>
  <packaging>pom</packaging>
```
##### 2、创建子模块（后续第一个示例基于SPI&Logger示例）
1. 右键 framework-aoe-parent-->new-->Project,选择maven Module,点击 Next> 输入名称：framework-aoe-spi-logger ,之后next（其中仍选择maven-archetype-quickstart），最后完成。
2. 此时可发现相邻工程：framework-aoe-parent、framework-aoe-spi-m1，其中两个工程重点pom.xml内容：
```language
<parent>
    <groupId>com.aoe.framework</groupId>
    <artifactId>framework-aoe-parent</artifactId>
    <version>0.0.1-SNAPSHOT</version>
  </parent>
  <groupId>com.aoe.framework</groupId>
  <artifactId>framework-aoe-spi-logger</artifactId>
  <version>0.0.1-SNAPSHOT</version>
  <name>framework-aoe-spi-logger</name>
```
```language
  <groupId>com.aoe.framework</groupId>
  <artifactId>framework-aoe-parent</artifactId>
  <version>0.0.1-SNAPSHOT</version>
  <packaging>pom</packaging>

  <name>framework-parent</name>
  <properties>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
  </properties>

  <modules>
    <module>framework-aoe-spi-logger</module>
  </modules>
```


