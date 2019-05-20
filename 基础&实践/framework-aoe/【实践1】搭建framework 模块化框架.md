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
##### 3、多模化工程导入
eclipse导入或open工程时，一般只需导入paraent项目即可，因pom.xml的module关联，子模块会自动导入。

##### 4、代码管理
开发环境基于eclipse，将当前工作空间上传至git 远程仓库：
```
git init
git add README.md
git commit -m "first commit"
git remote add origin https://github.com/better-yulong/study-code-effective-java.git
git push -u origin master

```
但发现，.metadata、RemoteSystemsTempFiles等目录未做ignore处理。
1. 创建.gitignore文件：
```language
# 忽略当前目录下的文件夹（看下面两行以“/”结尾表示忽略的是个文件夹）
.metadata/
RemoteSystemsTempFiles/

# 忽略子目录下的文件夹
**/.settings
**/target

# 忽略子目录下的文件
**.classpath
**.project
```
其中linux平台，在根目录cat .gitignore文件即可；而windows文件不能直接创建（无法识别后缀），即先行按如上内容在根目录编辑好gitignore.txt文件，之后命令行cmd运行：ren gitignore.txt .gitignore  即可。
2. 对于新工程未提交git之前按如上方式先行创建.gitignore文件，并使用git check-ignore命令检查；但若项目该忽略的工程已经提交至远程创建之后，则创建完成 .gitignore文件后还需如下操作：
```
	git rm -r --cached .
	git add .
	git commit -m 'update .gitignore'
	git push origin master
```
刷新git web页面，如预期远程仓库已无忽略的文件和目录。

##### 5、jar构建
如若在framework-aoe-parent 根目录执行 mvn package，默认会将 parent本身及pom.xml中的所有module依次全部打包。如若只想构建某个模块，则只需在对应模块根目录执行 mvn package即可。