1. jdk7+、Ant安装并设置JAVA_HOME、ANT_HOME变量
2. 下载安装Meurial
3. 下载openjdk，并执行get_soruce.sh
4. yum安装gcc
5. 找不到freeType ， 可确认已安装； 若无法找到，则用find / -name freetype 查找
6. ALSA_VERSION处理   
   ERROR: You seem to not have installed ALSA 0.9.1 or higher.
       Please install ALSA (drivers and lib). You can download the
       source distribution from http://www.alsa-project.org or go to
       http://www.freshrpms.net/docs/alsa/ for precompiled RPM packages.
      ---查看openjdk 包中的readme_build.html文件：Both alsa and alsa-devel packages are needed.
       确认是未安装alsa-lib-devel，yum 即可。
6、CUPS问题
ERROR: You do not have access to valid Cups header files.
       Please check your access to
           /usr/include/cups/cups.h
       and/or check your value of ALT_CUPS_HEADERS_PATH,
       CUPS is frequently pre-installed on many systems,
       or may be downloaded from http://www.cups.org
    yum install cups-devel
7、编译shell如下：   
       #!/bin/bash
        export ALT_BOOTDIR=/usr/local/java8
export LANG=C
export ALLOW_DOWNLOADS=true
export ALT_JDK_IMPORT_PATH=/home/zyl/study/openjdk/target
export ALT_FREETYPE_HEADERS_PATH=/usr/local/freetype/include
export ALT_FREETYPE_LIB_PATH=/usr/local/freetype/lib

unset JAVA_HOME
unset CLASSPATH
make sanity
-------------------------------------------------------------------------
1.	#语言选项，这个必须设置，否则编译好后会出现一个HashTable的NPE错  
2.	export LANG=C  
3.	  
4.	#Bootstrap JDK的安装路径。必须设置。   
5.	export ALT_BOOTDIR=/root/java/jdk/  
6.	export ALT_JDK_IMPORT_PATH=/root/java/jdk/  
7.	  
8.	#允许自动下载依赖  
9.	export ALLOW_DOWNLOADS=true  
10.	  
11.	#并行编译的线程数，设置为和CPU内核数量一致即可  
12.	export HOTSPOT_BUILD_JOBS=4  
13.	export ALT_PARALLEL_COMPILE_JOBS=4  
14.	  
15.	#比较本次build出来的映像与先前版本的差异。这个对我们来说没有意义，必须设置为false，否则sanity检查会报缺少先前版本JDK的映像。如果有设置dev或者DEV_ONLY=true的话这个不显式设置也行。   
16.	export SKIP_COMPARE_IMAGES=true  
17.	  
18.	#使用预编译头文件，不加这个编译会更慢一些  
19.	export USE_PRECOMPILED_HEADER=true  
20.	  
21.	#要编译的内容  
22.	export BUILD_LANGTOOLS=true   
23.	#export BUILD_JAXP=false  
24.	export BUILD_JAXWS=false   
25.	#export BUILD_CORBA=false  
26.	export BUILD_HOTSPOT=true   
27.	export BUILD_JDK=true  
28.	export DISABLE_HOTSPOT_OS_VERSION_CHECK=ok  
29.	  
30.	  
31.	#要编译的版本  
32.	#export SKIP_DEBUG_BUILD=false  
33.	#export SKIP_FASTDEBUG_BUILD=true  
34.	#export DEBUG_NAME=debug  
35.	  
36.	#把它设置为false可以避开javaws和浏览器Java插件之类的部分的build。   
37.	BUILD_DEPLOY=false  
38.	  
39.	#把它设置为false就不会build出安装包。因为安装包里有些奇怪的依赖，但即便不build出它也已经能得到完整的JDK映像，所以还是别build它好了。  
40.	BUILD_INSTALL=false  
41.	  
42.	#编译结果所存放的路径  
43.	#export ALT_OUTPUTDIR=/Users/IcyFenix/Develop/JVM/jdkBuild/openjdk_7u4/build  
44.	  
45.	#这两个环境变量必须去掉，不然会有很诡异的事情发生（我没有具体查过这些“”诡异的事情”，Makefile脚本检查到有这2个变量就会提示警告“）  
46.	unset JAVA_HOME  
47.	unset CLASSPATH 
--------------------------------------------------------------------------------------------------------------
8、编译成功，结果如下：
    Sanity check passed.
9、开始编译--参考（make all ARCH_DATA_MODEL=64 ALLOW_DOWNLOADS=true）
--bug清单，编译问题可在这儿查找：https://bugs.openjdk.java.net/secure/RapidBoard.jspa?projectKey=JDK&useStoredSettings=true&rapidView=9
10、 openJDK9 编译较简单，./configure  失败日志已提示失败信赖包，如：
        yum install gcc-c++.i686
        yum install libXtst-devel libXt-devel libXrender-devel libXi-devel
        yum install freetype-devel
        之后执行 make即可。

---额，后面其实各种失败。。。。
