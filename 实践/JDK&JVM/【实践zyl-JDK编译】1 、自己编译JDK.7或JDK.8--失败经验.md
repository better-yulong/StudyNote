>背景：自己编译JDK，环境：Fedora 系统。
- 可参考资料：https://www.cnblogs.com/zyx1314/p/5638596.html
            https://blog.csdn.net/lr222584/article/details/54882012
            http://www.bruceleeforever.org/#source-drops-download
            
            http://www.oracle.com/technetwork/java/javase/archive-139210.html
            
#### 一、检测环境信息：操作系统位数、磁盘容量、Bootstrap JDK 版本、内存
```[zyl@localhost ~]$ uname -a
Linux localhost.localdomain 4.8.6-300.fc25.x86_64 #1 SMP Tue Nov 1 12:36:38 UTC 2016 x86_64 x86_64 x86_64 GNU/Linux

zyl@localhost openjdk]$ cat /etc/fedora-release
Fedora release 25 (Twenty Five)

[zyl@localhost ~]$ df -h
文件系统                 容量  已用  可用 已用% 挂载点
devtmpfs                 1.5G     0  1.5G    0% /dev
tmpfs                    1.5G  332K  1.5G    1% /dev/shm
tmpfs                    1.5G  2.2M  1.5G    1% /run
tmpfs                    1.5G     0  1.5G    0% /sys/fs/cgroup
/dev/mapper/fedora-root   37G   12G   24G   33% /
tmpfs                    1.5G  152K  1.5G    1% /tmp
/dev/sda1                976M  114M  796M   13% /boot
tmpfs                    292M  2.3M  290M    1% /run/user/42
tmpfs                    292M  4.6M  288M    2% /run/user/1000
/dev/sr0                 1.4G  1.4G     0  100% /run/media/zyl/Fedora-WS-Live-25-1-3

[zyl@localhost ~]$ java -version
openjdk version "1.8.0_111"
OpenJDK Runtime Environment (build 1.8.0_111-b16)
OpenJDK 64-Bit Server VM (build 25.111-b16, mixed mode)

[zyl@localhost ~]$ free -h
              total        used        free      shared  buff/cache   available
Mem:           2.8G        1.3G        535M         14M        1.1G        1.4G
Swap:          2.0G          0B        2.0G

```

#### 二、检测编译工具
JDK由各个组成部分（hotspot、JDK API、JAXWS等），各个部分涉及的语言有C++、Java；而jdk 编码中包含编译代码的Ant 脚本。所以需要安装gcc,ant等。
```[zyl@localhost ~]$ git
usage: git [--version] [--help] [-C <path>] [-c name=value]
           [--exec-path[=<path>]] [--html-path] [--man-path] [--info-path]
           [-p | --paginate | --no-pager] [--no-replace-objects] [--bare]
           [--git-dir=<path>] [--work-tree=<path>] [--namespace=<name>]
           <command> [<args>]

[zyl@localhost ~]$ ant
bash: ant: 未找到命令...
安装软件包“ant”以提供命令“ant”？ [N/y] y
 * 正在队列中等待... 
下列软件包必须安装：
 ant-1.9.6-3.fc24.noarch Java build tool
 ant-lib-1.9.6-3.fc24.noarch Core part of ant
 xalan-j2-2.7.1-28.fc24.noarch Java XSLT processor
 xerces-j2-2.11.0-24.fc24.noarch Java XML parser
 xml-commons-apis-1.4.01-20.fc24.noarch APIs for DOM, SAX, and JAXP
 xml-commons-resolver-1.2-19.fc24.noarch Resolver subproject of xml-commons
[root@localhost ~]# ant -version
Apache Ant(TM) version 1.9.6 compiled on February 3 2016

root@localhost ~]# gcc --help
用法：gcc [选项] 文件...
选项：
  -pass-exit-codes         在某一阶段退出时返回其中最高的错误码。
  --help                   显示此帮助说明。

```

#### 三、jdk7 源码下载
1. oracle jdk:
    http://jdk7.java.net 的oracle jdk，但发现明确源码下载互联网用户约束，仅部分地区可支持下载，其他地区用户暂不支持下载。
```2. openjdk7u:
[zyl@localhost source_jdk7u]$ hg clone http://hg.openjdk.java.net/jdk7u/jdk7u-dev
destination directory: jdk7u-dev
requesting all changes
adding changesets
adding manifests
adding file changes
added 1164 changesets with 1133 changes to 34 files
updating to branch default
33 files updated, 0 files merged, 0 files removed, 0 files unresolved
[zyl@localhost source_jdk7u]$ ls
jdk7u-dev
[zyl@localhost source_jdk7u]$ cd jdk7u-dev/
[zyl@localhost jdk7u-dev]$ ls
ASSEMBLY_EXCEPTION  get_source.sh  LICENSE  make  Makefile  README  README-builds.html  test  THIRD_PARTY_README

[zyl@localhost jdk7u-dev]$ chmod 755 get_source.sh 
[zyl@localhost jdk7u-dev]$ ./get_source.sh 
# Repos:  corba jaxp jaxws langtools jdk hotspot 
Starting on corba
Starting on jaxp
Starting on jaxws
Starting on langtools
Starting on jdk
Starting on hotspot
# hg clone http://hg.openjdk.java.net/jdk7u/jdk7u-dev/corba corba
requesting all changes
adding changesets
adding manifests
adding file changes
transaction abort!
rollback completed
abort: stream ended unexpectedly (got 9391 bytes, expected 14295)
# exit code 255

```
> 经排查，原因为  hg.openjdk.java.net  域名解析异常，尝试简单解决，但无果，除非可找到代理服务器。
故暂时采用另外的方案，即直接去下载。
下载地址：http://download.java.net/openjdk/jdk7/promoted/b147/openjdk-7-fcs-src-b147-27_jun_2011.zip
参考资料：https://blog.csdn.net/baidu_19473529/article/details/76268765?locationNum=2&fps=1
之后下载并解压。

