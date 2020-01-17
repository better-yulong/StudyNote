参考资料：Docker教程： https://www.runoob.com/docker/centos-docker-install.html

### 一. 安装准备及环境检查
#### 1. Docker 支持以下的 64 位 CentOS 版本：CentOS 7CentOS 8 或 更高版本...（我本地为基于Vmware的CentOS8）
- centos-extras 库必须启用。默认情况下，此仓库是启用的，但是如果已禁用它，则需要重新启用它。建议使用 overlay2 存储驱动程序。
- 原本可用  lsb_release -a查看当前系统版本信息，却发现到CentOS8中该命令默认未安装。于是通过 cat /etc/redhat-release 查看系统版本信息为：CentOS Linux release 8.0.1905 (Core)。同时也可安装lsb_release 支持。

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
##### 3.1 安装所需的软件包
安装所需的软件包。yum-utils 提供了 yum-config-manager ，并且 device mapper 存储驱动程序需要 device-mapper-persistent-data 和 lvm2。
```language
	[root@localhost ~]# yum install -y yum-utils   device-mapper-persistent-data  lvm2
	上次元数据过期检查：5:45:29 前，执行于 2020年01月17日 星期五 10时14分32秒。
	Package device-mapper-persistent-data-0.7.6-1.el8.x86_64 is already installed.
	Package lvm2-8:2.03.02-6.el8.x86_64 is already installed.
        yum-utils 安装出现问题: 
  package python3-dnf-plugins-core-4.0.8-3.el8.noarch requires python3-hawkey >= 0.34.0, but none of the providers can be installed
  package python3-hawkey-0.35.1-8.el8.x86_64 requires libsolv.so.1()(64bit), but none of the providers can be installed
  package python3-hawkey-0.35.1-8.el8.x86_64 requires libsolvext.so.1()(64bit), but none of the providers can be installed
  package python3-hawkey-0.35.1-8.el8.x86_64 requires libsolv.so.1(SOLV_1.0)(64bit), but none of the providers can be installed
  package python3-hawkey-0.35.1-8.el8.x86_64 requires libsolvext.so.1(SOLV_1.0)(64bit), but none of the providers can be installed
  package python3-hawkey-0.35.1-9.el8_1.x86_64 requires libsolv.so.1()(64bit), but none of the providers can be installed
  package python3-hawkey-0.35.1-9.el8_1.x86_64 requires libsolvext.so.1()(64bit), but none of the providers can be installed
  package python3-hawkey-0.35.1-9.el8_1.x86_64 requires libsolv.so.1(SOLV_1.0)(64bit), but none of the providers can be installed
  package python3-hawkey-0.35.1-9.el8_1.x86_64 requires libsolvext.so.1(SOLV_1.0)(64bit), but none of the providers can be installed
  cannot install both libsolv-0.7.4-3.el8.x86_64 and libsolv-0.6.35-6.el8.x86_64  -即此处版本冲突
  problem with installed package rpm-ostree-libs-2018.8-2.el8.0.1.x86_64
  package rpm-ostree-libs-2018.8-2.el8.0.1.x86_64 requires libsolv.so.0()(64bit), but none of the providers can be installed
  package rpm-ostree-libs-2018.8-2.el8.0.1.x86_64 requires libsolv.so.0(SOLV_1.0)(64bit), but none of the providers can be installed
  package dnf-plugins-core-4.0.8-3.el8.noarch requires python3-dnf-plugins-core = 4.0.8-3.el8, but none of the providers can be installed
  package yum-utils-4.0.8-3.el8.noarch requires dnf-plugins-core = 4.0.8-3.el8, but none of the providers can be installed
  conflicting requests

```
- 从报错提示来看，yum-utils 安装时与python版本可能存在冲突，百度很久暂无法解决，于是百度yum-utils包并查看其环境依赖，发现：Requires  python(abi) = 2.7（来源http://rpmfind.net/linux/RPM/centos/7.7.1908/x86_64/Packages/yum-utils-1.1.31-52.el7.noarch.html；另外CentOS7 默认安装为Python2.7，而到了CentOS7则将CentOS版本升级到了3.6，可通过python -versio查看），想尝鲜CentOS8，结果这块儿又能坑，不过幸好Python2和Python3完整不兼容，且也可以共存，于是乎决定安装python2:
> 多次执行 sudo dnf install python2 终于安装python2.7成功（先尝试yum install python2多次报超时；故切换成dnf命令安装 ，另外默认安装的python版本即为2.7.3，也不用担心python2安装成其他低版本）
- 多次yum install  yum-utils，也多次报Timeout，可能也因为依赖国外镜像网络不稳定相关。切换成 dnf install yum-utils 瞬间就OK了。

