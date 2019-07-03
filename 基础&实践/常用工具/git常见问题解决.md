### git for windows下的Filename too long
解决
打开git命令行:git config --global core.longpaths true

### eclipse 提交git/svn忽略文件
1. 在主项目文件夹下面创建.gitignore文件内容如下
```language
# 忽略子目录下的文件
**.classpath
**.project
RemoteSystemsTempFiles/

.metadata/
Servers/
.recommenders/
**/target/
**/.classpath
**/.project
**/.settings
.idea/*
**/out/
**.iml
**.DS_Store
```
2. 提交所有文件（）
3. 如果添加了.gitignore还是没有作用那是因为你的项目已经提交到仓库了,这个时候需要清除仓库的数据