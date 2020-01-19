- 参考资料：https://www.sdnlab.com/23388.html
- Docker —— 从入门到实践（强烈推荐）: https://yeasy.gitbooks.io/docker_practice/content/cases/os/alpine.html 或 StudyNote-Resource中的pdf：Docker--从入门到实践.pdf

### 一. 什么是容器？
- 容器提供了在计算机上的隔离环境中安装和运行应用程序的方法。在容器内运行的应用程序仅可使用于为该容器分配的资源，例如：CPU，内存，磁盘，进程空间，用户，网络，卷等。在使用有限的容器资源的同时，并不与其他容器冲突。您可以将容器视为简易计算机上运行应用程序的隔离沙箱。
- 这个概念听起来很熟悉，有些类似于虚拟机。但它们有一个关键的区别：容器使用的一种非常不同的，轻量的技术来实现资源隔离。容器利用了底层 Linux 内核的功能，而不是虚拟机采用的 hypervisor 的方法。换句话说，容器调用 Linux 命令来分配和隔离出一组资源，然后在此空间中运行您的应用程序。我们快速来看下两个这样的功能：

#### Namespaces
- 简单的讲就是，Linux namespace 允许用户在独立进程之间隔离 CPU 等资源。进程的访问权限及可见性仅限于其所在的 Namespaces 。因此，用户无需担心在一个 Namespace 内运行的进程与在另一个 Namespace 内运行的进程冲突。甚至可以同一台机器上的不同容器中运行具有相同 PID 的进程。同样的，两个不同容器中的应用程序可以使用相同的端口。

#### Cgroups
- Cgroups 允许对可用资源设置限制和约束。例如，您可以在一台拥有 16 G 内存的计算机上创建一个 Namespace ，限制其内部进程可用内存为 1 GB。
- 到这，您可能已经猜到 Docker 的工作原理了。当您请求 Docker 运行容器时，Docker 会在您的计算机上设置一个资源隔离的环境。然后 Docker 会将打包的应用程序和关联的文件复制到 Namespace 内的文件系统中，此时环境的配置就完成了。之后 Docker 会执行您指定的命令运行应用程序。
- 简而言之，Docker 通过使用 Linux namespace 和 cgroup（以及其他一些来协调配置容器，将应用程序文件复制到为容器分配的磁盘，然后运行启动命令。Docker 还附带了许多其他用于管理容器的工具，例如：列出正在运行的容器，停止容器，发布容器镜像等许多其他工具。

### 二. Docker&Redis 镜像下载
#### 2.1 镜像下载方法
- 既然上一篇已完成Docker本地环境安装及运行hello-world成功，那么现在就尝试基于Docker运行一个Redis实例。
- Docker Hub 上有大量的高质量的镜像可以用，这里我们就说一下怎 么获取这些镜像。从 Docker 镜像仓库获取镜像的命令是 docker pull 。其命令格式为： 
	docker pull [选项] [Docker Registry 地址[:端口号]/]仓库名[:标签] 
- 具体的选项可以通过 docker pull --help 命令看到，这里我们说一下镜像名称 的格式。
	- Docker 镜像仓库地址：地址的格式一般是 <域名/IP>[:端口号] 。默认地址 是 Docker Hub。 
	- 仓库名：如之前所说，这里的仓库名是两段式名称，即 <用户名>/<软件名> 。 对于 Docker Hub，如果不给出用户名，则默认为 library ，也就是官方镜 像。 
- 比如：
```	
	$ docker pull ubuntu:18.04 
	 18.04: Pulling from library/ubuntu 
	 bf5d46315322: Pull complete 
	 9f13e0ac480c: Pull complete 
	 e8988b5b3097: Pull complete 
	 40af181810e7: Pull complete 
	 e6f7c7e5c03e: Pull complete 
	 Digest: sha256:147913621d9cdea08853f6ba9116c2e27a3ceffecf3b49298 3ae97c3d643fbbe 
	 Status: Downloaded newer image for ubuntu:18.04
```
- 上面的命令中没有给出 Docker 镜像仓库地址，因此将会从 Docker Hub 获取镜 像。而镜像名称是 ubuntu:18.04 ，因此将会获取官方镜像 library/ubuntu 仓库中标签为 18.04 的镜像。 
- 获取镜像 从下载过程中可以看到我们之前提及的分层存储的概念，镜像是由多层存储所构 成。下载也是一层层的去下载，并非单一文件。下载过程中给出了每一层的 ID 的 前 12 位。并且下载结束后，给出该镜像完整的 sha256 的摘要，以确保下载一 致性。 在使用上面命令的时候，你可能会发现，你所看到的层 ID 以及 sha256 的摘要和 这里的不一样。这是因为官方镜像是一直在维护的，有任何新的 bug，或者版本更 新，都会进行修复再以原来的标签发布，这样可以确保任何使用这个标签的用户可 以获得更安全、更稳定的镜像。 如果从 Docker Hub 下载镜像非常缓慢，可以参照 镜像加速器 一节配置加速器。

