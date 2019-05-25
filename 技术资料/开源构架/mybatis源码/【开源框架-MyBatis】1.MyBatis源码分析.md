- 基于首次阅读框架源码尽量选择早期版本，因为更接近到原始设计思路，易于理解，支持mybatis3.0.1版本（mybatis与ibatis有较大差别，做了比较大的重写）。近期版本的mybatis源码可在github中搜索mybatis-3查找，但前期的已无github可用，在mybatis官网可找到早期mybatis的源码zip包。（已下载并上传至github:https://github.com/better-yulong/StudyNote-Resource/blob/master/StudyNote-Resource/source-zip/mybatis-3-mybatis-3.0.1.zip）。
- 解压并import至eclipse，工程各种报错，发现该工程导入build path关联的jre1.5，初步怀疑与jdk版本有关系，查看mybatis.iml文件其中提到LANGUAGE_LEVEL="JDK_1_5"，尝试将JDK切换至1.6，再次构建工程错误消失。

### 一. Mybatis 源码单元测试及入口分析
工程导入之后其实是一筹莫展，根本无从下手，考虑到工程自带 src/test/java中各种代码，比如数据表建表脚本、数据insert脚本，初步分析这应该就是用来初始化单元测试依赖环境的。但继续查看，类过多且并未找到真实可运行的测试类。无奈之举（后面发现误打误撞对了），在 src/test/java 中搜索Junit关键字，查找可运行的单元测试方法，一搜还果然不少，再结合junit关键字及select关键字，确实找到了@Test注解