#### 一、概述
编译器可以分好几类，如前端编译器把*.java文件转变成 *.class 文件，即Sun的 Javac、Eclipse JDT 的增量式编译器（ECP）；又或者如后端运行期编译器把字节码转变成机器码，如HotSpot VM的C1、C2编译器；再者如静态提前编译器（AOT编译器）把*.java文件直接编译成本地机器代码。
前端编译器的优化主要为使用使用“语法糖”来改善程序员的编码风格和提交编码效率；而针对性能的优化则集中在后台运行期编译期。

#### 二、Javac 的源码及调试
javac源码位于 JDK_SCR_HOME/langtools/src/share/classes/com/sun/tools/javac/ 目录中，除JDK自身API外，仅调用了 com/sun 包内的代码。
##### 1、下载源码及工程导入：
Centos7 安装hg：  yum install mercurial 
         下载jdk8基础包：hg clone http://hg.openjdk.java.net/jdk8u/jdk8u/
         下载源码：进入基础包，./get_source.sh  下载完整源码包（其实也不包含受保护源码，如 com.misc包），因网络不稳定，常会报：stream ended unexpectedly (got 42241 bytes, expected 53431)
故修改get_sourece.sh最后面的部分，调整循环下载直接成功为止。（仅需修改文件末尾modify 片段）
 ```
	######### modify start by zyl ,to download until succeed.
	# Get clones of all absent nested repositories (harmless if already exist)
	sh ./common/bin/hgforest.sh clone "$@"
	while [ $? -ne 0 ]
	do
		sh ./common/bin/hgforest.sh clone "$@"
	done

	# Update all existing repositories to the latest sources
 	sh ./common/bin/hgforest.sh pull -u
	while [ $? -ne 0 ]
	do
		sh ./common/bin/hgforest.sh pull -u
	done

	######### modify end ,to download until succeed.

```

新建compiler_javac 的 Java工程并新建com.sun pacakge，导入langtools/src/share/classes/com/sun 包下所有目录及文件（就3个文件夹：javadoc、source、tools）。（书中说会因为代码访问规则导致 AnnotationProxyMaker 报错，但我本地未报错。因我的本地环境默认无规则）
 
虚拟机规范严格定义了Class文件格式，但对于把Java源码转变为Class编译过程并未严格定义，使得Class文件的编译与JDK具体实现相关，极端情况会出现同样源码Javac可编译通过但ECJ编译器无法编译（后续章节讲述）
从Sun Javac代码来看，核心是JavaCompiler.compile(fileObjects, classnames.toList(), processors)：
可参考：【读书实践 zyl】10. Javac 实现研究
核心代码：
```try {
            initProcessAnnotations(processors);   //准备过程：初始化插入式注解处理器

            // These method calls must be chained to avoid memory leaks
            delegateCompiler =
  		    processAnnotations(   //过程2：执行注解处理
                       enterTrees(stopIfError(CompileState.PARSE, 			
                           parseFiles(sourceFileObjects))),
                    classnames);
	    //parseFiles，过程1.1：词法分析、语法分析
	    //enterTrees，过程1.2：输入到符号表

  	    delegateCompiler.compile2();  //过程3：分析及字节码生成
            delegateCompiler.close();
            elapsed_msec = delegateCompiler.elapsed_msec;
	注：过程3内部核心代码：generate(desugar(flow(attribute(todo.remove()))));   

```
过程3说明：过程3.1 attribute方法：标注；过程3.2 flow方法：数据流分析；过程3.3 desugar方法：解语法糖；过程3.4 generate方法：生成字节码

