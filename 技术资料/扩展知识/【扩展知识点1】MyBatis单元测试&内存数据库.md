- 研究MyBatis源码时，
Apache Derby是一个完全用java编写的数据库，Derby是一个Open source的产品，基于Apache License 2.0分发。
- Apache Derby非常小巧，核心部分derby.jar只有2M，所以既可以做为单独的数据库服务器使用，也可以内嵌在应用程序中使用。Cognos 8 BI的Content Store默认就是使用的Derby数据库，可以在Cognos8的安装目录下看到一个叫derby10.1.2.1的目录，就是内嵌的10.1.2.1 版本的derby。
- Derby数据库有两种运行模式：
1） 内嵌模式。Derby数据库与应用程序共享同一个JVM，通常由应用程序负责启动和停止，对除启动它的应用程序外的其它应用程序不可见，即其它应用程序不可访问它；
2） 网络模式。Derby数据库独占一个JVM，做为服务器上的一个独立进程运行。在这种模式下，允许有多个应用程序来访问同一个Derby数据库。
- ..........