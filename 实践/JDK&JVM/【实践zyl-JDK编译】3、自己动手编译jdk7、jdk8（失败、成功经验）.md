> 基于VMware Centos 7.5 64位DVD安装版编译JDK7（编译JDK7失败记录，之后会有JDK8编译成功记录）：
#### 一、下载jdk7 源码
> jdk7下载地址:http://download.java.net/openjdk/jdk7/promoted/b147/openjdk-7-fcs-src-b147-27_jun_2011.zip:
> 段落引用解析后，查看 README-builds.html 文件 Minimum Build Environments：
    Linux X64 (64-bit)  Fedora 9  gcc 4.3  JDK 6u18

#### 二、下载 BOOT JDK：  JDK 6u18
打开Oracle jdk的下载官方页面：
    http://www.oracle.com/technetwork/java/javase/downloads/index.html
因jdk 6 版本较老，滑动页面至最下方：Java Archive，点击最后的download,即可进入归档的java 版本下载页面，找到对应的jdk，接受协议，点击下载（需登录oracle账号: 2*@*com/****) : jdk-6u18-linux-x64-rpm.bin
安装：添加x 权限，使用管理员权限执行：
     ./jdk-6u18-linux-x64-rpm.bin
    --- 命令行安装，输入q、yes  完成协议的同意；另因本机已默认安装jdk8,安装过程中需确认是否替换jdk8. 是否替换应该都可以，但我选择的替换。貌似也并没有什么卵用。
安装完成后，默认java -version 仍然是1.8，但到  /usr/java/jdk1.6.0_18/bin 中执行./java -version，版本号为1.6。

#### 三、检查依赖环境（阅读官方文档：README-builds.html)：
##### 1、-- The /usr/bin/make should be 3.81 or newer and should work fine for you.
[zyl@localhost make]$ make --version
GNU Make 3.82
Built for x86_64-redhat-linux-gnu

##### 2、内存、磁盘、PATH、JAVA_HOME等
 X64 only: The minimum recommended hardware for building the Linux version is an AMD Opteron class processor, at least 512 MB of RAM, and approximately 4 GB of free disk space.

The build will use the tools contained in /bin and /usr/bin of a standard installation of the Linux operating environment. You should ensure that these directories are in your PATH.

Note that some Linux systems have a habit of pre-populating your environment variables for you, for example JAVA_HOME might get pre-defined for you to refer to the JDK installed on your Linux system. You will need to unset JAVA_HOME. It's a good idea to run env and verify the environment variables you are getting from the default system settings make sense for building the OpenJDK. 

Basic Linux Check List
        1、Install the Bootstrap JDK, set ALT_BOOTDIR.
       ---- 已安装jdk 1.6
        2、Optional Import JDK, set ALT_JDK_IMPORT_PATH.
        ---- 可选项，不配置
       3、 Install or upgrade the FreeType development package.
              ----- 默认安装 ；报错再处理
        4、Install Ant 1.7.1 or newer, make sure it is in your PATH.
                 ----- 未安装：yum install ant.noarch  ：Apache Ant(TM) version 1.9.2 

查看 Build Dependencies ，应该还有部分依赖；但决定暂不处理，待编译报错再针对处理
Bootstrap JDK、Optional Import JDK（可选）、Ant、gcc/binutils、Zip and Unzip、CUPS、 XRender、FreeType 2、ALSA

#### 四、编译检查
Creating the Build

    Once a machine is setup to build the OpenJDK, the steps to create the build are fairly simple. The various ALT settings can either be made into variables or can be supplied on the gmake command.

        Use the sanity rule to double check all the ALT settings:

            gmake sanity [ARCH_DATA_MODEL=32 or 64] [other "ALT_" overrides] 

        Start the build with the command:

            gmake [ARCH_DATA_MODEL=32 or 64] [ALT_OUTPUTDIR=output_directory] [other "ALT_" overrides] 

    Solaris: Note that ARCH_DATA_MODEL is really only needed on Solaris to indicate you want to built the 64-bit version. And before the Solaris 64-bit binaries can be used, they must be merged with the binaries from a separate 32-bit build. The merged binaries may then be used in either 32-bit or 64-bit mode, with the selection occurring at runtime with the -d32 or -d64 options. 

- 错误一：
[zyl@localhost openjdk7]$ make sanity ALT_BOOTDIR= /usr/java/jdk1.6.0_18
/bin/sh: /usr/bin/gcc: 没有那个文件或目录
/bin/sh: /usr/bin/gcc: 没有那个文件或目录
    解决：（根据文档:README-builds.html)
    yum install gcc gcc-c++

- 错误二：
[zyl@localhost openjdk7]$ make sanity ALT_BOOTDIR= /usr/java/jdk1.6.0_18
    grep: /usr/include/alsa/version.h: 没有那个文件或目录
    解决：（根据文档:README-builds.html)
    rpm -qa | grep alsa    ---- 从结果看，未安装alsa-devel 
    yum list |grep alsa    ----- 确认需安装 alsa-lib-devel.x86_64
    yum install alsa-lib-devel.x86_64

- 错误三：
[zyl@localhost openjdk7]$ make sanity ALT_BOOTDIR= /usr/java/jdk1.6.0_18
    freetypecheck.c:32:22: 致命错误：ft2build.h：没有那个文件或目录
    
      BOOTDIR = /NO_BOOTDIR
        ALT_BOOTDIR = 
      BOOT_VER = /bin/sh: /NO_BOOTDIR/bin/java: 没有那个文件或目录 [requires at least 1.6]
    
    解决：（根据文档:README-builds.html)
    freetype(依据01文档及经验):
    yum list |grep freetype----- 确认需安装   freetype-devel.x86_64
    yum install freetype-devel.x86_64

自定义Shell脚本./build.sh：
    export ALT_BOOTDIR=/usr/java/jdk1.6.0_18
    unset JAVA_HOME
    make sanity
    
- 错误四：
[zyl@localhost openjdk7]$ ./build.sh
    
    WARNING: LANG has been set to zh_CN.UTF-8, this can cause build failures. 
             Try setting LANG to 'C'. 
     
    ERROR: You do not have access to valid Cups header files. 
           Please check your access to 
               /usr/include/cups/cups.h 
           and/or check your value of ALT_CUPS_HEADERS_PATH, 
           CUPS is frequently pre-installed on many systems, 
           or may be downloaded from http://www.cups.org 
    解决：
     Plus the following packages:
    
            cups devel: Cups Development Package
            alsa devel: Alsa Development Package
            ant: Ant Package
            Xi devel: libXi.so Development Package
安装cups:
    yum list |grep cups
     yum install cups-devel.x86_64
自定义Shell脚本./build.sh：
    export ALT_BOOTDIR=/usr/java/jdk1.6.0_18
    export LANG=C
    unset JAVA_HOME
    make sanity
    
到此 ，Sanity check passed.

#### 五、编译
遂修改脚本build.sh添加：make all
    export LANG=C
    export ALT_BOOTDIR=/usr/java/jdk1.6.0_18
    unset JAVA_HOME
    make sanity
    make all

执行./build.sh，短时间没报错，之后就是等待。。。

- 问题一，如期报错：
    BUILD FAILED
    /home/zyl/jdk_source/openjdk7/jaxp/build-defs.xml:70: ERROR: Cannot find source for project jaxp.
    
    HINT: Try setting drops.dir to indicate where the bundles can be found, or try setting the ant property allow.downloads=true to download the bundle from the URL.
    解决(官方文档：Managing the Source Drops)：
    build.sh脚本添加 export  ALLOW_DOWNLOADS=true
    
- 问题二：
    BUILD FAILED
    /home/zyl/jdk_source/openjdk7/build/linux-amd64/jaxws/build/xml_generated/build-drop-jaf_src.xml:96: Redirection detected from https to http. Protocol switch unsafe, not allowed.
解决：（参考01实践）
下载依赖包：
  cd ......./openjdk/
  mkdir drop
  cd drop/
下载plug
   wget http://download.java.net/jaxp/1.4.5/jaxp145_01.zip
   wget https://netix.dl.sourceforge.net/project/jdk7src/input-archives/jdk7-jaf-2010_08_19.zip
  wget http://download.java.net/glassfish/components/jax-ws/openjdk/jdk7/jdk7-jaxws2_2_4-b03-2011_05_27.zip

在编译中添加环境变量：
export ALT_DROPS_DIR=/home/zyl/source_jdk7/openjdk/drop

遂修改脚本build.sh:
    export LANG=C
    export ALT_BOOTDIR=/usr/java/jdk1.6.0_18
    export  ALLOW_DOWNLOADS=true
    export ALT_DROPS_DIR=/home/zyl/jdk_source/openjdk7/drop
    unset JAVA_HOME
    make sanity
    make all

- 问题三：
    make[5]: Entering directory `/home/zyl/jdk_source/openjdk7/build/linux-amd64/hotspot/outputdir'
    >&2 echo "*** This OS is not supported:" `uname -a`; exit 1;
    解决：
[zyl@localhost openjdk7]$ uname -r
3.10.0-862.el7.x86_64
[zyl@localhost openjdk7]$ vi hotspot/make/linux/Makefile 
    在这行最后加上当前的内核版本3.10%，如下：
SUPPORTED_OS_VERSION = 2.4% 2.5% 2.6% 2.7% 3.10%

- 问题四：
    头文件宏定义冲突
    /home/mengxiansen/openjdk/openjdk/hotspot/src/share/vm/runtime/interfaceSupport.hpp:430:0: error: "__LEAF" redefined [-Werror]
     #define __LEAF(result_type, header) 
    /usr/include/x86_64-linux-gnu/sys/cdefs.h:42:0: note: this is the location of the previous definition
      define __LEAF , __leaf__
    解决：
在interfaceSupport.hpp增加
#ifdef __LEAF
#undef __LEAF
#define __LEAF(result_type, header)                                  \
  TRACE_CALL(result_type, header)                                    \
  debug_only(NoHandleMark __hm;)                                     \
  /* begin of body */