#### 四、进行编译
1. 环境变量配置
>取消JAVA_HOME、CLASSPATH配置、设置LANG和ALT_BOOTDIR配置，其实还依赖其他很多环境变量，但都可使用默认值。ALT_BOOTDIR的值即是当前环境jdk 的路径。
虽然直接执行java 命令可用，但对于linux 系统很多已经默认安装了jdk，而java安装路径如何查找呢？
```[zyl@localhost jdk7u-dev]$ which java
/usr/bin/java
[zyl@localhost jdk7u-dev]$ ls /usr/bin/java -al
lrwxrwxrwx. 1 root root 22 11月 16 2016 /usr/bin/java -> /etc/alternatives/java
[zyl@localhost jdk7u-dev]$ ls /etc/alternatives/java -al
lrwxrwxrwx. 1 root root 72 11月 16 2016 /etc/alternatives/java -> /usr/lib/jvm/java-1.8.0-openjdk-1.8.0.111-1.b16.fc25.x86_64/jre/bin/java
[zyl@localhost jdk7u-dev]$ ls /usr/lib/jvm/java-1.8.0-openjdk-1.8.0.111-1.b16.fc25.x86_64/lib -al
总用量 39832
drwxr-xr-x. 3 root root     4096 3月  14 13:53 .
drwxr-xr-x. 7 root root     4096 3月  14 13:53 ..
drwxr-xr-x. 3 root root     4096 3月  14 13:52 amd64
-rw-r--r--. 1 root root 17738229 10月 20 2016 ct.sym
-rw-r--r--. 1 root root   170677 10月 20 2016 dt.jar
-rw-r--r--. 1 root root    19429 10月 20 2016 ir.idl
-rw-r--r--. 1 root root   449591 10月 20 2016 jconsole.jar
-rwxr-xr-x. 1 root root    11400 10月 20 2016 jexec
-rw-r--r--. 1 root root     1637 10月 20 2016 orb.idl
-rw-r--r--. 1 root root  2311472 10月 20 2016 sa-jdi.jar
-rw-r--r--. 1 root root 20058686 10月 20 2016 tools.jar

设置环境变量
[zyl@localhost jdk7u-dev]$ export LANG=C
[zyl@localhost jdk7u-dev]$ export ALT_BOOTDIR=/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.111-1.b16.fc25.x86_64/

```

2. 编写编译shell脚本，完成编译配置
```
[zyl@localhost source_jdk7]$ pwd
/home/zyl/source_jdk7/openjdk
[zyl@localhost source_jdk7]$ dir
openjdk  openjdk-7-fcs-src-b147-27_jun_2011.zip
[zyl@localhost openjdk]$ vim build.sh
# 语言选项，必须设置，否则编译好后会出现一个 HashTable 的 NPE错 export LANG=C # Bootstrap JDK 解压路径，必须设置 export ALT_BOOTDIR=/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.111-1.b16.fc25.x86_64 # 允许自动下载 export ALLOW_DOWNLOADS=true # 并行编译线程数 export HOTSPOT_BUILD_JOBS=4 export ALT_PARALLEL_COMPILE_JOBS=4 # 比较本次 build 出来的映像与先前版本的差异，对我们没有意义 # 必须设置为 false，否则 sanity 检查为报缺少先前版本 JDK 的映像的错误提示 export SKIP_COMPARE_IMAGE=false # 使用预编译头文件，不加这个编译会变慢 export USE_PRECOMPILED_HEADER=true # 要编译的内容 这里我们全编译其实只要前三个就可以了自行注释 export BUILD_LANGTOOLS=true export BUILD_HOTSPOT=true export BUILD_JDK=true export BUILD_JAXWS=true export BUILD_JAXP=true export BUILD_CORBA=true # 要编译的版本 # export SKIP_DEBUG_BUILD=false # export SKIP_FASTDEBUG_BUILD=true # export DEBUG_NAME=debug # 把它设置为 false 可以避开 javaws 和浏览器 Java 插件之类的部分的 build BUILD_DEPLOY=false # 把它设置为 false 就不会 build 出安装包，因为安装包里有奇怪的依赖 # 但即使不 build 出它也能得到完整的 JDK 映像，所以还是别 build BUILD_INSTALL=false # 编译结果所存放的路径 export ALT_OUTPUTDIR=/home/zyl/source_jdk7/build # 这两个环境变量必须去掉，不然会发生奇怪的事情 # Makefile 检查到这两个变量就会提示警告 unset JAVA_HOME unset CLASSPATH make sanity

```
>chmod a+x build.sh ；./build.sh 执行后报错，报错信息如下：
```ERROR: You seem to not have installed ALSA 0.9.1 or higher. 
       Please install ALSA (drivers and lib). You can download the 
       source distribution from http://www.alsa-project.org or go to 
       http://www.freshrpms.net/docs/alsa/ for precompiled RPM packages. 
 
ERROR: FreeType version  2.3.0  or higher is required. 
 make[2]: Entering directory '/home/zyl/source_jdk7/openjdk/jdk/make/tools/freetypecheck'
/bin/mkdir -p /home/zyl/source_jdk7/build/btbins
rm -f /home/zyl/source_jdk7/build/btbins/freetype_versioncheck
Makefile:68: recipe for target '/home/zyl/source_jdk7/build/btbins/freetype_versioncheck' failed
make[2]: Leaving directory '/home/zyl/source_jdk7/openjdk/jdk/make/tools/freetypecheck'
Failed to build freetypecheck.  

ERROR: You do not have access to valid Cups header files. 
       Please check your access to 
           /usr/include/cups/cups.h 
       and/or check your value of ALT_CUPS_HEADERS_PATH, 
       CUPS is frequently pre-installed on many systems, 
       or may be downloaded from http://www.cups.org 
 
Exiting because of the above error(s). 

```
- 根据如上报错，涉及3个错误，逐个解决：
1、FreeType:  提示需要2.3.0以上版本，即yum 安装：yum install freetype.x86_64 
      然而，在修复2、3 ALS、CPUS问题后编译仍然报错。
    最终是采用：
    [root@localhost ~]# yum list |grep freetype
    [root@localhost ~]# yum install freetype-devel.x86_64 freetype.x86_64