#### 2.2 下载及管理Redis镜像
- 既然Docker Hub可以获取常用的各种镜像，那么进入Docker Hub官网：https://hub.docker.com/ ，点击页面导航：Browse Popular Images ，即进入https://hub.docker.com/search?q=&type=image。之后搜索进入 redis镜像页面，可查看到镜像下载及运行所需详细资料，如 "docker pull redis"可下载redis镜像laste镜像，抑或可点击"View Available Tags"查看所有可用的镜像tag用于下载指定版本image（如 docker pull redis:alpine3.11）.
- 运行：docker pull redis ，然后还是老问题，要么超时要么下载缓慢不忍直视，参考文档配置镜像加速器之后，杠杠的。
##### 2.2.1 配置镜像加速器
国内从 Docker Hub 拉取镜像有时会遇到困难，此时可以配置镜像加速器。国内很 多云服务商都提供了国内加速器服务，例如：
- Azure 中国镜像 https://dockerhub.azk8s.cn
- 阿里云加速器(需登录账号获取) 
- 网易云加速器 https://hub-mirror.c.163.com 由于镜像服务可能出现宕机，建议同时配置多个镜像。
> 各个镜像站测试结果请 到 docker-practice/docker-registry-cn-mirror-test 查看。 国内各大云服务商均提供了 Docker 镜像加速服务，建议根据运行 Docker 的云 平台选择对应的镜像加速服务，具体请参考官方文档。
本节我们以 Azure 中国镜像 https://dockerhub.azk8s.cn 为例进行介绍。 
###### Ubuntu 16.04+、Debian 8+、CentOS 7 
对于使用 systemd 的系统，请在 /etc/docker/daemon.json 中写入如下内容 （如果文件不存在请新建该文件）
 { "registry-mirrors": [ "https://dockerhub.azk8s.cn", "https://hub-mirror.c.163.com" ] }
> 注意，一定要保证该文件符合 json 规范，否则 Docker 将不能启动。 之后重新启动服务。 
```	
	$ sudo systemctl daemon-reload 
	$ sudo systemctl restart docker 
```
镜像加速器 注意：如果您之前查看旧教程，修改了 docker.service 文件内容，请去掉 您添加的内容（ --registry-mirror=https://dockerhub.azk8s.cn ）。

##### 2.2.1 下载Redis 镜像
###### 下载Redis latest镜像
```
[root@localhost rpm]# docker pull redis
Using default tag: latest
latest: Pulling from library/redis
8ec398bc0356: Pull complete 
da01136793fa: Pull complete 
cf1486a2c0b8: Pull complete 
a44f7da98d9e: Pull complete 
c677fde73875: Pull complete 
727f8da63ac2: Pull complete 
Digest: sha256:90d44d431229683cadd75274e6fcb22c3e0396d149a8f8b7da9925021ee75c30
Status: Downloaded newer image for redis:latest
docker.io/library/redis:latest

```
###### 下载Redis alpine3.11镜像
```language
[root@localhost rpm]# docker pull redis:alpine3.11
alpine3.11: Pulling from library/redis
c9b1b535fdd9: Pull complete 
8dd5e7a0ba4a: Pull complete 
e20c1cdf5aef: Pull complete 
f06a0c1e566e: Pull complete 
230b5c8df708: Pull complete 
0cb9ac88f5bf: Pull complete 
Digest: sha256:cb9783b1c39bb34f8d6572406139ab325c4fac0b28aaa25d5350495637bb2f76
Status: Downloaded newer image for redis:alpine3.11
docker.io/library/redis:alpine3.11
```

