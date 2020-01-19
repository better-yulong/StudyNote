
### 启动ubuntu 容器
### 1.1 启动ubuntu容器
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

### 二. docker run 示例
- 为容器指定一个名字：docker run -d --name=ubuntu_server ubuntu:latest
- 容器暴露80端口，并指定宿主机80端口与其通信(: 之前是宿主机端口，之后是容器需暴露的端口)：docker run -d --name=ubuntu_server -p 80:80 ubuntu:latest
- 指定容器内目录与宿主机目录共享(: 之前是宿主机文件夹，之后是容器需共享的文件夹)：docker run -d --name=ubuntu_server -v /etc/www:/var/www ubuntu:latest

#### 2.1 启动Nginx实例并实现端口映射
##### 2.1.1 默认方式启动nginx容器
docker run nginx
```language
[root@localhost ~]# docker container ls
CONTAINER ID        IMAGE               COMMAND                  CREATED              STATUS              PORTS               NAMES
1ab3fb2220d0        nginx               "nginx -g 'daemon of…"   About a minute ago   Up About a minute   80/tcp              quirky_jennings
[root@localhost ~]# curl http://localhost:80
curl: (7) Failed to connect to localhost port 80: 拒绝连接
[root@localhost ~]# ps -ef|grep nginx
root      48734  44122  0 16:31 pts/3    00:00:00 docker run nginx
root      48762  48747  0 16:31 ?        00:00:00 nginx: master process nginx -g daemon off;
101       48800  48762  0 16:31 ?        00:00:00 nginx: worker process
root      48900  40611  0 16:33 pts/0    00:00:00 grep --color=auto nginx
[root@localhost ~]# 

```
可发现nginx实例已正常运行，通过docker container ls可发现容器运行，Nginx实例使用默认端口80接受http请求，但是使用curl http://localhost:80 却无法正常访问。原因系docker容器运行时会基于进程完成资源隔离，而此处的80端口也仅为容器内部端口；如若需在宿主机访问容器内部资源，则需使用各种方式将容器内资源暴露至宿主机，如上面的-v可实现文件资源共享；而 -p 命令则可以类似于NAT实现端口映射，即将nginx容器内部80端口映射至宿主机80端口：
```
[root@localhost ~]# docker run -d -p 80:80 nginx
73fac46b4fabfd637b3fe5955f9b21fabf0c80542dac3572b66c9aa099ee3912

```
容器状态及验证：
```language
[root@localhost ~]# docker container ls
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                NAMES
73fac46b4fab        nginx               "nginx -g 'daemon of…"   36 seconds ago      Up 34 seconds       0.0.0.0:80->80/tcp   thirsty_matsumoto
[root@localhost ~]# ps -ef|grep nginx
root      49007  48991  0 16:34 ?        00:00:00 nginx: master process nginx -g daemon off;
101       49041  49007  0 16:34 ?        00:00:00 nginx: worker process
root      49065  40611  0 16:34 pts/0    00:00:00 grep --color=auto nginx
[root@localhost ~]# curl http://localhost:80
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
[root@localhost ~]# 

```
同时在宿主机使用firefox访问：http://localhost:80 也可正常打开非常熟悉的nginx默认主页。

### 三. docker run 示例