2、ALSA:  Please install ALSA (drivers and lib) ，之后查看 ---查看openjdk 包中的readme_build.html文件：Both alsa and alsa-devel packages are needed.
      即安装deval 包：  yum install alsa-lib-devel.x86_64
3、CUPS：  根据提示，大部分系统默认安装，检查安装情况
      [root@localhost ~]# echo $ALT_CUPS_HEADERS_PATH
      结果为空，即未安装；遂安装：    yum install cups-devel
         
- 再次执行 build.sh ，编译成功，日志如下：
```[zyl@localhost openjdk]$ ./build.sh  ( cd  ./jdk/make && \   make sanity HOTSPOT_IMPORT_CHECK=false JDK_TOPDIR=/home/zyl/source_jdk7/openjdk/jdk JDK_MAKE_SHARED_DIR=/home/zyl/source_jdk7/openjdk/jdk/make/common/shared EXTERNALSANITYCONTROL=true SOURCE_LANGUAGE_VERSION=7 TARGET_CLASS_VERSION=7 MILESTONE=internal BUILD_NUMBER=b00 JDK_BUILD_NUMBER=b00 FULL_VERSION=1.7.0-internal-zyl_2018_08_24_09_25-b00 PREVIOUS_JDK_VERSION=1.6.0 JDK_VERSION=1.7.0 JDK_MKTG_VERSION=7 JDK_MAJOR_VERSION=1 JDK_MINOR_VERSION=7 JDK_MICRO_VERSION=0 PREVIOUS_MAJOR_VERSION=1 PREVIOUS_MINOR_VERSION=6 PREVIOUS_MICRO_VERSION=0 ARCH_DATA_MODEL=64 COOKED_BUILD_NUMBER=0 ALT_OUTPUTDIR=/home/zyl/source_jdk7/build ALT_LANGTOOLS_DIST=/home/zyl/source_jdk7/build/langtools/dist ALT_CORBA_DIST=/home/zyl/source_jdk7/build/corba/dist ALT_JAXP_DIST=/home/zyl/source_jdk7/build/jaxp/dist ALT_JAXWS_DIST=/home/zyl/source_jdk7/build/jaxws/dist ALT_HOTSPOT_IMPORT_PATH=/home/zyl/source_jdk7/build/hotspot/import BUILD_HOTSPOT=true ; ) make[1]: Entering directory '/home/zyl/source_jdk7/openjdk/jdk/make' make[1]: Leaving directory '/home/zyl/source_jdk7/openjdk/jdk/make' Build Machine Information:    build machine = localhost.localdomain Build Directory Structure:    CWD = /home/zyl/source_jdk7/openjdk    TOPDIR = .    LANGTOOLS_TOPDIR = ./langtools    JAXP_TOPDIR = ./jaxp    JAXWS_TOPDIR = ./jaxws    CORBA_TOPDIR = ./corba    HOTSPOT_TOPDIR = ./hotspot    JDK_TOPDIR = ./jdk Build Directives:    BUILD_LANGTOOLS = true     BUILD_JAXP = true     BUILD_JAXWS = true     BUILD_CORBA = true     BUILD_HOTSPOT = true     BUILD_JDK    = true     DEBUG_CLASSFILES =      DEBUG_BINARIES =   Hotspot Settings:        HOTSPOT_BUILD_JOBS  = 4        HOTSPOT_OUTPUTDIR   = /home/zyl/source_jdk7/build/hotspot/outputdir        HOTSPOT_EXPORT_PATH = /home/zyl/source_jdk7/build/hotspot/import    Bootstrap Settings:   BOOTDIR = /usr/lib/jvm/java-1.8.0-openjdk-1.8.0.111-1.b16.fc25.x86_64     ALT_BOOTDIR = /usr/lib/jvm/java-1.8.0-openjdk-1.8.0.111-1.b16.fc25.x86_64   BOOT_VER = 1.8.0 [requires at least 1.6]   OUTPUTDIR = /home/zyl/source_jdk7/build     ALT_OUTPUTDIR = /home/zyl/source_jdk7/build   ABS_OUTPUTDIR = /home/zyl/source_jdk7/build   Build Tool Settings:   SLASH_JAVA = /NOT-SET     ALT_SLASH_JAVA =    VARIANT = OPT   JDK_DEVTOOLS_DIR = /NOT-SET/devtools     ALT_JDK_DEVTOOLS_DIR =    ANT_HOME =    UNIXCOMMAND_PATH = /bin/     ALT_UNIXCOMMAND_PATH =    COMPILER_PATH = /usr/bin/     ALT_COMPILER_PATH =    DEVTOOLS_PATH = /usr/bin/     ALT_DEVTOOLS_PATH =    UNIXCCS_PATH = /usr/ccs/bin/     ALT_UNIXCCS_PATH =    USRBIN_PATH = /usr/bin/     ALT_USRBIN_PATH =    COMPILER_NAME = GCC6   COMPILER_VERSION = GCC6   CC_VER = 6.2.1 [requires at least 4.3.0]   ZIP_VER = 3.0 [requires at least 2.2]   UNZIP_VER = 6.00 [requires at least 5.12]   ANT_VER = 1.9.6 [requires at least 1.7.1]   TEMPDIR = /home/zyl/source_jdk7/build/tmp   Build Directives:   OPENJDK = true   USE_HOTSPOT_INTERPRETER_MODE =    PEDANTIC =    DEV_ONLY =    NO_DOCS =    NO_IMAGES =    TOOLS_ONLY =    INSANE =    COMPILE_APPROACH = parallel   PARALLEL_COMPILE_JOBS = 4     ALT_PARALLEL_COMPILE_JOBS = 4   FASTDEBUG =    COMPILER_WARNINGS_FATAL = false   COMPILER_WARNING_LEVEL =    SHOW_ALL_WARNINGS =    INCREMENTAL_BUILD = false   CC_HIGHEST_OPT =    CC_HIGHER_OPT =    CC_LOWER_OPT =    CXXFLAGS =  -O2 -fPIC -DCC_NOEX -W -Wall  -Wno-unused -Wno-parentheses -fno-omit-frame-pointer -D_LITTLE_ENDIAN     CFLAGS =  -O2   -fno-strict-aliasing -fPIC -W -Wall  -Wno-unused -Wno-parentheses -pipe -fno-omit-frame-pointer -D_LITTLE_ENDIAN     BOOT_JAVA_CMD = /usr/lib/jvm/java-1.8.0-openjdk-1.8.0.111-1.b16.fc25.x86_64/bin/java -XX:-PrintVMOptions -XX:+UnlockDiagnosticVMOptions -XX:-LogVMOutput -Xmx512m -Xms512m -XX:PermSize=32m -XX:MaxPermSize=160m   BOOT_JAVAC_CMD = /usr/lib/jvm/java-1.8.0-openjdk-1.8.0.111-1.b16.fc25.x86_64/bin/javac  -J-XX:ThreadStackSize=1536 -J-XX:-PrintVMOptions -J-XX:+UnlockDiagnosticVMOptions -J-XX:-LogVMOutput -J-Xmx512m -J-Xms512m -J-XX:PermSize=32m -J-XX:MaxPermSize=160m -encoding ascii -source 6 -target 6 -XDignore.symbol.file=true   BOOT_JAR_CMD = /usr/lib/jvm/java-1.8.0-openjdk-1.8.0.111-1.b16.fc25.x86_64/bin/jar   BOOT_JARSIGNER_CMD = /usr/lib/jvm/java-1.8.0-openjdk-1.8.0.111-1.b16.fc25.x86_64/bin/jarsigner   JAVAC_CMD = /NOT-SET/re/jdk/1.7.0/promoted/latest/binaries/linux-amd64/bin/javac  -J-XX:ThreadStackSize=1536 -J-XX:-PrintVMOptions -J-XX:+UnlockDiagnosticVMOptions -J-XX:-LogVMOutput -J-Xmx512m -J-Xms512m -J-XX:PermSize=32m -J-XX:MaxPermSize=160m  -source 7 -target 7 -encoding ascii -Xbootclasspath:/home/zyl/source_jdk7/build/classes    JAVAH_CMD = /NOT-SET/re/jdk/1.7.0/promoted/latest/binaries/linux-amd64/bin/javah -bootclasspath /home/zyl/source_jdk7/build/classes   JAVADOC_CMD = /NOT-SET/re/jdk/1.7.0/promoted/latest/binaries/linux-amd64/bin/javadoc -J-XX:-PrintVMOptions -J-XX:+UnlockDiagnosticVMOptions -J-XX:-LogVMOutput -J-Xmx512m -J-Xms512m -J-XX:PermSize=32m -J-XX:MaxPermSize=160m -bootclasspath /home/zyl/source_jdk7/build/classes   Build Platform Settings:   USER = zyl   PLATFORM = linux   ARCH = amd64   LIBARCH = amd64   ARCH_FAMILY = amd64   ARCH_DATA_MODEL = 64   ARCHPROP = amd64   ALSA_VERSION = 1.1.1   OS_VERSION = 4.8.6-300.fc25.x86_64 [requires at least 2.6]   OS_VARIANT_NAME = Fedora   OS_VARIANT_VERSION = 25   MB_OF_MEMORY = 2917   GNU Make Settings:   MAKE = make   MAKE_VER = 4.1 [requires at least 3.81]   MAKECMDGOALS = sanity   MAKEFLAGS = w   SHELL = /bin/sh   Target Build Versions:   JDK_VERSION = 1.7.0   MILESTONE = internal   RELEASE = 1.7.0-internal   FULL_VERSION = 1.7.0-internal-zyl_2018_08_24_09_25-b00   BUILD_NUMBER = b00   External File/Binary Locations:   USRJDKINSTANCES_PATH = /opt/java   BUILD_JDK_IMPORT_PATH = /NOT-SET/re/jdk/1.7.0/promoted/latest/binaries     ALT_BUILD_JDK_IMPORT_PATH =    JDK_IMPORT_PATH = /NOT-SET/re/jdk/1.7.0/promoted/latest/binaries/linux-amd64     ALT_JDK_IMPORT_PATH =    LANGTOOLS_DIST =      ALT_LANGTOOLS_DIST = /home/zyl/source_jdk7/build/langtools/dist   CORBA_DIST =      ALT_CORBA_DIST = /home/zyl/source_jdk7/build/corba/dist   JAXP_DIST =      ALT_JAXP_DIST = /home/zyl/source_jdk7/build/jaxp/dist   JAXWS_DIST =      ALT_JAXWS_DIST = /home/zyl/source_jdk7/build/jaxws/dist   HOTSPOT_DOCS_IMPORT_PATH = /NO_DOCS_DIR     ALT_HOTSPOT_DOCS_IMPORT_PATH =    HOTSPOT_IMPORT_PATH = /home/zyl/source_jdk7/build/hotspot/import     ALT_HOTSPOT_IMPORT_PATH = /home/zyl/source_jdk7/build/hotspot/import   HOTSPOT_SERVER_PATH = /home/zyl/source_jdk7/build/hotspot/import/jre/lib/amd64/server     ALT_HOTSPOT_SERVER_PATH =    CACERTS_FILE = ./../src/share/lib/security/cacerts     ALT_CACERTS_FILE =    CUPS_HEADERS_PATH = /usr/include     ALT_CUPS_HEADERS_PATH =    OpenJDK-specific settings:   FREETYPE_HEADERS_PATH = /usr/include     ALT_FREETYPE_HEADERS_PATH =    FREETYPE_LIB_PATH = /usr/lib     ALT_FREETYPE_LIB_PATH =    Previous JDK Settings:   PREVIOUS_RELEASE_PATH = USING-PREVIOUS_RELEASE_IMAGE     ALT_PREVIOUS_RELEASE_PATH =    PREVIOUS_JDK_VERSION = 1.6.0     ALT_PREVIOUS_JDK_VERSION =    PREVIOUS_JDK_FILE =      ALT_PREVIOUS_JDK_FILE =    PREVIOUS_JRE_FILE =      ALT_PREVIOUS_JRE_FILE =    PREVIOUS_RELEASE_IMAGE = /usr/lib/jvm/java-1.8.0-openjdk-1.8.0.111-1.b16.fc25.x86_64     ALT_PREVIOUS_RELEASE_IMAGE =  Sanity check passed.
执行成功，Sanity检查结果如下： Sanity check passed.
执行make，报错信息截取如下：
WARNING: LANG has been set to zh_CN.UTF-8, this can cause build failures. 
         Try setting LANG to 'C'. 
 
BUILD FAILED
/home/zyl/source_jdk7/openjdk/langtools/make/build.xml:860: Error running /NO_BOOTDIR/bin/javac compiler

```
从上面来看，build.sh 脚本中  export  LANG=C 未生效，而BOOTDIR 值应该也有问题。
- 解决方案：
1.设置环境变量方式有误： 永久环境变量（属于文件，比如profile以及home目录下的），临时环境变量（当前shell以及子线程），普通环境变量（当前shell） --- 如上LANG 环境变量警告问题
解决：在当前shell执行相关export
2. FreeType 没有安装
解决：yum install freetype-devel
3. ALSA 没有安装（声卡  应该没什么关系）
解决：sudo yum install alsa*
4：You do not have access to valid Cups header files. 
解决：sudo yum install cups-devel.x86_64