#### 2.3 本地镜像管理
```language
[root@localhost rpm]# docker image --help
Usage:	docker image COMMAND
Manage images
Commands:
  build       Build an image from a Dockerfile
  history     Show the history of an image
  import      Import the contents from a tarball to create a filesystem image
  inspect     Display detailed information on one or more images
  load        Load an image from a tar archive or STDIN
  ls          List images
  prune       Remove unused images
  pull        Pull an image or a repository from a registry
  push        Push an image or a repository to a registry
  rm          Remove one or more images
  save        Save one or more images to a tar archive (streamed to STDOUT by default)
  tag         Create a tag TARGET_IMAGE that refers to SOURCE_IMAGE

Run 'docker image COMMAND --help' for more information on a command.
```
##### 2.3.1 查看本地已下载镜像（即列出之前hello-world及redis两个版本的镜像）
```language
[root@localhost rpm]# docker image ls
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
redis               alpine3.11          b68707e68547        26 hours ago        29.8MB
redis               latest              9b188f5fb1e6        2 weeks ago         98.2MB
hello-world         latest              fce289e99eb9        12 months ago       1.84kB

```
##### 2.3.1 删除本地已下载镜像
```language
[root@localhost rpm]# docker image rm redis
Untagged: redis:latest
Untagged: redis@sha256:90d44d431229683cadd75274e6fcb22c3e0396d149a8f8b7da9925021ee75c30
Deleted: sha256:9b188f5fb1e6e1c7b10045585cb386892b2b4e1d31d62e3688c6fa8bf9fd32b5
Deleted: sha256:fe7afb618c11b8be098a10564a9a1682f83915bfdbaaa5af48791950d418b2d5
Deleted: sha256:3a284ce371b3431ba30071057478e2db8fc096232b1a84f092c4df9e06a4a3e4
Deleted: sha256:4396548b331d1b748c8ba1542f8da54e0a8b84102d8205440aac61e3941bdf71
Deleted: sha256:c80de70938af062d3c273f9925641ec672fe182a796bb4a096a37963c92e071a
Deleted: sha256:e807dfe0532b9dae274911841bab81588e9e34591a5b809b8da39471fb75fdbd
Deleted: sha256:556c5fb0d91b726083a8ce42e2faaed99f11bc68d3f70e2c7bbce87e7e0b3e10

```
如上未指定镜像tag标记，则默认删除其latest标记对应的镜像文件，如期需删除指定版本，则：
```language
[root@localhost rpm]# docker image rm redis:alpine3.11
Untagged: redis:alpine3.11
Untagged: redis@sha256:cb9783b1c39bb34f8d6572406139ab325c4fac0b28aaa25d5350495637bb2f76
Deleted: sha256:b68707e68547e636f2544e9283f02beed46d536f644573c8b35c368f9abbe078
Deleted: sha256:acd9269c24b16b344128cf4e650d279ec513a8f780e6c5a8f9a178c65de29e04
Deleted: sha256:db13aece52c4640f7d2604cecd55a5d19062326eb3dbb73954b8cf25850e666f
Deleted: sha256:36fdba8170f9fe2ea6ce2dce4eed3502e925d2f64dd5b0977e3a4219e0563d17
Deleted: sha256:db7132f0597427fa3f6e0cab947fbf58e1a4e9aaa26216c40bdea5fa9772c7fa
Deleted: sha256:0632cca2bb6b87a9f38a5a61b26a4bdcf4c064608516502d6984386712078377
Deleted: sha256:5216338b40a7b96416b8b9858974bbe4acc3096ee60acbc4dfb1ee02aecceb10

```
如上删除镜像成功，可通过docker image ls 确认。