#endif

注：找到 #define __LEAF(result_type, header)      ,在上方加红色字体两行ifdef 、undef 。

- 问题五：
    /home/zyl/jdk_source/openjdk7/hotspot/src/share/vm/runtime/interfaceSupport.hpp:25:0: error: unterminated #ifndef
     #ifndef SHARE_VM_RUNTIME_INTERFACESUPPORT_HPP
解决：
1,权限问题   ---- 使用root 执行，验证无效
2,少了#endif  ---- 加了又报错说多了endif 
把这个文件拖下来看源码，c++ 不熟，但借助工具来看，貌似没有少这个东东。 


### **-----各种尝试，仍然无解；无奈之下，决定尝试编译jdk8（成功）**
参考：
    https://www.aliyun.com/jiaocheng/781416.html
    https://www.linuxidc.com/Linux/2017-06/144713.htm
> openjdk8跟之前的版本编译方式不一样,之前的版本是基于Ant 、ALT_*环境变量的编译方式,而openjdk8开始则是”configure &;&; make”模式。(详细说明可以阅览官方文档)
#### 一：jdk源码及bootstrap jdk 下载。
基于“【实践zyl 】openjdk 源码下载技巧” 下载openjdk 8 源码：
    openjdk-8-src-b132-03_mar_2014.zip
