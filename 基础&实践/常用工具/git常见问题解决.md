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
https://www.cnblogs.com/jing-tian/p/11154485.html
Git中从远程的分支获取最新的版本到本地有这样2个命令：fetch和pull
++相同点：++
　首先在作用上他们的功能是大致相同的，都是起到了更新代码的作用。
1.fetch：相当于是从远程获取最新版本到本地，不会自动merge
　git fetch orgin master //将远程仓库的master分支下载到本地当前branch中
　git log -p master  ..origin/master //比较本地的master分支和origin/master分支的差别

　git merge origin/master //进行合并
这个命令会访问远程仓库，从中拉取所有你还没有的数据。 执行完成后，你将会拥有那个远程仓库中所有分支的引用，可以随时合并或查看。

如果你使用git clone 命令克隆了一个仓库，命令会自动将其添加为远程仓库（git remote -v）并默认以 “origin” 为简写。 所以，git fetch origin 会抓取克隆（或上一次抓取）后新推送的所有工作。 必须注意 git fetch 命令会将数据拉取到你的本地仓库 - 它并不会自动合并或修改你当前的工作。 当准备好时你必须手动将其合并入你的工作。

如果你有一个分支设置为跟踪一个远程分支，可以使用 git pull命令来自动的抓取然后合并远程分支到当前分支。 这对你来说可能是一个更简单或更舒服的工作流程；默认情况下，git clone 命令会自动设置本地 master 分支跟踪克隆的远程仓库的 master 分支（或不管是什么名字的默认分支）。 运行 git pull 通常会从最初克隆的服务器上抓取数据并自动尝试合并到当前所在的分支。
2.git pull：相当于是从远程获取最新版本并merge到本地
git pull origin master    //相当于git fetch 和 git merge
注:用git pull更新代码的话就比较简单暴力了但是根据commit ID来看的话，他们实际的实现原理是不一样的，所以不要用git pull，用git fetch和git merge更加安全。

 

6.放弃本地修改，强制回退
https://blog.csdn.net/sayyy/article/details/83895698
$ git fetch --all
$ git reset --hard origin/master 
$ git pull