### 三. 操作容器（Redis镜像）
- 可参考Docker Hub上redis页的资料：https://hub.docker.com/_/redis ，基于对镜像下载、启动（各种模式）、关闭、端口映射等非常的详细。上面示例已删除redis镜像文件，此处需执行docker pull redis重新下载redis latest镜像文件。
- 实操之前，先大概了解下docker run支持的参数：
```language
[root@localhost rpm]# docker run  --help

Usage:	docker run [OPTIONS] IMAGE [COMMAND] [ARG...]

Run a command in a new container

Options:
      --add-host list                  Add a custom host-to-IP mapping (host:ip)
  -a, --attach list                    Attach to STDIN, STDOUT or STDERR
      --blkio-weight uint16            Block IO (relative weight), between 10 and 1000, or
                                       0 to disable (default 0)
      --blkio-weight-device list       Block IO weight (relative device weight) (default [])
      --cap-add list                   Add Linux capabilities
      --cap-drop list                  Drop Linux capabilities
      --cgroup-parent string           Optional parent cgroup for the container
      --cidfile string                 Write the container ID to the file
      --cpu-period int                 Limit CPU CFS (Completely Fair Scheduler) period
      --cpu-quota int                  Limit CPU CFS (Completely Fair Scheduler) quota
      --cpu-rt-period int              Limit CPU real-time period in microseconds
      --cpu-rt-runtime int             Limit CPU real-time runtime in microseconds
  -c, --cpu-shares int                 CPU shares (relative weight)
      --cpus decimal                   Number of CPUs
      --cpuset-cpus string             CPUs in which to allow execution (0-3, 0,1)
      --cpuset-mems string             MEMs in which to allow execution (0-3, 0,1)
  -d, --detach                         Run container in background and print container ID
      --detach-keys string             Override the key sequence for detaching a container
      --device list                    Add a host device to the container
      --device-cgroup-rule list        Add a rule to the cgroup allowed devices list
      --device-read-bps list           Limit read rate (bytes per second) from a device
                                       (default [])
      --device-read-iops list          Limit read rate (IO per second) from a device
                                       (default [])
      --device-write-bps list          Limit write rate (bytes per second) to a device
                                       (default [])
      --device-write-iops list         Limit write rate (IO per second) to a device (default [])
      --disable-content-trust          Skip image verification (default true)
      --dns list                       Set custom DNS servers
      --dns-option list                Set DNS options
      --dns-search list                Set custom DNS search domains
      --domainname string              Container NIS domain name
      --entrypoint string              Overwrite the default ENTRYPOINT of the image
  -e, --env list                       Set environment variables
      --env-file list                  Read in a file of environment variables
      --expose list                    Expose a port or a range of ports
      --gpus gpu-request               GPU devices to add to the container ('all' to pass
                                       all GPUs)
      --group-add list                 Add additional groups to join
      --health-cmd string              Command to run to check health
      --health-interval duration       Time between running the check (ms|s|m|h) (default 0s)
      --health-retries int             Consecutive failures needed to report unhealthy
      --health-start-period duration   Start period for the container to initialize before
                                       starting health-retries countdown (ms|s|m|h) (default 0s)
      --health-timeout duration        Maximum time to allow one check to run (ms|s|m|h)
                                       (default 0s)
      --help                           Print usage
  -h, --hostname string                Container host name
      --init                           Run an init inside the container that forwards
                                       signals and reaps processes
  -i, --interactive                    Keep STDIN open even if not attached
      --ip string                      IPv4 address (e.g., 172.30.100.104)
      --ip6 string                     IPv6 address (e.g., 2001:db8::33)
      --ipc string                     IPC mode to use
      --isolation string               Container isolation technology
      --kernel-memory bytes            Kernel memory limit
  -l, --label list                     Set meta data on a container
      --label-file list                Read in a line delimited file of labels
      --link list                      Add link to another container
      --link-local-ip list             Container IPv4/IPv6 link-local addresses
      --log-driver string              Logging driver for the container
      --log-opt list                   Log driver options
      --mac-address string             Container MAC address (e.g., 92:d0:c6:0a:29:33)
  -m, --memory bytes                   Memory limit
      --memory-reservation bytes       Memory soft limit
      --memory-swap bytes              Swap limit equal to memory plus swap: '-1' to enable
                                       unlimited swap
      --memory-swappiness int          Tune container memory swappiness (0 to 100) (default -1)
      --mount mount                    Attach a filesystem mount to the container
      --name string                    Assign a name to the container
      --network network                Connect a container to a network
      --network-alias list             Add network-scoped alias for the container
      --no-healthcheck                 Disable any container-specified HEALTHCHECK
      --oom-kill-disable               Disable OOM Killer
      --oom-score-adj int              Tune host's OOM preferences (-1000 to 1000)
      --pid string                     PID namespace to use
      --pids-limit int                 Tune container pids limit (set -1 for unlimited)
      --privileged                     Give extended privileges to this container
  -p, --publish list                   Publish a container's port(s) to the host
  -P, --publish-all                    Publish all exposed ports to random ports
      --read-only                      Mount the container's root filesystem as read only
      --restart string                 Restart policy to apply when a container exits
                                       (default "no")
      --rm                             Automatically remove the container when it exits
      --runtime string                 Runtime to use for this container
      --security-opt list              Security Options
      --shm-size bytes                 Size of /dev/shm
      --sig-proxy                      Proxy received signals to the process (default true)
      --stop-signal string             Signal to stop a container (default "SIGTERM")
      --stop-timeout int               Timeout (in seconds) to stop a container
      --storage-opt list               Storage driver options for the container
      --sysctl map                     Sysctl options (default map[])
      --tmpfs list                     Mount a tmpfs directory
  -t, --tty                            Allocate a pseudo-TTY
      --ulimit ulimit                  Ulimit options (default [])
  -u, --user string                    Username or UID (format: <name|uid>[:<group|gid>])
      --userns string                  User namespace to use
      --uts string                     UTS namespace to use
  -v, --volume list                    Bind mount a volume
      --volume-driver string           Optional volume driver for the container
      --volumes-from list              Mount volumes from the specified container(s)
  -w, --workdir string                 Working directory inside the container

```

