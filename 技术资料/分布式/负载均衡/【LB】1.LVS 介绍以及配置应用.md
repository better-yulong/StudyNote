出自：https://www.cnblogs.com/yangliheng/p/5692068.html
### 1、负载均衡集群介绍
#### 1.1、什么是负载均衡集群
负载均衡集群提供了一种廉价、有效、透明的方法，来扩展网络设备和服务器的负载、带宽、增加吞吐量、加强网络数据的处理能力、提高网络的灵活性和可用性
- 搭建负载均衡器的需求：
1. 把单台计算机无法承受的大规模的并发访问或数据流量分担到多台节点设备上分别处理，减少用户等待时间，提升用户体验
2. 单个重负载的运算分担到多台节点设备上做并行处理，每个节点设备处理结束后，将结果汇总，返回给用户，系统处理能力得到大幅度的提高。
3. 7*24的服务保证，任意一个或多个有限后面节点设备宕机，要求不能影响业务。
- 在负载均衡器集群中，所有的计算节点都应该提供相同的服务。集群负载均衡获取所有对该服务的入站要求，然后将这些请求尽可能的平均分配在所有集群节点上。
#### 1.2、常见的负载均衡器
- a 根据工作在的协议层划分可划分为：
1. 四层负载均衡（位于内核层）：根据请求报文中的目标地址和端口进行调度
2. 七层负载均衡（位于应用层）：根据请求报文的内容进行调度，这种调度属于「代理」的方式
- b 根据软硬件划分：
- 硬件负载均衡：
1. F5 的 BIG-IP
2. Citrix 的 NetScaler
3. 这类硬件负载均衡器通常能同时提供四层和七层负载均衡，但同时也价格不菲
- 软件负载均衡：
1. TCP 层：LVS，HaProxy，Nginx
2. 基于 HTTP 协议：Haproxy，Nginx，ATS（Apache Traffic Server），squid，varnish
3. 基于 MySQL 协议：mysql-proxy

### 2、LVS（Linux Virtual Server）介绍
- Internet的快速增长使多媒体网络服务器面对的访问数量快速增加，服务器需要具备提供大量并发访问服务的能力，因此对于大负载的服务器来讲， CPU、I/O处理能力很快会成为瓶颈。由于单台服务器的性能总是有限的，简单的提高硬件性能并不能真正解决这个问题。为此，必须采用多服务器和负载均衡技术才能满足大量并发访问的需要。Linux 虚拟服务器(Linux Virtual Servers,LVS) 使用负载均衡技术将多台服务器组成一个虚拟服务器。它为适应快速增长的网络访问需求提供了一个负载能力易于扩展，而价格低廉的解决方案。
- LVS (Linux Virtual Server)其实是一种集群(Cluster)技术，采用IP负载均衡技术（LVS 的 IP 负载均衡技术是通过 IPVS 模块来实现的，linux内核2.6版本以上是默认安装IPVS的）和基于内容请求分发技术。调度器具有很好的吞吐率，将请求均衡地转移到不同的服务器上执行，且调度器自动屏蔽掉服务器的故障，从而将一组服务器构成一个高性能的、高可用的虚拟服务器。整个服务器集群的结构对客户是透明的，而且无需修改客户端和服务器端的程序。
![LVS-01](https://github.com/better-yulong/StudyNote-Resource/blob/master/StudyNote-Resource/tech/LB/LB-1-01.png)
LVS负载均衡调度技术是在LINUX内核中实现的，因此被称之为LINUX虚拟服务器。我们使用该软件配置LVS时候，不能直接配置内核中的IPVS，而需要使用IPVS的管理工具ipvsadm进行管理，当然我们也可以通过keepalived软件直接管理IPVS，并不是通过ipvsadm来管理ipvs。
- LVS项目介绍 http://www.linuxvirtualserver.org/zh/lvs1.html 
- LVS集群的体系结构 http://www.linuxvirtualserver.org/zh/lvs2.html 
- LVS集群中的IP负载均衡技术 http://www.linuxvirtualserver.org/zh/lvs3.html
- LVS集群的负载调度 http://www.linuxvirtualserver.org/zh/lvs4.html 