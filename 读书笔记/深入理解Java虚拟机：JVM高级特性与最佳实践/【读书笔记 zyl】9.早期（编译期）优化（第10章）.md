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
- 字节码生成是Javac编译过程最后一个阶段，Javac源码则 com.sun.tools.javac.jvm.Gen类完成。字节码生成阶段不仅是把前面各阶段生成的信息（语法树、符号表）转化成字节码写到磁盘，同时还进行了少量的代码添加和转换工作。
- 如之前章节提到的实例构造器方法<init>()和类构造器方法<clinit>()即是在该阶段添加到语法树（实例构造器并不是指默认构造函数，若代码中没有提供任何构造函数，那编译喊叫则会添加一个无参、访问修饰符（public、protected或private）与当前类一致的默认构造函数，该工作在填充符号表阶段就已经完成）。这两个构造器实际上是一个代码收敛的过程，编译器会把语句块（实例构造器是"{}"块，而类构造器则是"static{}"块）、变量初始化（实例变量和类变量）、调用父类的实例构造器等操作收敛到<init>()和<clinit>()方法，并且保证一定是按先执行父类实例构造器、初始化变量、语句块的顺序执行，其在 Gen.normalizeDefs()方法来实现。另外如部分优化程序的实现逻辑，如把字符串加号替换为StringBuffer或StringBuilder（与JDK版本有关）的append()操作。
- 完成对语法树的遍历及调整后，会把填充所有所需信息的符号表交给 com.sun.tools.javac.jvm.ClassWriter类，由该类的writeClass()方法输出字节码生成最终的Class文件。

#### 三、语法糖的味道
##### 1、泛型与类型擦除
泛型是JDK1.5新增的一项特性，其本质是参数化类型（Parametersized Type)的应用，也就是说所操作的数据类型被指定为一个参数。这种参数可以用在类、接口和方法创建，分别称为泛型类、泛型接口、泛型方法。
- 泛型出现之前，只能基于Object是所有类的父类和类型强制转换配合实现泛型。如JDK1.5之前HashMap的get()方法，返回值为Object对象，然后基于强制类型转换可转换成任何对象，也同时这种无限可能性只有程序员和运行期的虚拟机才知道Objet具体是什么类型的对象（因编译期无法检查Object强制类型转换是否可成功），使得运行期会出现许多ClassCastException。
- 而C#的泛型则是在编译期、运行期需存在对应的方法表和类型数据，这种实现称为类型膨胀，基于这种方法实现的泛型称为真实泛型。而Java语言的泛型仅存在源码，编译期检查通过，编译后的字节码文件即已经将原来的原生类型（Raw Type，也称作裸类型），同时在对应地方插入强制类型转换代码。即对于运行期的Java语言来说，Array<Integer>（书中原文是Array<int>,但实际因泛型不能基本类型会报语法错）与Array<String>是对应同一个类。Java语言的泛型实现方法称为类型擦除，基于这种方法实现的泛型称为伪泛型。
```language
	如源码：
		ArrayList<Integer> list = new ArrayList<Integer>();
		list.add(Integer.valueOf(1));
		Integer temp = list.get(0);
	编译后通过反编译工具查看class文件如下，即自动通过强制类型转换实现：
		ArrayList localArrayList = new ArrayList();
    		localArrayList.add(Integer.valueOf(1));
    		Integer localInteger = (Integer)localArrayList.get(0);	
```
- 那当泛型遇到重载呢？
```language
	public static void method1_1(ArrayList<String> list){
		System.out.println("invoke method method1_1(ArrayList<String> list)");
	}
	public static void method1_1(ArrayList<Integer> list){
		System.out.println("invoke method method1_1(ArrayList<String> list)");
	}
```
在IDE或javac编译时报错：Erasure of method method1_1(ArrayList<String>) is the same as another method in type GenericTypesTest 。初步来看，因为类型擦除后两个方法完全一样导致无法重载。但实际还有另一部分原因。
```language
	public static String method1_1(ArrayList<String> list){
		System.out.println("invoke method method1_1(ArrayList<String> list)");
		return "";
	}
	public static Integer method1_1(ArrayList<Integer> list){
		System.out.println("invoke method method1_1(ArrayList<String> list)");
		return 1;
	}
	
	public static void main(String[] args){
		method1_1(new ArrayList<String>());
		method1_1(new ArrayList<Integer>());

	}
```
因IDE该测试工程的jdk版本为jdk7，IDE会直接报错（Erasure of method method1_1(ArrayList<String>) is the same as another method in type GenericTypesTest）；故通过javac命令基于JDK1.7编译，会报编译错：
```
	GenericTypesTest.java:17: 错误: 名称冲突: method1_1(ArrayList<Integer>)
和method1_1(ArrayList<String>)具有相同疑符                                
        public static Integer method1_1(ArrayList<Integer> list){
                            ^
1 个错误   

```
 而下载JDK6，使用JDK6编译、运行则均可正确，结果如下