##### 3.2 设置仓库 
使用以下命令来设置稳定的仓库。
```
[root@localhost ~]# yum-config-manager --add-repo  https://download.docker.com/linux/centos/docker-ce.repo
添加仓库自：https://download.docker.com/linux/centos/docker-ce.repo
```
##### 3.3 安装 Docker Engine-Community
```language
[root@localhost ~]# dnf install docker-ce docker-ce-cli containerd.io
上次元数据过期检查：0:01:34 前，执行于 2020年01月17日 星期五 17时31分56秒。
错误：
 问题: package docker-ce-3:19.03.5-3.el7.x86_64 requires containerd.io >= 1.2.2-3, but none of the providers can be installed
  - cannot install the best candidate for the job
  - package containerd.io-1.2.10-3.2.el7.x86_64 is excluded
  - package containerd.io-1.2.2-3.3.el7.x86_64 is excluded
  - package containerd.io-1.2.2-3.el7.x86_64 is excluded
  - package containerd.io-1.2.4-3.1.el7.x86_64 is excluded
  - package containerd.io-1.2.5-3.1.el7.x86_64 is excluded
  - package containerd.io-1.2.6-3.3.el7.x86_64 is excluded
(尝试添加 '--skip-broken' 来跳过无法安装的软件包 或 '--nobest' 来不只使用最佳选择的软件包)
```
而通过发现默认适配最优的版本是1.2.0
```language
[root@localhost ~]# yum list|grep containerd.io
containerd.io.x86_64                                 1.2.0-3.el7 
```
- 各种尝试 yum install containerd.io-1.2.2-3.el7.x86_64或者及上面提示的dnf --nobest均无解，无奈决定手动下载 containerd.io-1.2.2-3.el7.x86_64.rpm 安装（题外话，下载因网络问题各种艰辛，尝试了不下二十次终于成功：https://download.docker.com/linux/centos/7/x86_64/stable/Packages/）
- 下载后安装 containerd.io:
```
[root@localhost rpm]# rpm -ihv containerd.io-1.2.2-3.el7.x86_64.rpm 
警告：containerd.io-1.2.2-3.el7.x86_64.rpm: 头V4 RSA/SHA512 Signature, 密钥 ID 621e9f35: NOKEY
错误：依赖检测失败：
	runc 与 containerd.io-1.2.2-3.el7.x86_64 冲突
	runc 被 containerd.io-1.2.2-3.el7.x86_64 取代

```
之后参考https://www.cnblogs.com/zjz20/p/11715437.html，卸载runc：
```[root@localhost ~]# yum erase runc  （erase等价与remove，删除runc）

```
卸载成功后，再次安装containerd.io终于成功：
```language
root@localhost rpm]# rpm -ihv containerd.io-1.2.2-3.el7.x86_64.rpm 
警告：containerd.io-1.2.2-3.el7.x86_64.rpm: 头V4 RSA/SHA512 Signature, 密钥 ID 621e9f35: NOKEY
Verifying...                          ################################# [100%]
准备中...                          ################################# [100%]
正在升级/安装...
   1:containerd.io-1.2.2-3.el7        ################################# [100%]

```