> bootstrap jdk：进入 oracle jdk下载页面
    https://www.oracle.com/technetwork/java/javase/downloads/index.html
，选择最下方“Java Archive” 后的 DOWNLOAD即可下载历史版本jdk。遂下载 jdk 7u80的
64位tar.gz包：jdk-7u80-linux-x64.tar.gz,然后解压。
    
#### 二、构建编译环境
进入 openjdk8 源码目录
//生成配置信息并构建编译环境 
> bash ./configure --with-target-bits=64 --with-boot-jdk=/home/zyl/soft/jdk1.7.0_80/ --with-debug-level=slowdebug --enable-debug-symbols ZIP_DEBUGINFO_FILES=0  
以上的参数简单作一些说明: 
```–with-target-bits=64 :指定生成64位jdk; 
–with-boot-jdk=/usr/java/MYBOOTJDK_1.7/:启动jdk的路径; 
–with-debug-level=slowdebug:编译时debug的级别,有release, fastdebug, slowdebug 三种级别; slowdebug 级别可以生成最多的调试信息 
–enable-debug-symbols ZIP_DEBUGINFO_FILES=0:生成调试的符号信息,并且不压缩

```

若在configure过程中提示安装工具,则在安装完工具后执行make clean进行清理方可再次configure,否则会config不成功。