```
	invoke method method1_1(ArrayList<String> list)
	invoke method method1_1(ArrayList<Integer> list)
```
- 对于public static Integer method1_1(ArrayList<Integer> list)、public static void method1_1(ArrayList<String> list) 两个方法，在JDK1.6 可重载成功，对以前重载仅以方法名和参数来判断而返回值不参与重载选择的认知确实是个挑战。其实JDK1.6 也不是根据返回值来确定重载，只是因为方法返回值不同才可并存在同一Class文件中。至于第6章介绍Class文件方法表（method_info）的数据结构提到的方法重载要求方法具备不同的特征签名（signature），而返回值并不在方法特殊签名但存在于方法描述。---即Class文件格式只要描述符不一致即可以共存（即使方法特征签名（方法名、参数）相同但返回值不同）。
- 对于上面method1_1泛型在JDK1.6编译运行成功，而JDK1.7直接编译不通过的问题，顺便做了个有趣的测试，即JDK1.6编译通过的Class文件是否可在JDK1.8环境运行呢？结果还真如预期，运行成功。> 段落引用 至此有了比较清楚的理解。JRE对Class文件的校验允许相同签名但符号描述不同的方法共存，即都可以运行成功。因Signature 是虚拟机规范中最重要的一项属性，它存储的是方法在字节字节码层面的特征签名，故虚拟机规范也有修改要求JDK1.7及以上版本需避免类型擦除对实际编码的影响，故在JDK1.7的Javac编译时对仅返回值不同方法描述符相同的源码在编译时做了验证；同时为了运行时兼容低版本生成的Class文件故运行时又可支持。
- 书中提到：从Signature 属性得出结论：类型擦除仅是对方法的Code属性中字节码进行擦除，而实现元数据仍保留了泛型信息（这也是为何可以通过反射取得参数化类型的要本依据）。但关于Code属性字节码类型擦除而元数据保留泛型信息一开始并未理解，故决定深究，先看下面几段代码：
```language
源码：
	public class GenericTypesTest2<T> {
	
	
		public <A> void testGenericTypeClear(A a,T b){
			System.out.println(a.toString());
			ArrayList<Integer> list0 = new ArrayList<Integer>();
			Integer temp0 = list0.get(0);  
			ArrayList<T> list = new ArrayList<T>();
			T temp = list.get(0);  //编译后反编译Class文件实际就是对get返回的Object对象做的强制转换成Integer对象。
		}
	
		public void testGenPass(T t){
			System.out.println(t.hashCode());
		}

	}
```
```language
反编译Class文件后代码：
	public class GenericTypesTest2<T>
	{
  		public <A> void testGenericTypeClear(A paramA, T paramT)
  		{
    			System.out.println(paramA.toString());
    			java.util.ArrayList localArrayList1 = new java.util.ArrayList();
    			Integer localInteger = (Integer)localArrayList1.get(0);
    			java.util.ArrayList localArrayList2 = new java.util.ArrayList();
    			Object localObject = localArrayList2.get(0);
  		}
  
  		public void testGenPass(T paramT) {
   			 System.out.println(paramT.hashCode());
  		}
	}
```
```language
Bytecode viewer查看class文件二进制：
testGenericTypeClear 方法Signature值：<<A:Ljava/lang/Object;>(TA;TT)v> 
```
方法class格式见图：
![方法class格式见图](https://github.com/better-yulong/StudyNote-picture/blob/master/StudyNote-picgure/10-001.PNG)
- 通过如上GenericTypesTest2类源码及反编译、二进制示例相对理解比较清晰，泛型即为类型参数化。即然是参数，那么就涉及到参数的声明、作用域、类型信息。而泛型就可以理解为可以动态指定实际类型的参数。对于很多地方理解的如泛型接口、泛型类、泛型方法的分类个人感觉只不过是泛型参数声明位置、使用域有区别布局，且并非排它而是可组合。
1. 泛型声明：泛型作为区别与Java运行存在且支持的类型，泛型对象及泛型集合对象声明使用<And>，如List<A>  或 A a。再就是声明的位置只能是在class 类名、interface接口名之后，或者方法的修改符如private之后返回参数之前。同时若需同时声明多个泛型，可在<>内逗号分隔。复杂点示例如下：
```language
public class GenericTypesTest2<T,M,N> {
	
	private T t ;
	
	public <A> T testGenericTypeClear(A a,ArrayList<T> b,ArrayList<A> c,T d){
		System.out.println(a.toString());
		ArrayList<Integer> list0 = new ArrayList<Integer>();
		Integer temp0 = list0.get(0);  
		ArrayList<T> list = new ArrayList<T>();
		T temp = list.get(0);  //编译后反编译Class文件实际就是对get返回的Object对象做的强制转换成Integer对象。
		return temp;
	}
	
	public void testGenPass(T t){
		ArrayList<N> list = new ArrayList<N>();
		System.out.println(list.hashCode());
		
	}

}
```
2. 泛型作用域：因泛型不同于已知类型，而变量通常包含类型、参数名。故规范确认为<A>声明泛型A,而其作用域与实例变量、局部变量、方法入参相同。即定义在class、interface 关键字后的泛型可使用于实例变量、方法入参、方法内、方法返回值；而定义在方法的泛型使用域则为方法入参、方法内、方法返回值 。重要的一点：因泛型是动态类型，故泛型参数均不能使用static修饰，但可使用final修饰。
3. 这点即是Code属性类型擦除但元数据保留泛型信息的理解：元数据保存在JVM的方法区或元数据区，而每个Class的Objct对象则保存在堆（对应元数据区类的无数据信息）。那Code属性和元数据区究竟保存了什么呢？先看看的Class文件结构：
![Class文件结构](https://github.com/better-yulong/StudyNote-picture/blob/master/StudyNote-picgure/10-002.PNG)
Class文件结构：基本信息（魔树、主次版本等）、Class常量池、接口列表Interfaces、Fields数组（实例变量、类变量，其中实例变量也有Signature属性）、Methods数组（各方法又包含Code属性、Signature属性）、Attributes数组（类的Signature属性、源java文件名）
Code属性详细说明是每个method的Code属性，而这个Code属性仅是包含源码方法{}内的代码（也可认为即是方法内部代码编译后的JAVA操作指令。），而在{}内部使用的泛型相关代码则只能使用Object替换（声明在接口、类、方法上）或者强制类型转换（如方法内容声明List<Integer>并调用get方法时）根据之前章节可知道Code属性可认为即是方法内部代码编译后的JAVA操作指令。
那元数据又怎么理解呢？其实可以得意理解为Class的骨架信息，或者对我们而言就是可以通过反射机制操作的数据（Constructor类、Field类、Method类）。那接口、类字义的泛型信息在Attributes数组的Signature属性会保留，而方法的泛型信息则在method的Signature属性保留）。
4. 那么更进一步，涉及到类与类型？注意区分类型（Type）与类（Class）的区别，这里Class是Type的一种是Type的子集，直接子类只有一个也就是Class，代表着类型中的原始类型以及基本类型，而像数组、枚举等“类型”是相对于Class来说。如method.getGenericReturnType() 可以理解获取源码定义的类型信息（如泛型A），method.getReturnType() 则为类型擦除后的类型信息（如class java.lang.Object)。
##### 2、自动装箱、拆箱与遍历循环
先来看段示例代码：
```
	public class AutoBoxTest {

		public static void main(String[] args) {
			Integer a = 1 ; 
			Integer b = 2 ;
			Integer c = 3 ;
			Integer d = 3 ;
			Integer e = 321 ;
			Integer f = 321 ;
			Long g= 3l;
			Integer h = new Integer(3);
			System.out.println(c==d);//true
			System.out.println(e==f);//false
			System.out.println(c==(a+b));//true
			System.out.println(c.equals(a+b));//true
			System.out.println(g==(a+b));//true
			System.out.println(g.equals(a+b));//false
			System.out.println(h==c);//false	
		}
	}
```
如上运行结果理解需要依赖几点：
1. Integer、Byte、Short、Integer、Long类都有缓存的范围，其中Byte，Short，Integer，Long为 -128 到 127，Character范围为 0 到 127；除了 Integer 可以通过参数改变范围外，其它的都不行（Integer的缓存上界high可以通过jvm参数-XX:AutoBoxCacheMax=size指定，取指定值与127的最大值并且不超过Integer表示范围，而下界low不能指定，只能为-128。）。缓存范围内，自动装箱时会取自常量池中的对象。
2. 包装类的“==”运算在没有遇到算术运算符的情况下不会自动拆箱、装箱。
3. equals方法直接比较值，不处理数据类型转换，即不会触发装箱。
4. new Integer()会忽略Integer缓存创建新对象，Integer.valueOf()则会优化取缓存数据。
5. 自动装箱、拆箱实际为自动类型转换,如 g==(a+b) 涉及基本数据类型到Integer、Long的装箱；但  new Integer(2)==new Double(2) 这种比较会直接报错：不支持的操作数运算。
6. 建议实际编码尽量避免使用自动装箱与拆箱，避免因理解不透彻不可控。
##### 3、条件编译
Javac编译器无需使用预处理器，其并非一个个编译Java文件，而是将编译单元的语法树顶级节点输入到待处理列表后再进行编译，因此各个文件之间能够互相提供符号信息。
```language
		if(true){
			System.out.println("true");
		}else{
			System.out.println("false");
		}
```
编译后反编译发现，只剩： System.out.println("true");同时IDE也会提示DeadCode。但：
```language
		while(true){ //while(true）可正常编译运行，编译后补充替换为 for (;;)
			System.out.println("while true");
		}
		
		while(false){//while(false）编译报错：
			System.out.println("while true");
		}

```


	
