参考资料：Docker教程： https://www.runoob.com/docker/centos-docker-install.html

### 一. 安装准备及环境检查
#### 1. Docker 支持以下的 64 位 CentOS 版本：CentOS 7CentOS 8 或 更高版本...
 centos-extras 库必须启用。默认情况下，此仓库是启用的，但是如果已禁用它，则需要重新启用它。建议使用 overlay2 存储驱动程序。

#### 2. 卸载旧版本
- 较旧的 Docker 版本称为 docker 或 docker-engine 。如果已安装这些程序，请卸载它们以及相关的依赖项。因是基于Vmware全新安装的Centos8虚拟机，不太确定是否已有安装docker。
- yum list  //列出所有可安装(包含已安装)的软件包。那如何查看已安装的包列表呢？可通过yum list --help 查看list支持的二级参数，故可发现  $yum list installed //列出所有已安装的软件包。
```language
[root@localhost ~]# yum list --installed | grep docker
pcp-pmda-docker.x86_64                           4.3.0-3.el8                                            @AppStream

```
不太明白  pcp-pmda-docker作用，百度发现其还有类似 pcp-pmda-ngix，貌似用于docker容器运行监控，暂时忽略，初步来看当前环境应该没有安装docker。

#### 3. 安装 Docker Engine-Community
在新主机上首次安装 Docker Engine-Community 之前，需要设置 Docker 仓库。之后，您可以从仓库安装和更新 Docker。
- 设置仓库
安装所需的软件包。yum-utils 提供了 yum-config-manager ，并且 device mapper 存储驱动程序需要 device-mapper-persistent-data 和 lvm2。
```language
yum install -y yum-utils   device-mapper-persistent-data  lvm2
上次元数据过期检查：5:45:29 前，执行于 2020年01月17日 星期五 10时14分32秒。
Package device-mapper-persistent-data-0.7.6-1.el8.x86_64 is already installed.
Package lvm2-8:2.03.02-6.el8.x86_64 is already installed.

```
