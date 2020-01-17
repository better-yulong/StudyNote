参考资料：Docker教程： https://www.runoob.com/docker/centos-docker-install.html

- Docker 支持以下的 64 位 CentOS 版本：CentOS 7CentOS 8 或 更高版本...
 centos-extras 库必须启用。默认情况下，此仓库是启用的，但是如果已禁用它，则需要重新启用它。建议使用 overlay2 存储驱动程序。

-卸载旧版本

较旧的 Docker 版本称为 docker 或 docker-engine 。如果已安装这些程序，请卸载它们以及相关的依赖项。


yum list  //列出所有可安装(包含已安装)的软件包
那如何查看已安装的包列表呢？可通过yum list --help 查看list支持的二级参数，故可发现  $yum list installed //列出所有已安装的软件包 