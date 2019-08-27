近期基于业务需要，正好在对交易进行性能测试，其中交易研发同学根据服务调用日志反馈记账接口偶尔处理时长超过200ms，于是此基于此，决定使用Arthas进行定位。

### 一.Linux Arthas安装及验证
根据Arthas官网（https://alibaba.github.io/arthas/index.html），选择全量安装模式下载arthas-packaging-3.1.1-bin.7z 并解压缩；以cmd模式进入对应目录，根据文档使用arthas-boot.jar，直接用java -jar的方式启动：