##### 2、解析与填充符号表
解析步骤由如上代码片段的 parseFiles()方法，即过程1.1完成 ，解析过程包括经典编译原理中的词法分析和语法分析。
###### a、词法、语法分析
词法分析是将源代码的字符流转变化标记（Token）集合，单个字符是程序编写过程的最小元素，而标记则是编译过程的最小元素，关键字、变量名、运算符都可以成为标记。如”int a=b+2“这句代码则包含了6个标记，分别是int、a、=、b、+、2。而关键字 int 由3个字符构成，但它不可拆分，其只是一个Token。词法分析过程由 com.sun.tools.javac.parser.Scanner类来实现，也叫做扫描 scanner.
核心代码Scanner的实现为Scanner类的nextToken()中的间接引用的 JavaTokenizer 为的 readToken() 方法，其中即为逐行逐字符（真的是字符，如数字、大小写字母、空格、换行、美元、下划线）参考TokenKind类型（包含java中关键字、操作运算符列表、@符号、分号、逗号、各种括号等等，以此为分隔符来拆分Token）匹配并确认并设置每个Token的类型来将字符流转化为标记（Token)集合。通过源码，发现可通过调整 scannerDebug参数来显示词法分析的debug日志。
语法分析是根据Token序列构造抽象语法树的过程，抽象语法树(Abstract Syntax Tree，AST）是一种用来描述程序代码语法结构的树形表示方式，语法树的第一个节点都程序代码中的一个语法结构（Construct），例如包、类型、修饰符、运算符、接口、返回值甚至代码注释都是一个语法结构），而语法分析过程则由 com.sun.tools.javac.parser.Parser 类实现，该阶段产生的抽象语法树由com.sun.tools.javac.tree.JCTree类表示。
经过词法分析、语法分析产生抽象语法树后，编译器即不再对源码文件进行操作，后续操作均建立在抽象语法树之上。
基于eclipse AST 插件显示示例部分代码的抽象语法树视图，可直接认识：
 
核心代码parseFiles 的实现为为JavacParser类的parser方法---> JavacParser.parseCompilationUnit 方法：1、parser方法解析是否保留注册、debug调试行等参数初始化；2、parseCompilationUnit 方法则根据标记Token的类型循环遍历填充完成 ListBuffer<JCTree>。
###### b、填充符号表
词法分析、语法分析之后即是填充符号表，对应 enterTrees 方法间接依赖的Enter类的complete方法。符号表（Synbol Table）是由一组符号地址和符号信息构成的表格（数据结构可以是哈希表、有序符号表、树状符号表、栈结构符号表等）
符号表可用于语义检查、生产中间代码、地址分配等。
##### 3、注解处理器
JDK 1.5 之后，Java语言提供了对注解（Annotation）的支持，即一组插入式注解处理器的标准API，可读取、修改、添加抽象语法树中的任意元素。语法树的每次修改均需重新加到解析及填充符号表的过程重新处理，每次循环称为Round，即回环过程。因其可直接干涉编译器的行为并操作任意元素，故基于插入式注解处理器实现的插件可有很大发挥。
Javac源码，插入式注解处理器的初始化过程是在 initProcessAnnotations() 方法完成，而它的执行过程则是 processAnnotations()方法完成。
##### 4、语义分析与字节码生成
语法树能表示一个结构正确的源程序的抽象，但无法保证其符合逻辑。而语义分析主要任务是对结构正确的源程序进行上下文性质的审查，如类型审查。而语义分析过程分为标注检查(attribute方法)以及数据及控制流分析(flow方法)两个步骤。
###### a、标注检查(attribute方法)
标注检查包括诸如变量使用前是否已被声明、变量与赋值之间的数据类型是否匹配等，而其中还有一个重要的动作被称为常量折叠。常量折叠如  int a = 1+2，该行代码对应的插入式表达式会在语法树上直接标注为3，所以代码中的 int a =1+2  与 int a=3 在执行时没有区别。标注检查的实现在 com.sun.javac.comp.Attr 类和 com.sun.tools.javac.comp.Check 类。
###### b、数据及控制流分析
- 数据及控制流分析是对程序上下文逻辑更进一步的验证，其可检查出诸如程序局部变量在使用前是否有赋值、方法的每条路径是否都有返回值、是否所有的受查异常都被正确处理等。编译时期的数据及控制流分析与类加载时间的数据及控制流分析目的基本一致，但校验范围有所区别，有一些校验只能在编译期或者运行期才能确定。以方法内final局部变量为例：
```language
	//方法1
	public void foo(final int arg){
		final int var = 0 ;
		//do something
		//arg = 3 ;  ++L1++
		//var = 3 ;  ++L2++
	}
	
	//方法2
	public void foo(int arg){
		int var = 0 ;
	}
```
1. 理解1：L1、L2两行代码若去年注释在IDE或javc编译时会直接报错，因final的局部变量不可再被赋值（或修改对象引用）
2. 理解2：局部变量与字段（实例变量、类变量）不同，字段会在Class常量池中有CONSTANT_Fieldref_info的符号引用及访问标记(Access_Flags)，而局部变量没有且其编译时可能连名称也不会保留（编译时可代码混淆）。但实际方法1、方法2编译后的Class文件完全相同，区别仅在于final修改的局部变量编译期会在语义分析阶段检查是否有被重新赋值或修改引用。即局部变量final修饰是编译期类型保障，不影响运行期。
- 数据及流程控制分析的入口是flow()方法，具体类为 com.sun.tools.javac.comp.Flow 完成。
###### c、解语法糖
- 语法糖（Syntactic Sugar)也称糖衣语法，主要为增加程序可读性，从而减少程序代码出错的机会。而Java中最常用的语法糖主要是泛型（不同语言有差异，如C#的泛型则不一定是语法糖实现）、变长参数、自动装箱/拆箱。
- 语法糖在编译阶段即被还原回简单的基础语法结构，该过程称为解语法糖；故语法糖仅编译支持，运行时不支持。对应过程由 desugar()方法触发，在 com.sun.tools.javac.comp.TtransTypes类和 com.sun.tools.javac.comp.Lower类完成。
###### d、字节码生成
字节码生成是Javac编译过程最后一个阶段，Javac源码则 com.sun.tools.javac.jvm.Gen类完成。字节码生成阶段不仅是把前面各阶段生成的信息（语法树、符号表）转化成字节码写到磁盘，同时还进行了少量的代码添加和转换工作。
- 如之前章节提到的实例构造器方法<init>()和类构造器方法<clinit>()即是在该阶段添加到语法树（实例构造器并不是指默认构造函数，若代码中没有提供任何构造函数，那编译喊叫则会添加一个无参、访问修饰符（public、protected或private）与当前类一致的默认构造函数，该工作在填充符号表阶段就已经完成）。这两个构造器实际上是一个代码收敛的过程，编译器会把语句块（实例构造器是"{}"块，而类构造器则是"static{}"块）、变量初始化（实例变量和类变量）、调用父类的实例构造器（仅是