#### 3.1 基于Docker启动一个Redis实例
--name参数应该是指定container名称，此处为redis1,运行：
```language
[root@localhost ~]# docker run --name redis1 redis
1:C 19 Jan 2020 06:53:51.773 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
1:C 19 Jan 2020 06:53:51.773 # Redis version=5.0.7, bits=64, commit=00000000, modified=0, pid=1, just started
1:C 19 Jan 2020 06:53:51.773 # Warning: no config file specified, using the default config. In order to specify a config file use redis-server /path/to/redis.conf
1:M 19 Jan 2020 06:53:51.790 * Running mode=standalone, port=6379.
1:M 19 Jan 2020 06:53:51.790 # WARNING: The TCP backlog setting of 511 cannot be enforced because /proc/sys/net/core/somaxconn is set to the lower value of 128.
1:M 19 Jan 2020 06:53:51.790 # Server initialized
1:M 19 Jan 2020 06:53:51.790 # WARNING overcommit_memory is set to 0! Background save may fail under low memory condition. To fix this issue add 'vm.overcommit_memory = 1' to /etc/sysctl.conf and then reboot or run the command 'sysctl vm.overcommit_memory=1' for this to take effect.
1:M 19 Jan 2020 06:53:51.790 # WARNING you have Transparent Huge Pages (THP) support enabled in your kernel. This will create latency and memory usage issues with Redis. To fix this issue run the command 'echo never > /sys/kernel/mm/transparent_hugepage/enabled' as root, and add it to your /etc/rc.local in order to retain the setting after a reboot. Redis must be restarted after THP is disabled.
1:M 19 Jan 2020 06:53:51.790 * Ready to accept connections

```
Terminal终端CTRL+C退出后，再次运行，可发现redis1再次运行无法使用同一名称,因已被使用。
```language
[root@localhost ~]# docker run --name redis1 redis
docker: Error response from daemon: Conflict. The container name "/redis1" is already in use by container "084bd37d255cb8b167816a32058c763edd8f4661781dc45e3d65a7a4c47f6526". You have to remove (or rename) that container to be able to reuse that name.
See 'docker run --help'.

```
纳尼？无法启动同名容器，咋整？如若想继续使用 redis1 容器又该如何？
#### 3.2 启动多个Redis容器
docker run --name redis1 redis 中的 --name 参数可指定容器名称，而同一镜像的容器名称具有唯一性。那想要运行多个Redis容器实例，目前看可行的是每次运行指定不没看过的name即可；而如若不指定name又如何呢？分别在两个Terminal终端启动Redis容器：
```
[root@localhost ~]# docker run  redis
1:C 19 Jan 2020 07:02:35.273 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
1:C 19 Jan 2020 07:02:35.273 # Redis version=5.0.7, bits=64, commit=00000000, modified=0, pid=1, just started
1:C 19 Jan 2020 07:02:35.273 # Warning: no config file specified, using the default config. In order to specify a config file use redis-server /path/to/redis.conf
1:M 19 Jan 2020 07:02:35.275 * Running mode=standalone, port=6379.
1:M 19 Jan 2020 07:02:35.275 # WARNING: The TCP backlog setting of 511 cannot be enforced because /proc/sys/net/core/somaxconn is set to the lower value of 128.
1:M 19 Jan 2020 07:02:35.275 # Server initialized
1:M 19 Jan 2020 07:02:35.275 # WARNING overcommit_memory is set to 0! Background save may fail under low memory condition. To fix this issue add 'vm.overcommit_memory = 1' to /etc/sysctl.conf and then reboot or run the command 'sysctl vm.overcommit_memory=1' for this to take effect.
1:M 19 Jan 2020 07:02:35.275 # WARNING you have Transparent Huge Pages (THP) support enabled in your kernel. This will create latency and memory usage issues with Redis. To fix this issue run the command 'echo never > /sys/kernel/mm/transparent_hugepage/enabled' as root, and add it to your /etc/rc.local in order to retain the setting after a reboot. Redis must be restarted after THP is disabled.
1:M 19 Jan 2020 07:02:35.275 * Ready to accept connections


```
```language
[root@localhost ~]# docker run redis
1:C 19 Jan 2020 07:06:46.154 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
1:C 19 Jan 2020 07:06:46.154 # Redis version=5.0.7, bits=64, commit=00000000, modified=0, pid=1, just started
1:C 19 Jan 2020 07:06:46.154 # Warning: no config file specified, using the default config. In order to specify a config file use redis-server /path/to/redis.conf
1:M 19 Jan 2020 07:06:46.170 * Running mode=standalone, port=6379.
1:M 19 Jan 2020 07:06:46.171 # WARNING: The TCP backlog setting of 511 cannot be enforced because /proc/sys/net/core/somaxconn is set to the lower value of 128.
1:M 19 Jan 2020 07:06:46.171 # Server initialized
1:M 19 Jan 2020 07:06:46.171 # WARNING overcommit_memory is set to 0! Background save may fail under low memory condition. To fix this issue add 'vm.overcommit_memory = 1' to /etc/sysctl.conf and then reboot or run the command 'sysctl vm.overcommit_memory=1' for this to take effect.
1:M 19 Jan 2020 07:06:46.171 # WARNING you have Transparent Huge Pages (THP) support enabled in your kernel. This will create latency and memory usage issues with Redis. To fix this issue run the command 'echo never > /sys/kernel/mm/transparent_hugepage/enabled' as root, and add it to your /etc/rc.local in order to retain the setting after a reboot. Redis must be restarted after THP is disabled.
1:M 19 Jan 2020 07:06:46.171 * Ready to accept connections

```