运行后报错：
- 问题一：
> configure: error: Could not find all X11 headers (shape.h Xrender.h XTest.h Intrinsic.h). You might be able to fix this by running 'sudo yum install libXtst-devel libXt-devel libXrender-devel'.
configure exiting with result code 1
解决：
    yum install libXtst-devel libXt-devel libXrender-devel

搞定，配置成功：
```Build performance summary:
* Cores to use:   1
* Memory limit:   1821 MB
* ccache status:  not installed (consider installing)

Build performance tip: ccache gives a tremendous speedup for C++ recompilations.
You do not have ccache installed. Try installing it.
You might be able to fix this by running 'sudo yum install ccache'.

```
注：其实如我之前尝试过编译 jdk7等，还会有其他依赖包缺失（如cups,freetype,alsa等）

#### 三、编译
```
make all ZIP_DEBUGINFO_FILES=0
漫长等待之后，编译成功:花费了一个多小时
----- Build times -------
Start 2018-09-04 10:12:59
End   2018-09-04 11:26:19
00:04:59 corba
00:02:08 demos
00:14:19 docs
00:20:44 hotspot
00:03:22 images
00:02:24 jaxp
00:02:30 jaxws
00:16:18 jdk
00:05:14 langtools
00:01:20 nashorn
01:13:20 TOTAL
-------------------------
Finished building OpenJDK for target 'all'

```

#### 四、验证
通过日志或者在源码根目录下执行  find . -name java ，编译后的jdk 相对路径为：
./build/linux-x86_64-normal-server-slowdebug/jdk  ，进入 bin 路径：
```
[zyl@localhost bin]$ java -version
openjdk version "1.8.0_181"
OpenJDK Runtime Environment (build 1.8.0_181-b13)
OpenJDK 64-Bit Server VM (build 25.181-b13, mixed mode)
---------------------------------------------------------
[zyl@localhost bin]$ ./java -version
openjdk version "1.8.0-internal-debug"
OpenJDK Runtime Environment (build 1.8.0-internal-debug-zyl_2018_09_04_10_11-b00)
OpenJDK 64-Bit Server VM (build 25.0-b70-debug, mixed mode)
---默认java指向环境变量的java命令，使用 ./java则运行当前目录的java命令
```

从执行结果来看，一个是系统默认jdk8，一个是我自己编译的 jdk8 （有zyl标记及编译时间戳)

#### 五、编译：
进入自己编译的jdk所在bin目录：
```[zyl@localhost bin]$ vi Test.java
[zyl@localhost bin]$ cat Test.java
    public class Test{
    public static void main(String[] args){
        System.out.println("hello world !");
    }
    }
[zyl@localhost bin]$ ./javac Test.java 
[zyl@localhost bin]$ ./java Test
hello world !

```
喜大普奔，普天同庆。。。折腾了2周多，总算暂时编译成功了，虽然只是jdk8.