#### 实践：
```
[zyl@localhost openjdk]$ export LANG=C
[zyl@localhost openjdk]$ export ALT_BOOTDIR=/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.111-1.b16.fc25.x86_64
[zyl@localhost openjdk]$ unset JAVA_HOME
[zyl@localhost openjdk]$ unset CLASSPATH 

```
再次 make ，如上错误已经不存在，开始编译过程，编译过程中警告和报错：
警告：
warning: [options] bootstrap class path not set in conjunction with -source 1.6
解决方案（按如下操作再编译警告不在在）：
```[zyl@localhost source_jdk7]$ find . -name rules.make
./openjdk/hotspot/make/linux/makefiles/rules.make
./openjdk/hotspot/make/solaris/makefiles/rules.make
./openjdk/hotspot/make/windows/makefiles/rules.make
[zyl@localhost source_jdk7]$ vi openjdk/hotspot/make/linux/makefiles/rules.make 
# Settings for javac
BOOT_SOURCE_LANGUAGE_VERSION = 8   --- 原值为6，修改为8
BOOT_TARGET_CLASS_VERSION = 8  --- 原值为6，修改为8

错误：
build-bootstrap-javac:
 .............
    [javac] /home/zyl/source_jdk7/openjdk/langtools/src/share/classes/com/sun/tools/javac/comp/Resolve.java:2157: warning: [overrides] Class Resolve.InapplicableSymbolsError.Candidate overrides equals, but neither it nor any superclass overrides hashCode method
    [javac]         private class Candidate {
    [javac]                 ^
    [javac] error: warnings found and -Werror specified
    [javac] 1 error
    [javac] 1 warning

BUILD FAILED
/home/zyl/source_jdk7/openjdk/langtools/make/build.xml:452: The following error occurred while executing this line:
/home/zyl/source_jdk7/openjdk/langtools/make/build.xml:795: Compile failed; see the compiler error output for details.

```
初步分析来看，感觉与jdk版本有关，决定把把bootstrap jdk 换成oracle 的 jdk7，于是使用命令yum remove 卸载jdk1.8.0 :  yum remove java-1.8.0-openjdk.x86_64 
http://www.oracle.com/technetwork/java/javase/downloads/java-archive-downloads-javase7-521261.html    
下载：jdk-7u80-linux-x64.rpm     oracle账号：28*@*com/*****
```
[root@localhost soft]# rpm -ivh jdk-7u80-linux-x64.rpm 
准备中...                          ################################# [100%]
正在升级/安装...
   1:jdk-2000:1.7.0_80-fcs            ################################# [100%]
Unpacking JAR files...
rt.jar...
......
[root@localhost soft]# ls /usr/java/jdk1.7.0_80/bin/
appletviewer  jar        javafxpackager  jcmd     ......
[root@localhost soft]# java -version
java version "1.7.0_80"
Java(TM) SE Runtime Environment (build 1.7.0_80-b15)
Java HotSpot(TM) 64-Bit Server VM (build 24.80-b11, mixed mode)

```
退出shell 窗口重新进入，避免之前设置的环境变量冲突；需修改jdk路径。
```
[zyl@localhost openjdk]$ export LANG=C
[zyl@localhost openjdk]$ export ALT_BOOTDIR=/usr/java/jdk1.7.0_80
[zyl@localhost openjdk]$ unset JAVA_HOME
[zyl@localhost openjdk]$ unset CLASSPATH 
```

之后再次make ，终于终于不再报错，开始编译了，之后就等待。。。然，等待一段时间后再次报错。
```BUILD FAILED
/home/zyl/source_jdk7/openjdk/jaxp/build-defs.xml:70: ERROR: Cannot find source for project jaxp.

HINT: Try setting drops.dir to indicate where the bundles can be found, or try setting the ant property allow.downloads=true to download the bundle from the URL.
e.g. ant -Dallow.downloads=true -OR- ant -Ddrops.dir=some_directory

# 添加变量开关
export ALLOW_DOWNLOADS=true

```

- 添加环境变量后，再次编译。
```
build/linux-amd64/jaxws/build/xml_generated/build-drop-jaf_src.xml:96: Redirection detected from https to http. Protocol switch unsafe, not allowed.
下载依赖包：
  cd source_jdk7/openjdk/
  mkdir drop
  cd drop/
  wget http://download.java.net/jaxp/1.4.5/jaxp145_01.zip
  wget https://netix.dl.sourceforge.net/project/jdk7src/input-archives/jdk7-jaf-2010_08_19.zip
  wget http://download.java.net/glassfish/components/jax-ws/openjdk/jdk7/jdk7-jaxws2_2_4-b03-2011_05_27.zip
```

- 在编译中添加环境变量：
export ALT_DROPS_DIR=/home/zyl/source_jdk7/openjdk/drop

- 添加环境变量后，再次编译。
make[5]: Entering directory '/home/zyl/source_jdk7/build/hotspot/outputdir'
>&2 echo "*** This OS is not supported:" `uname -a`; exit 1;
*** This OS is not supported: Linux localhost.localdomain 4.8.6-300.fc25.x86_64 #1 SMP Tue Nov 1 12:36:38 UTC 2016 x86_64 x86_64 x86_64 GNU/Linux
/home/zyl/source_jdk7/openjdk/hotspot/make/linux/Makefile:239: recipe for target 'check_os_version' failed

- 在编译中添加环境变量解决编译错误：OpenJDK7太新了，不在支持的操作系统列表里
export DISABLE_HOTSPOT_OS_VERSION_CHECK=OK

- 添加环境变量后，再次编译。
make[6]: g++: Command not found
/home/zyl/source_jdk7/openjdk/hotspot/make/linux/makefiles/adlc.make:205: recipe for target '../generated/adfiles/adlparse.o' failed

- 安装依赖软件：
```
yum install gcc-c++.x86_64 

/home/zyl/source_jdk7/openjdk/hotspot/src/share/vm/code/dependencies.hpp:161:61: error: enumerator value for 'all_types' is not an integer constant
     all_types      = ((1<<TYPE_LIMIT)-1) & ((-1)<<FIRST_TYPE),
                                                             ^
cc1plus: all warnings being treated as errors
/home/zyl/source_jdk7/openjdk/hotspot/make/linux/makefiles/vm.make:260: recipe for target 'precompiled.hpp.gch' failed
make[6]: *** [precompiled.hpp.gch] Error 1
make[6]: Leaving directory '/home/zyl/source_jdk7/build/hotspot/outputdir/linux_amd64_compiler2/product'
/home/zyl/source_jdk7/openjdk/hotspot/make/linux/makefiles/top.make:117: recipe for target 'the_vm' failed

```

- 想到之前有关于VERSION=8的设置，决定把配置还原为6
```
[zyl@localhost source_jdk7]$ vi openjdk/hotspot/make/linux/makefiles/rules.make
# Settings for javac
BOOT_SOURCE_LANGUAGE_VERSION = 6   
BOOT_TARGET_CLASS_VERSION = 6 

```
- 再次编译，报错如下：
```Waning
/usr/java/jdk1.7.0_80/bin/javac -g -encoding ascii -source 6 -target 6 -classpath /usr/java/jdk1.7.0_80/lib/tools.jar -sourcepath /home/zyl/source_jdk7/openjdk/hotspot/agent/src/share/classes -d /home/zyl/source_jdk7/build/hotspot/outputdir/linux_amd64_compiler2/product/../generated/saclasses @/home/zyl/source_jdk7/build/hotspot/outputdir/linux_amd64_compiler2/product/../generated/agent2.classes.list
warning: [options] bootstrap class path not set in conjunction with -source 1.6
Note: Some input files use unchecked or unsafe operations.
Note: Recompile with -Xlint:unchecked for details.

/home/zyl/source_jdk7/openjdk/hotspot/src/share/vm/code/dependencies.hpp:161:61: error: enumerator value for 'all_types' is not an integer constant
     all_types      = ((1<<TYPE_LIMIT)-1) & ((-1)<<FIRST_TYPE),
                                                             ^
cc1plus: all warnings being treated as errors
/home/zyl/source_jdk7/openjdk/hotspot/make/linux/makefiles/vm.make:260: recipe for target 'precompiled.hpp.gch' failed
make[6]: *** [precompiled.hpp.gch] Error 1
make[6]: Leaving directory '/home/zyl/source_jdk7/build/hotspot/outputdir/linux_amd64_compiler2/product'
/home/zyl/source_jdk7/openjdk/hotspot/make/linux/makefiles/top.make:117: recipe for target 'the_vm' failed

```

- 从日志来看，当前BOOTSTRAP jdk 版本为1.7，但是强制使用6来编译；遂尝试将VERSION=8的值修改为7，再次编译。
```
/home/zyl/source_jdk7/openjdk/hotspot/src/share/vm/memory/threadLocalAllocBuffer.inline.hpp:97:25: error: invalid suffix on literal; C++11 requires a space between literal and string macro [-Werror=literal-suffix]
     gclog_or_tty->print("TLAB: %s thread: "INTPTR_FORMAT" [id: %2d]"
permissive]
     all_types      = ((1<<TYPE_LIMIT)-1) & ((-1)<<FIRST_TYPE),
                                            ~~~~~^~~~~~~~~~~~~
/home/zyl/source_jdk7/openjdk/hotspot/src/share/vm/code/dependencies.hpp:161:61: error: enumerator value for 'all_types' is not an integer constant
     all_types      = ((1<<TYPE_LIMIT)-1) & ((-1)<<FIRST_TYPE),
                                                             ^
cc1plus: all warnings being treated as errors
/home/zyl/source_jdk7/openjdk/hotspot/make/linux/makefiles/vm.make:260: recipe for target 'precompiled.hpp.gch' failed
make[6]: *** [precompiled.hpp.gch] Error 1
make[6]: Leaving directory '/home/zyl/source_jdk7/build/hotspot/outputdir/linux_amd64_compiler2/product'
/home/zyl/source_jdk7/openjdk/hotspot/make/linux/makefiles/top.make:117: recipe for target 'the_vm' failed
make[5]: *** [the_vm] Error 2
make[5]: Leaving directory '/home/zyl/source_jdk7/build/hotspot/outputdir/linux_amd64_compiler2/product'
/home/zyl/source_jdk7/openjdk/hotspot/make/linux/Makefile:289: recipe for target 'product' failed

```
- literal-suffix——宏和字符串中间要加空格
网上各种找资料，说是jdk 的 bug，因为C++11 对宏方法参数空格处理差异，说是新版本修复，但也可能通过手动修改源码，添加空格解决。但并没有其他可行的方法，所以决定尝试编译jdk 8.
故卸载 jdk 1.7，重新安装jdk1.8，下载 jdk 1.8 源码。
[root@localhost ~]# rpm -qa |grep jdk   --- 查看当前安装的jdk 版本。
yum install java --- 默认安装的版本即为1.8


修改build.sh 脚本中的BOOT_DIR 目录后执行如下报错：
No configurations found for /home/zyl/jdk8_source/openjdk-8u/openjdk/! Please run configure to create a configuration.

初步分析需要先行configure 操作，确实源码目录有configure 文件，具体呢？其实所有源码包文件均有 README-builds.html ，浏览器中打开：第一步是检查openjdk构建所需系统是否已有。
```
The very first step in building the OpenJDK is making sure the system itself has everything it needs to do OpenJDK builds. 
Building the OpenJDK is now done with running a configure script which will try and find and verify you have everything you need, followed by running make, e.g.

    bash ./configure
    make all 

```


于是执行 ./configure ，错误信息如下 ：
```configure: Found potential Boot JDK using well-known locations (in /usr/lib/jvm/jre-openjdk)
configure: Potential Boot JDK found at /usr/lib/jvm/jre-openjdk did not contain bin/javac; ignoring
configure: (This might be an JRE instead of an JDK)
configure: Found potential Boot JDK using well-known locations (in /usr/lib/jvm/jre-1.8.0-openjdk-1.8.0.151-1.b12.fc25.x86_64)
configure: Potential Boot JDK found at /usr/lib/jvm/jre-1.8.0-openjdk-1.8.0.151-1.b12.fc25.x86_64 did not contain bin/javac; ignoring
configure: (This might be an JRE instead of an JDK)
configure: Found potential Boot JDK using well-known locations (in /usr/lib/jvm/jre-1.8.0-openjdk)
configure: Potential Boot JDK found at /usr/lib/jvm/jre-1.8.0-openjdk did not contain bin/javac; ignoring
configure: (This might be an JRE instead of an JDK)
configure: Found potential Boot JDK using well-known locations (in /usr/lib/jvm/jre-1.8.0)
configure: Potential Boot JDK found at /usr/lib/jvm/jre-1.8.0 did not contain bin/javac; ignoring
configure: (This might be an JRE instead of an JDK)
configure: Found potential Boot JDK using well-known locations (in /usr/lib/jvm/jre)
configure: Potential Boot JDK found at /usr/lib/jvm/jre did not contain bin/javac; ignoring
configure: (This might be an JRE instead of an JDK)
configure: Found potential Boot JDK using well-known locations (in /usr/lib/jvm/java-1.8.0-openjdk-1.8.0.151-1.b12.fc25.x86_64)
configure: Potential Boot JDK found at /usr/lib/jvm/java-1.8.0-openjdk-1.8.0.151-1.b12.fc25.x86_64 did not contain bin/java; ignoring
configure: Could not find a valid Boot JDK. You might be able to fix this by running 'sudo yum install java-1.7.0-openjdk'.
configure: This might be fixed by explicitely setting --with-boot-jdk
configure: error: Cannot continue
configure exiting with result code 1
[zyl@localhost openjdk]$ ./configure --with-boot-jdk=/home/zyl/soft/jdk1.8.0_181

添加参数再执行：./configure --with-boot-jdk=/home/zyl/soft/jdk1.8.0_181
checking for Mac OS X Java Framework... no
checking for X... no
configure: error: Could not find X11 libraries. You might be able to fix this by running 'sudo yum install libXtst-devel libXt-devel libXrender-devel'.
configure exiting with result code 1

执行    yum install libXtst-devel libXt-devel libXrender-devel  完成安装，再次 ./configure --with-boot-jdk=/home/zyl/soft/jdk1.8.0_181
配置成功，执行结果如下：
==================================================
A new configuration has been successfully created in
/home/zyl/jdk8_source/openjdk-8u/openjdk/build/linux-x86_64-normal-server-release
using configure arguments '--with-boot-jdk=/home/zyl/soft/jdk1.8.0_181'.

Configuration summary:
* Debug level:    release
* JDK variant:    normal
* JVM variants:   server
* OpenJDK target: OS: linux, CPU architecture: x86, address length: 64

Tools summary:
* Boot JDK:       java version "1.8.0_181" Java(TM) SE Runtime Environment (build 1.8.0_181-b13) Java HotSpot(TM) 64-Bit Server VM (build 25.181-b13, mixed mode)  (at /home/zyl/soft/jdk1.8.0_181)
* C Compiler:     gcc (GCC) 6.4.1 20170727 (Red Hat-1) version 6.4.1-1) (at /usr/bin/gcc)
* C++ Compiler:   g++ (GCC) 6.4.1 20170727 (Red Hat-1) version 6.4.1-1) (at /usr/bin/g++)

Build performance summary:
* Cores to use:   1
* Memory limit:   2917 MB
* ccache status:  not installed (consider installing)

Build performance tip: ccache gives a tremendous speedup for C++ recompilations.
You do not have ccache installed. Try installing it.
You might be able to fix this by running 'sudo yum install ccache'.

```


再次编译，仍然报错：
    /home/zyl/jdk8_source/openjdk-8u/openjdk/hotspot/src/share/vm/memory/threadLocalAllocBuffer.inline.hpp:97:25: 错误：invalid suffix on literal; C++11 requires a space between literal and string macro [-Werror=literal-suffix]
从报错来看，仍然还是与C++11的新特性有关，于是尝试c++降级。---同时网上也确实有类似安装指定c++版本以降级支持各版本编译的事情。----不要安装编译器版本高于5的，因为默认启用c++14 导致编译中断。

补充：安装gcc4.6.4   http://mirror.hust.edu.cn/gnu/gcc/gcc-4.8.5/
据悉： gcc 4.7.1 开始支持C++11 特性；另参考jdk 源码目录中 build-readme.html如下资料：即编译 jdk7 至少需要 gcc 4.3 版本，最终选择的是  gcc  4.6.4。
Minimum Build Environments：
 

安装 gcc ：查询资料【强烈推荐zyl-实践-gcc】GCC 手动编译安装（gcc降级以支持jdk编译）
并将gcc 版本切换至 gcc  4.6.4.

到此，再次编译jdk8，先configure 再执行build.sh 脚本。

/home/zyl/jdk8_source/openjdk-8u/openjdk/hotspot/src/share/vm/memory/generation.hpp:425:17: 错误：invalid suffix on literal; C++11 requires a space between literal and string macro [-Werror=literal-suffix]
         warning("time warp: "INT64_FORMAT" to "INT64_FORMAT, (int64_t)_time_of_last_gc, (int64_t)now);

最终，暂未找到解决办法，以失败结束本次尝试。























9、开始编译--参考（make all ARCH_DATA_MODEL=64 ALLOW_DOWNLOADS=true）
--bug清单，编译问题可在这儿查找：https://bugs.openjdk.java.net/secure/RapidBoard.jspa?projectKey=JDK&useStoredSettings=true&rapidView=9
10、 openJDK9 编译较简单，./configure  失败日志已提示失败信赖包，如：
        yum install gcc-c++.i686
        yum install libXtst-devel libXt-devel libXrender-devel libXi-devel
        yum install freetype-devel
        之后执行 make即可。
