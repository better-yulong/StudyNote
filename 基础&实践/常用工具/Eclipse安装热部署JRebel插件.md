参考资料：https://www.cnblogs.com/new-life/p/9265530.html

### 一.Eclipse安装JRebel
在Eclipse的Eclipse Marketplace中查找JRebel插件，正常安装，稍等片刻即可，重启Eclipse

### 二.JRebel激活
- JRebel需要激活码激活（激活参考：https://www.e-learn.cn/content/qita/2354778）。重启Eclipse之后进入JRebel Configuration （或通过show view查找 或者在Open Perspective中查找）；从https://github.com/ilanyu/ReverseProxy/releases（ReverseProxy_windows_amd64.exe
)下载地址激活软件，并启动（关注端口号）。
- 在Jrebel Activation Details页面选择"Team URL"：
  1. 其中Url输入： http://127.0.0.1:8888/574ac103-205f-4fe9-af26-06f9ad914c3d （注意574ac103-205f-4fe9-af26-06f9ad914c3d为Version 4 UUID，因多人使用有可能过期，可在https://www.uuidgenerator.net/ 页面重新重头并替换）
  2. 邮箱随便填写，点击Change license即可完成激活。 

### 三.Eclipse使用JRebel
Eclipse每次启动后，默认会自动检查JRebel证书，即上面的ReverseProxy本地证件代理每次均需启动，这就是使用破解版的代价哈；之后进入JRebel--JRebel Configuration界面参考：https://www.jb51.net/softjc/614942.html 完成项目热部署的配置。
1. 点击Startup,选择Run via IDE，Servers选择你配置的tomcat；
2. 点击Projects,选择需要JRebel监测的项目; 
3. 点击Advanced，勾选Miscellaneous下的所有选项（之后会自动重启eclipse）