同时基于--name启动 redis2 实例。最后再打开一个Terminal终端，运行docker container ls 可查看运行的容器（最后一列即为容器name）：
```
[root@localhost ~]# docker container ls
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS               NAMES
373aa0f57dd7        redis               "docker-entrypoint.s…"   15 seconds ago      Up 12 seconds       6379/tcp            redis2
0ed4fa468c95        redis               "docker-entrypoint.s…"   4 minutes ago       Up 4 minutes        6379/tcp            priceless_chandrasekhar
4a245bacd2ff        redis               "docker-entrypoint.s…"   8 minutes ago       Up 8 minutes        6379/tcp            angry_wilson

```

#### 3.3 容器启动高级应用
- 启动容器有两种方式，一种是基于镜像新建一个容器并启动，另外一个是将在终止 状态（ stopped ）的容器重新启动。 因为 Docker 的容器实在太轻量级了，很多时候用户都是随时删除和新创建容器。而上面即是新建启动，而容器该重新启动呢？
那么首先需要知道可重启启动的容器名称标识，而上面的 docker conatiner ls仅能列出运行态的容器。那么结合 docker container --help 及 docker container ls --help  则可知道如下即可查看所有状态的container容器：
```
[root@localhost ~]# docker container ls  -a
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS                      PORTS               NAMES
373aa0f57dd7        redis               "docker-entrypoint.s…"   6 minutes ago       Exited (0) 2 minutes ago                        redis2
0ed4fa468c95        redis               "docker-entrypoint.s…"   10 minutes ago      Up 10 minutes               6379/tcp            priceless_chandrasekhar
4a245bacd2ff        redis               "docker-entrypoint.s…"   14 minutes ago      Up 14 minutes               6379/tcp            angry_wilson
084bd37d255c        redis               "docker-entrypoint.s…"   23 minutes ago      Exited (0) 20 minutes ago                       redis1
8d8ee9652e3f        hello-world         "/hello"                 4 hours ago         Exited (0) 4 hours ago                          goofy_burnell
 
```
其中第一列CONTAINER ID为容器IDeas，第二列IMAGE即为容器对应镜像名称，第三列COMMAND待定，第四列CREATED为容器创建大致时间，第五列STATUS则为容器状态（如Exit为退出，而Up则为运行状态），第六列PORTS则为容器运行时Redis实例对应内部端口，第七列NAMES为容器名称。

