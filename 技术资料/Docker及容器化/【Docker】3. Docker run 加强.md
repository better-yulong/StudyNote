
### ubuntu 容器
### 1.1 运行ubuntu容器
一般可先行docker pull ubuntu方式先行下载镜像，之后基于docker run运行容器；但实例也可直接如下运行，当镜像不存在时，会默认从Docker Hub中下载。
```
[root@localhost ~]# docker run -i -t -d ubuntu:latest
Unable to find image 'ubuntu:latest' locally
latest: Pulling from library/ubuntu
5c939e3a4d10: Pull complete 
c63719cdbe7a: Pull complete 
19a861ea6baf: Pull complete 
651c9d2d6c4f: Pull complete 
Digest: sha256:8d31dad0c58f552e890d68bbfb735588b6b820a46e459672d96e585871acc110
Status: Downloaded newer image for ubuntu:latest
51bce5d0f79f6a64477b7d16e062e362191a5ed2f08947cb5c1730198efffdab

```
对于未显示指定name的container,那么如何停止呢？先ls查看运行容器列表，之后可通过name或者container id停止：
```language
[root@localhost ~]# docker container ls
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS               NAMES
51bce5d0f79f        ubuntu:latest       "/bin/bash"              19 minutes ago      Up 19 minutes                           eloquent_hypatia
446ed4bf8972        redis               "docker-entrypoint.s…"   53 minutes ago      Up 53 minutes       6379/tcp            redis3
0ed4fa468c95        redis               "docker-entrypoint.s…"   About an hour ago   Up About an hour    6379/tcp            priceless_chandrasekhar
4a245bacd2ff        redis               "docker-entrypoint.s…"   About an hour ago   Up About an hour    6379/tcp            angry_wilson
[root@localhost ~]# dokcer container stop ubuntu
bash: dokcer: 未找到命令...
相似命令是： 'docker'
[root@localhost ~]# docker container stop ubuntu
Error response from daemon: No such container: ubuntu
[root@localhost ~]# docker container stop 51bce5d0f79f
51bce5d0f79f
```

### 1.2 Docker run 参数详解：
```
命令格式：docker run [OPTIONS] IMAGE [COMMAND] [ARG...]
Usage: Run a command in a new container
中文意思为：通过run命令创建一个新的容器（container）

常用选项说明
-d, --detach=false， 指定容器运行于前台还是后台，默认为false
-i, --interactive=false， 打开STDIN，用于控制台交互
-t, --tty=false， 分配tty设备，该可以支持终端登录，默认为false
-u, --user=""， 指定容器的用户
-a, --attach=[]， 登录容器（必须是以docker run -d启动的容器）
-w, --workdir=""， 指定容器的工作目录
-c, --cpu-shares=0， 设置容器CPU权重，在CPU共享场景使用
-e, --env=[]， 指定环境变量，容器中可以使用该环境变量
-m, --memory=""， 指定容器的内存上限
-P, --publish-all=false， 指定容器暴露的端口
-p, --publish=[]， 指定容器暴露的端口
-h, --hostname=""， 指定容器的主机名
-v, --volume=[]， 给容器挂载存储卷，挂载到容器的某个目录
--volumes-from=[]， 给容器挂载其他容器上的卷，挂载到容器的某个目录
--cap-add=[]， 添加权限，权限清单详见：http://linux.die.net/man/7/capabilities
--cap-drop=[]， 删除权限，权限清单详见：http://linux.die.net/man/7/capabilities
--cidfile=""， 运行容器后，在指定文件中写入容器PID值，一种典型的监控系统用法
--cpuset=""， 设置容器可以使用哪些CPU，此参数可以用来容器独占CPU
--device=[]， 添加主机设备给容器，相当于设备直通
--dns=[]， 指定容器的dns服务器
--dns-search=[]， 指定容器的dns搜索域名，写入到容器的/etc/resolv.conf文件
--entrypoint=""， 覆盖image的入口点
--env-file=[]， 指定环境变量文件，文件格式为每行一个环境变量
--expose=[]， 指定容器暴露的端口，即修改镜像的暴露端口
--link=[]， 指定容器间的关联，使用其他容器的IP、env等信息
--lxc-conf=[]， 指定容器的配置文件，只有在指定--exec-driver=lxc时使用
--name=""， 指定容器名字，后续可以通过名字进行容器管理，links特性需要使用名字
--net="bridge"， 容器网络设置:
bridge 使用docker daemon指定的网桥
host //容器使用主机的网络
container:NAME_or_ID >//使用其他容器的网路，共享IP和PORT等网络资源
none 容器使用自己的网络（类似--net=bridge），但是不进行配置
--privileged=false， 指定容器是否为特权容器，特权容器拥有所有的capabilities
--restart="no"， 指定容器停止后的重启策略:
no：容器退出时不重启
on-failure：容器故障退出（返回值非零）时重启
always：容器退出时总是重启
--rm=false， 指定容器停止后自动删除容器(不支持以docker run -d启动的容器)
--sig-proxy=true， 设置由代理接受并处理信号，但是SIGCHLD、SIGSTOP和SIGKILL不能被代理

```

### 二. docker run -it指令
