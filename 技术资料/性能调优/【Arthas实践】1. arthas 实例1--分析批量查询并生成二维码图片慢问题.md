背景：
    基于业务需求，需批量查询500条数据并生成二维码图片（含logo图片、head文字、tail文字）压缩包，本地验证500条需10秒左右，分析看是否有进一步优化空间。

### 一. Arthas环境
1. 基于 https://alibaba.github.io/arthas/download.html   下载 arthas-boot-3.2.0.jar；然而执行 java -jar arthas-boot-3.2.0.jar ，提示错误：arthas-boot-3.2.0.jar中没有主清单属性 
2. 之后参考： https://www.jianshu.com/p/4e711a780aa3  从地址 https://alibaba.github.io/arthas/arthas-boot.jar 下载arthas-boot.jar 后执行 java -jar arthas-boot.jar 运行成功。

### 二.实践

d:\>java -jar arthas-boot.jar
[INFO] arthas-boot version: 3.2.0
[INFO] Found existing java process, please choose one and input the serial number of the process, eg : 1. Then hit ENTER.
* [1]: 14132 com.***.Application
  [2]: 8980 org.springframework.ide.vscode.boot.app.BootLanguagServerBootApp
  [3]: 9028 D:\Program
  [4]: 11980 D:\work\ide\eclipse\\plugins/org.eclipse.equinox.launcher_1.5.400.v20190515-0925.jar



