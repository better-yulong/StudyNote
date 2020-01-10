### git for windows下的Filename too long
解决
打开git命令行:git config --global core.longpaths true

### eclipse 提交git/svn忽略文件
1. 在主项目文件夹下面创建.gitignore文件内容如下
```language
	# 忽略当前目录下的文件夹（看下面两行以“/”结尾表示忽略的是个文件夹）
.metadata/
RemoteSystemsTempFiles/
.recommenders/
Servers/

# 忽略子目录下的文件夹
**/.settings
**/target

# 忽略子目录下的文件
**.classpath
**.project
*.class

```
2. 提交所有文件（git add . / git commit -m 'comment' / git push origin master）
3. 如果添加了.gitignore还是没有作用那是因为你的项目已经提交到仓库了,这个时候需要清除仓库的数据
```language
	// 清除仓库的所有的数据
	git rm -r --cached .
	// 然后添加
	git add .
	// 最后提交本地
	git commit -m 'update .gitignore'
	// 与远程同步
```

4. Git操作----删除untracked files（即新建文件暂未使用git add 添加至本地暂存空间）
# 删除 untracked files
git clean -f
 
# 连 untracked 的目录也一起删掉
git clean -fd
 
# 连 gitignore 的untrack 文件/目录也一起删掉 （慎用，一般这个是用来删掉编译出来的 .o之类的文件用的）
git clean -xfd
 
# 在用上述 git clean 前，墙裂建议加上 -n 参数来先看看会删掉哪些文件，防止重要文件被误删
git clean -nxfd
git clean -nf
git clean -nfd

5. git pull和git fetch的区别


6. 