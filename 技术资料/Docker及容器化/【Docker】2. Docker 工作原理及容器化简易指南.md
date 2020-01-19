- 参考资料：https://www.sdnlab.com/23388.html
- Docker —— 从入门到实践: https://yeasy.gitbooks.io/docker_practice/content/cases/os/alpine.html

### 一. 什么是容器？
- 容器提供了在计算机上的隔离环境中安装和运行应用程序的方法。在容器内运行的应用程序仅可使用于为该容器分配的资源，例如：CPU，内存，磁盘，进程空间，用户，网络，卷等。在使用有限的容器资源的同时，并不与其他容器冲突。您可以将容器视为简易计算机上运行应用程序的隔离沙箱。
- 这个概念听起来很熟悉，有些类似于虚拟机。但它们有一个关键的区别：容器使用的一种非常不同的，轻量的技术来实现资源隔离。容器利用了底层 Linux 内核的功能，而不是虚拟机采用的 hypervisor 的方法。换句话说，容器调用 Linux 命令来分配和隔离出一组资源，然后在此空间中运行您的应用程序。我们快速来看下两个这样的功能：

#### Namespaces
- 简单的讲就是，Linux namespace 允许用户在独立进程之间隔离 CPU 等资源。进程的访问权限及可见性仅限于其所在的 Namespaces 。因此，用户无需担心在一个 Namespace 内运行的进程与在另一个 Namespace 内运行的进程冲突。甚至可以同一台机器上的不同容器中运行具有相同 PID 的进程。同样的，两个不同容器中的应用程序可以使用相同的端口。

#### Cgroups
- Cgroups 允许对可用资源设置限制和约束。例如，您可以在一台拥有 16 G 内存的计算机上创建一个 Namespace ，限制其内部进程可用内存为 1 GB。
- 到这，您可能已经猜到 Docker 的工作原理了。当您请求 Docker 运行容器时，Docker 会在您的计算机上设置一个资源隔离的环境。然后 Docker 会将打包的应用程序和关联的文件复制到 Namespace 内的文件系统中，此时环境的配置就完成了。之后 Docker 会执行您指定的命令运行应用程序。
- 简而言之，Docker 通过使用 Linux namespace 和 cgroup（以及其他一些来协调配置容器，将应用程序文件复制到为容器分配的磁盘，然后运行启动命令。Docker 还附带了许多其他用于管理容器的工具，例如：列出正在运行的容器，停止容器，发布容器镜像等许多其他工具。

### 二. Docker&Redis
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

#### 2.2 下载Redis镜像
既然Docker Hub可以获取常用的各种镜像，那么进入Docker Hub官网：https://hub.docker.com/ ，点击页面导航：Browse Popular Images ，即进入https://hub.docker.com/search?q=&type=image。之后搜索进入 redis镜像页面，可查看到镜像下载及运行所需详细资料，如 "docker pull redis"可下载redis镜像
