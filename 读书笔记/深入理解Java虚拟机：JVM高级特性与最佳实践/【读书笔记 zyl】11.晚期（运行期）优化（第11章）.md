
#### 一、概述
> Java程序最初是通过解释器（Interpreter）解释执行，当虚拟机发现某方法或代码运行特别频繁，会把该部分代码认定为”热点代码“(Hot Spot Code）。为提供热点代码执行，运行时虚拟机会将这些代码编译成本地平台相关的机器码并进行各种上层次的优化，完成该任务的编译器称为即时编译器（Just In Time Compiler），简称不JIT编译器。
> 即时编译器非Java虚拟机规范所要求必须存在，更没有要求如何实现。但即时编译器性能好坏、代码优化程度则是衡量虚拟机优秀与否最关键的指标，它也是虚拟机最核心且最能体现技术水平的部分。

#### 二、HotSpot虚拟机内的限时编译器
带着如下问题去了解限时编译器的动作过程：
- 为何HotSpot虚拟机要使用解释器与编译器并存的架构？
- 为何HotSpot虚拟机要实现两个不同的即时编译器？
- 程序何是使用解释器执行？何时使用编译器执行？
- 哪些程序代码会被编译为本地代码？如何编译为本地代码？
- 如何从外部观察即时编译器的编译过程和编译结果？
##### 1、解释器和编译器
- 并非所有Java虚拟机都采用解释器和编译器并存架构，但主流商用虚拟机如HotSpot、J9等均同理包含解释器和编译器（JRockit面向服务端，其无解释器使得启动响应时间长）。解释器和编译器各有优势：程序需迅速启动和执行，则解释器可节省编译时间立即执行；运行期，编译器将热点代码编译成本地代码可获得更高的执行效率但内存资源消耗则会同步增加。另解释器与编译器涉及到激进优化相关需配合工作。
- HotSpot虚拟机内置两个限时编译器，分别称为Client Compiler和 Server Compiler，即简称为C1编译器和C2编译器（也叫Opto编译器）。主流HotSpot虚拟机默认采用解释器与其中一种编译器配合工作。程序使用的编译器取决于虚拟机运行模式，HotSpot虚拟机会根据自身版本及宿主机器的硬件性能自动选择，或者用户也可使用 "-client" 或 "-server" 参数强制指定运行在Client模式或者Server模式。
- 解释器和编译器搭配使用的方式在虚拟机中称为"混合模式"(Mixed Mode),用户可使用参数"-Xint"强制虚拟机运行于"解释模式"（Interpreted Mode），此模式编译器不工作，全部代码都使用解释方式执行。另外，也可使用参数"-Xcomp"强制虚拟机运行于编译模式（Compiled Mode),该模式优化采用编译方式运行，但若编译方式无法进行则解释器需介入执行。
```
	D:\>java -version
	java version "1.8.0_112"
	Java(TM) SE Runtime Environment (build 1.8.0_112-b15)
	Java HotSpot(TM) 64-Bit Server VM (build 25.112-b15, ++mixed mode++)

	D:\>java -Xint -version
	java version "1.8.0_112"
	Java(TM) SE Runtime Environment (build 1.8.0_112-b15)
	Java HotSpot(TM) 64-Bit Server VM (build 25.112-b15, ++interpreted mode++)

	D:\>java -Xcomp -version
	java version "1.8.0_112"
	Java(TM) SE Runtime Environment (build 1.8.0_112-b15)
	Java HotSpot(TM) 64-Bit Server VM (build 25.112-b15, ++compiled mode++)
```
即时编译器编译本地代码需占用程序运行时间，且依赖解释器收集性能监控信息以便编译出优化程序更高的代码。为在程序启动响应速度与运行效率之间达到平衡，HotSpot虚拟机采用逐渐启用分层编译（Tiered Compilation)策略，其在JDK 1.7的Server模式首次作为默认编译器被开启。分层编译可划分为3个层次：
1. 第0层：程序解释执行，解释器不开启性能监控功能（Profiling),可触发第1层编译。
2. 第1层：也称为C1编译，将字节码编译为本地代码，进行简单、可靠优化；若有必要将加入性能监控逻辑。
3. 第2层：也称为C2编译，将字节码编译为本地代码，但会启用一些编译耗时较长的优化，甚至会根据监控信息进行不可靠的激进优化。

##### 2、编译对象与触发条件
> 热点代码：
1. 被多次调用的方法：方法调用多，方法体内执行次数多。由方法调用触发编译，因此整个方法作为编译对象，这种编译为虚拟机标准的JIT编译方式。
2. 被多次执行的循环体：方法调用较少，但方法内部循环体执行次数多。由循环体触发编译，但依然以整个方法（而不是单独的循环体）作为编译对象，这种编译发生于方法执行过程，故形象地称之为栈上替换（On Stack Replacement,简称OSR编译）
> 判断代码是否为热点代码及是否需触发即时编译，该行为称之为热点探测（Hot Spot Detection)，目前主要的热点探测判定方法有两种：
- 基于采样的热点探测（Sample Based Hot Spot Detection):采用这种方法的虚拟机会周期性检查各线程的栈顶，如果发现某（或某些）方法经常出现在栈顶，则认为该方法为"热点方法"。基于采样的热点探测好处是实现简单、高效，且可很容易获取方法调用关系（将调用堆栈展开即可）；缺点则是很难精确的确认一个方法的热度，容易受到线程阻塞或其他外界因素的影响而扰乱热点探测。
- 基于启动项器的热点探测（Counter Based Hot Spot Detection):采用这种方法的虚拟机会为每个方法（甚至是代码块）建立计数器，统计方法的执行次数。若执行次数超过一定的阈值即认为该方法为"热点方法"。这种统计方式实现成本高，需要为每个方法建立并维护计数器，且不能直接获取方法的调用关系；但是其统计相对来说更精确和严谨。
> HotSpot虚拟机采用基于计数器的热点探测方法，因此它为每个类准备了两个计数器：方法调用计数器（Invocation Counter)和回边计数器(Back Edge Counter，循环体内循环代码的执行次数)。两个计数器拥有一个确定的阈值，当计数器超过该阈值，就会触发JIT编译。默认阈值在Client模式是1500次，Server模式是10000次，阈值可通过虚拟机参数-XX:CompileThreshold 来设置。当一个方法被调用时，会先判断是否存在被JIT编译过的版本，若存在则优先使用编译后的本地代码执行；若不存在，则此方法的调用计数器加1，然后判断方法调用计数器与回边计数器值之和是否超过方法调用计数器阈值。若已超过，则向即时编译器提交一个该方法的代码编译请求。
- 如果不做任何设置，执行引擎并不会同步等待编译请求完成，而是继续进入解释器按照解释方式执行字节码，直接提交的请求被编译器完成。编译完成后，这个方法的调用入口地址会被系统自动改写成新的地址，下一次调用就会使用已编译的版本。整个JIT编译的交互过程如图：
![JIT编译交互](https://github.com/better-yulong/StudyNote-Resource/blob/master/StudyNote-Resource/11-001.PNG)
- 如果不做任何设置，方法调用计数器并非该方法被调用的绝对次数，而是一段时间之内方法被调用的次数。当超过一定时间限度，如果方法的调用次数仍不足以提交即时编译器编译，那这个方法的调用计数器会被减少一半，该过程称为方法调用计数器热度的衰减（Counter Decay）。而这段时间则称之为方法的半衰周期（Counter Half Life Time）。执行衰减的动作是虚拟机在垃圾收集时顺便进行，可使用虚拟机参数 -XX:-UseCounterDecay来关闭热度衰减，让方法计数器统计方法调用的绝对次数；那么只要程序运行时间足够长，绝大部分方法都会被编译成本地代码。另外，也可使用-XX:CounterHalfLifeTime 参数设置半衰周期时间，单位是秒。
- 回边计数器，作用是编译一个方法中循环体代码执行的次数，字节码中遇到控制流向后跳转的指令称为"回边"（Back Edge），目的是为了触发OSR编译。虚拟机虽有提供-XX:BackEdgeThreshold供用户设置但实际虚拟机并非使用，需设置另外的参数-XX:OnStackReplacePercentage来间接调整回边计数器的阈值，其计算公式：
> 虚拟机运行在Client模式，回边计数器阈值计算公式：
方法调用计数器阈值（Compile Threshold)*OSR比率（OnStackReplacePercentage）/100，其中OnStackReplacePercentage默认值为933；若都取默认值，那Client模式虚拟机的回边计数器阈值为 13995。
> 虚拟机运行在Server模式，回边计数器阈值计算公式：
方法调用计数器阈值（Compile Threshold)*（OSR比率（OnStackReplacePercentage）-解释器监控比率（InterpreterProfilePercentage）/100，其中OnStackReplacePercentage默认值为140,InterpreterProfilePercentage默认值为33；若都取默认值，那Client模式虚拟机的回边计数器阈值为 10700。
> 当解释器遇到一条回边指令时，会先查找将要执行的代码片段是否已有编译好的版本；若有，则优先执行已编译的代码，否则将回边计数器值加1，然后判断方法调用计数器和回边计数器之和是否超过回边计数器的阈值。超过阈值则提交一个OSR编译请求并且把回边计数器的值降低一些以便继续在解释器中执行循环，等待编译器输出编译结果。执行过程如图：
- ![OSR编译交互](https://github.com/better-yulong/StudyNote-Resource/blob/master/StudyNote-Resource/11-002.PNG)
> 与方法计数器不同，回边计数器没有计数热度衰减过程，因此该计数器统计的是方法循环执行的绝对次数。当计数器溢出，会同步把方法计数器的值也调整为溢出状态，这样下次进入该方法会执行标准编译过程。
> 上面两个图仅描述Client VM的即时编译方式，而Server VM 相对更复杂。JVM源码MethodOop.hpp（一个MethodOop对象个Java方法），其定义了Java方法在虚拟机的内存布局，其中明确定义了方法调用计数器和回边计数喊叫的位置和长度。
- JVM会自动的识别热点方法，并对它们使用方法内联优化。那么一段代码需要执行多少次才会触发JIT优化呢？通常这个值由-XX:CompileThreshold参数进行设置：使用client编译器时，默认为1500；使用server编译器时，默认为10000。但是一个方法就算被JVM标注成为热点方法，JVM仍然不一定会对它做方法内联优化。其中有个比较常见的原因就是这个方法体太大了，分为两种情况：
1. 如果方法是经常执行的，默认情况下，方法大小小于325字节的都会进行内联（可以通过** -XX:MaxFreqInlineSize=N**来设置这个大小）
2. 如果方法不是经常执行的，默认情况下，方法大小小于35字节才会进行内联（可以通过** -XX:MaxInlineSize=N **来设置这个大小）
> 我们可以通过增加这个大小，以便更多的方法可以进行内联；但是除非能够显著提升性能，否则不推荐修改这个参数。因为更大的方法体会导致代码内存占用更多，更少的热点方法会被缓存，最终的效果不一定好。
- 可通过命令：java -XX:+PrintFlagsFinal -XX:+PrintFlagsInitial   查看对应参数值 

##### 3、编译过程
默认设置下，无论是方法调用产生的编译请求还是OSR请求，虚拟机代码编译未完成前都仍将按照解释方式继续执行，而编译动作则在后台编译线程中进行。可通过参数 -XX:-BackgroundCompilation 禁止后台编译；该参数设置后，一旦达到JIT编译条件，执行线程向虚拟机提交编译请求后会等待直到编译过程完成后再开始执行编译器输出的本地代码。Service Compiler和Client Compiler编译过程不一样。
> Client Compiler 主要关注点是局部性的优化而放弃了许多耗时较长的全局优化优化手段，是一个简单快速的三段式编译器。
- 第一阶段：一个平台独立的前端将字节码构造成一种高级中间代码表示（High-Level Intermediate Representation，HIR）。HIR使用静态单分配（Static Single Assignment，SSA）的形式来代表代码值，可使得一些在HIR的构造过程之中和之后进行的优化运行更容易实现。在此之前编译器会完成一部分基础优化，如HIR之前完成方法内联（方法内联就是把调用方函数代码"复制"到调用方函数中,减少因函数调用开销的技术）、常量传播（编译优化时将能够计算出结果的变量直接替换为常量）等优化。
- 第二阶段：一个平台独立的后端从HIR中产生低级中间代码表示（Low Level Intermediate Representation，LIR），在此之前会完成HIR上完成部分优化，如空值检查消除、范围检查消除等，以便让HIR达到更高效的代码表示形式。
- 最后阶段是在平台相关的后端使用线性扫描算法（Linear Scan Registere Allocation）在LIR上分配寄存器，并在LIR上做窥也（Reephloe）优化，然后生产机器代码。
- Client Compiler大致执行过程：
- ![Client Compiler执行过程](https://github.com/better-yulong/StudyNote-Resource/blob/master/StudyNote-Resource/11-003.PNG)
> Server Compiler则是专门面向服务端的典型应用并为服务端的性能配置特别调整过的编译器，也是一个充分优化过的高级编译器。它会执行所有经典的优化动作，如无用代码消除（Dead Code Elimination）、循环展开（Loop Unrolling）、循环表达式外提(Loop Expression Hoisting)、消除公共子表达式（Common Subexpression Elimination）、常量传播（Constant Propagation）、基本块重排序（Basic Block Reordering）等；同时，还会实施Java语言特性密切相关的，如范围检查消息（Range Check Elimination）、空值检查消除（Null Check　Elimination；并非所有空值检查消除都依赖编译器优化，其中部分是在代码运行过程中自动优化）。另外，还可能根据解释器或Client　Compiler提供的性能监控信息，进行不稳定的激进优化，如守护内联（Guarded Inlining)、分支频率预测（Branch Frequency Prediction）。
> Server Compiler的寄存器分配器是一个全局图着色分配器，它可充分利用某些处理器架构（如RISC）上的大寄存器集合。以即时编译标准看，Server Compiler比较缓慢，但它的编译速度则远超过传统的静态优化编译器，相对于Client Compiler编译输出的代码以质量则有所提高，可减少本地代码的执行时间，从而抵消额外的编译时间开销，所以也有很多非服务端应用选择使用Server模式的虚拟机运行。

##### 4、查看及分析限时编译结果
> 该部分验证均使用自编译的openjdk8.
虚拟机的即时编译用户不知道也无感知，虽执行结果没影响但运行速度差别则很大。虚拟机提供了部分参数输出即时编译和某些优化手段（如方法内联）的执行状况。
该节的运行参数部分需要Debug或FastDebug版的虚拟机支持，Product版本的虚拟机无法使用。即编译时需要指定调试信息级别（–with-debug-level=slowdebug:编译时debug的级别,有release, fastdebug, slowdebug 三种级别; slowdebug 级别可以生成最多的调试信息），而我编译时使用的slowdebug级别。（本想编译后上传github，但发现该级别编译整体编译后达到3.6G，而jdk也达到560M，且复制迁移到windows之后无法正常使用）
```
public class JITTragerCondition1 {

	public static final int NUM = 1500 ;
	
	public static int doubleValue(int i){
		//
		for(int j=0;j<100000;j++){
		 int a = NUM ;
		}
		return i*2;
	}
	
	public static long calcSum(){
		long sum = 0 ;
		for(int i=0 ; i<=100; i++){
			sum += doubleValue(i);
		}
		return sum ;
	}
		
	public static void main(String[] args) {
		for(int i=0 ; i<NUM ; i++){
			calcSum();
		}

	}

}
```
编译并运行（-XX:+PrintCompilation 参数可要求虚拟机在即时编译时将编译成本地代码的方法名称打印出来）:
```
./javac  JITTragerCondition1.java
./java -XX:+PrintCompilation JITTragerCondition1
   2778    1       3       java.lang.String::equals (81 bytes)
   2842    2     n 0       java.lang.System::arraycopy (native)   (static)
   2901    3       3       java.lang.String::charAt (29 bytes)
   2939    4       3       java.lang.String::hashCode (55 bytes)
   2953    5       3       java.lang.String::length (6 bytes)
   2984    6       3       java.lang.Object::<init> (1 bytes)
   3207    7       3       java.lang.String::indexOf (70 bytes)
   3223    8       3       java.lang.Math::min (11 bytes)
   3350    9       1       java.lang.Object::<init> (1 bytes)
   3353    6       3       java.lang.Object::<init> (1 bytes)   made not entrant
   3371   10 %     3       JITTragerCondition1::doubleValue @ 2 (22 bytes)
   3414   11       3       JITTragerCondition1::doubleValue (22 bytes)
   3446   12 %     4       JITTragerCondition1::doubleValue @ 2 (22 bytes)
   3457   10 %     3       JITTragerCondition1::doubleValue @ -2 (22 bytes)   made not entrant
   3490   13       4       JITTragerCondition1::doubleValue (22 bytes)
   3494   11       3       JITTragerCondition1::doubleValue (22 bytes)   made not entrant
   3520   14       2       JITTragerCondition1::calcSum (26 bytes)
   3554   15 %     4       JITTragerCondition1::calcSum @ 4 (26 bytes)

```
通过输出，可发现calcSum、doubleValue 已经被编译；同时还可添加参数 -XX:+PrintInlining 可打印JIT编译时方法内联信息：
```
 ./java -XX:+PrintCompilation -XX:+PrintInlining JITTragerCondition1
   2035    1       3       java.lang.String::equals (81 bytes)
   2105    2     n 0       java.lang.System::arraycopy (native)   (static)
   2145    3       3       java.lang.String::charAt (29 bytes)
                              @ 18  java/lang/StringIndexOutOfBoundsException::<init> (not loaded)   not inlineable
   2178    4       3       java.lang.String::hashCode (55 bytes)
   2187    5       3       java.lang.String::length (6 bytes)
   2212    6       3       java.lang.Object::<init> (1 bytes)
   2439    7       3       java.lang.String::indexOf (70 bytes)
                              @ 66   java.lang.String::indexOfSupplementary (71 bytes)   callee is too large
   2477    8       3       java.lang.Math::min (11 bytes)
   2618    9       1       java.lang.Object::<init> (1 bytes)
   2621    6       3       java.lang.Object::<init> (1 bytes)   made not entrant
   2647   10 %     3       JITTragerCondition1::doubleValue @ 2 (22 bytes)
   2708   11       3       JITTragerCondition1::doubleValue (22 bytes)
   2741   12 %     4       JITTragerCondition1::doubleValue @ 2 (22 bytes)
   2746   10 %     3       JITTragerCondition1::doubleValue @ -2 (22 bytes)   made not entrant
   2768   13       4       JITTragerCondition1::doubleValue (22 bytes)
   2773   11       3       JITTragerCondition1::doubleValue (22 bytes)   made not entrant
   2816   14       2       JITTragerCondition1::calcSum (26 bytes)
                              @ 12   JITTragerCondition1::doubleValue (22 bytes)   inlining prohibited by policy

```
通过输出，可看到方法doubleValue()被内联编译到calcSum()方法中（与书中示例有差异，书中main方法也被编译）。从示例可看出，doubleValue()、calcSum()运行是不会再被解释执行，而doubleValue()则不会被调用因其已被内联编译到clacSum方法中了。
- 除了查看哪些方法被编译，还可进一步查看即时编译器生成的机器码，但全是0和1无法简单查看，必须反汇编成基本的汇编语言才可被阅读。可下载或编译反汇编适配器，并于运行时添加参数  -XX：+PrinterAssembly参数要求虚拟机打印编译方法的汇编代码。
- 也可使用 -XX:+PrintOptoAssembly(用于Server VM)或-XX:+PrintLIR(用于Client VM）查看比较接近最终结果的中间代码（伪汇编代码）。如：
```
[a@localhost bin]$ ./java -XX:+PrintCompilation -XX:+PrintInlining -XX:+PrintOptoAssembly JITTragerCondition1
   2435    1       3       java.lang.String::equals (81 bytes)
   2490    2     n 0       java.lang.System::arraycopy (native)   (static)
   2528    3       3       java.lang.String::charAt (29 bytes)
                              @ 18  java/lang/StringIndexOutOfBoundsException::<init> (not loaded)   not inlineable
   2565    4       3       java.lang.String::hashCode (55 bytes)
   2573    6       3       java.lang.Object::<init> (1 bytes)
   2595    5       3       java.lang.String::length (6 bytes)
   2733    7       3       java.lang.String::indexOf (70 bytes)
                              @ 66   java.lang.String::indexOfSupplementary (71 bytes)   callee is too large
   2759    8       3       java.lang.Math::min (11 bytes)
   2877    9       1       java.lang.Object::<init> (1 bytes)
   2881    6       3       java.lang.Object::<init> (1 bytes)   made not entrant
   2882   10 %     3       JITTragerCondition1::doubleValue @ 2 (18 bytes)
   2912   11       3       JITTragerCondition1::doubleValue (18 bytes)
   2940   12 %     4       JITTragerCondition1::doubleValue @ 2 (18 bytes)
{method}
 - this oop:          0x00007f0a2d5012c0
 - method holder:     'JITTragerCondition1'
 - constants:         0x00007f0a2d501058 constant pool [29] {0x00007f0a2d501058} for 'JITTragerCondition1' cache=0x00007f0a2d5014e8
 - access:            0xc1000009  public static 
 - name:              'doubleValue'
 - signature:         '(I)I'
 - max stack:         3
 - max locals:        2
 - size of params:    1
 - method size:       13
 - highest level:     3
 - vtable index:      -2
 - i2i entry:         0x00007f0a19023be0
 - adapters:          AHE@0x00007f0a280dcf68: 0xa0000000 i2c: 0x00007f0a19148520 c2i: 0x00007f0a19148659 c2iUV: 0x00007f0a1914862c
 - compiled entry     0x00007f0a19231d40
 - code size:         18
 - code start:        0x00007f0a2d5012a8
 - code end (excl):   0x00007f0a2d5012ba
 - method data:       0x00007f0a2d5015b8
 - checked ex length: 0
 - linenumber start:  0x00007f0a2d5012ba
 - localvar length:   0
 - compiled code: nmethod   2948   11       3       JITTragerCondition1::doubleValue (18 bytes)
#
#  int ( rawptr:BotPTR )
#
#r018 rsi:rsi   : parm 0: rawptr:BotPTR
# -- Old rsp -- Framesize: 32 --
#r191 rsp+28: in_preserve
#r190 rsp+24: return address
#r189 rsp+20: in_preserve
#r188 rsp+16: saved fp register
#r187 rsp+12: pad2, stack alignment
#r186 rsp+ 8: pad2, stack alignment
#r185 rsp+ 4: Fixed slot 1
#r184 rsp+ 0: Fixed slot 0
#
000   N28: #	B1 <- BLOCK HEAD IS JUNK   Freq: 1
000   	# breakpoint
      	nop 	# 11 bytes pad for loops and calls

010   B1: #	N28 <- BLOCK HEAD IS JUNK   Freq: 1
010   	# stack bang (32 bytes)
	pushq   rbp	# Save rbp
	subq    rsp, #16	# Create frame

01c   	movl    RBP, [RSI + #8 (8-bit)]	# int
01f   	movq    RDI, RSI	# spill
022   	call_leaf,runtime  OSR_migration_end
        No JVM State Info
        # 
02f   	sall    RBP, #1
031   	movl    RAX, RBP	# spill
033   	addq    rsp, 16	# Destroy frame
	popq   rbp
	testl  rax, [rip + #offset_to_poll_page]	# Safepoint: poll for GC

03e   	ret
03e

   2949   10 %     3       JITTragerCondition1::doubleValue @ -2 (18 bytes)   made not entrant
   2999   13       4       JITTragerCondition1::doubleValue (18 bytes)
{method}
 - this oop:          0x00007f0a2d5012c0
 - method holder:     'JITTragerCondition1'
 - constants:         0x00007f0a2d501058 constant pool [29] {0x00007f0a2d501058} for 'JITTragerCondition1' cache=0x00007f0a2d5014e8
 - access:            0xc1000009  public static 
 - name:              'doubleValue'
 - signature:         '(I)I'
 - max stack:         3
 - max locals:        2
 - size of params:    1
 - method size:       13
 - highest level:     3
 - vtable index:      -2
 - i2i entry:         0x00007f0a19023be0
 - adapters:          AHE@0x00007f0a280dcf68: 0xa0000000 i2c: 0x00007f0a19148520 c2i: 0x00007f0a19148659 c2iUV: 0x00007f0a1914862c
 - compiled entry     0x00007f0a19231d40
 - code size:         18
 - code start:        0x00007f0a2d5012a8
 - code end (excl):   0x00007f0a2d5012ba
 - method data:       0x00007f0a2d5015b8
 - checked ex length: 0
 - linenumber start:  0x00007f0a2d5012ba
 - localvar length:   0
 - compiled code: nmethod   3003   11       3       JITTragerCondition1::doubleValue (18 bytes)
#
#  int ( int )
#
#r018 rsi   : parm 0: int
# -- Old rsp -- Framesize: 32 --
#r191 rsp+28: in_preserve
#r190 rsp+24: return address
#r189 rsp+20: in_preserve
#r188 rsp+16: saved fp register
#r187 rsp+12: pad2, stack alignment
#r186 rsp+ 8: pad2, stack alignment
#r185 rsp+ 4: Fixed slot 1
#r184 rsp+ 0: Fixed slot 0
#
abababab   N1: #	B1 <- B1  Freq: 1
abababab
000   B1: #	N1 <- BLOCK HEAD IS JUNK   Freq: 1
000   	# stack bang (32 bytes)
	pushq   rbp	# Save rbp
	subq    rsp, #16	# Create frame

00c   	movl    RAX, RSI	# spill
00e   	sall    RAX, #1
010   	addq    rsp, 16	# Destroy frame
	popq   rbp
	testl  rax, [rip + #offset_to_poll_page]	# Safepoint: poll for GC

01b   	ret
01b

   3004   11       3       JITTragerCondition1::doubleValue (18 bytes)   made not entrant
   3022   14       2       JITTragerCondition1::calcSum (26 bytes)
                              @ 12   JITTragerCondition1::doubleValue (18 bytes)   inlining prohibited by policy
   3048   15 %     4       JITTragerCondition1::calcSum @ 4 (26 bytes)
                              @ 12   JITTragerCondition1::doubleValue (18 bytes)   inline (hot)

```
对于反汇编伪代码，pudouct版本的虚拟机也可打印：java -XX:+UnlockDiagnosticVMOptions -XX:+PrintCompilation -XX:+PrintInlining  com/test/jvm/jit/JITTragerCondition1 ,（仅参数：-XX:+UnlockDiagnosticVMOptions 无效，需三个参数配置使用）
```
----该示例使用官网下载标准版本的jdk验证：
java -XX:+UnlockDiagnosticVMOptions -XX:+PrintCompilation -XX:+PrintInlining  com/test/jvm/jit/J
ITTragerCondition1
    519    1       3       java.lang.String::equals (81 bytes)
    522    2       3       java.lang.Object::<init> (1 bytes)
    524    3       3       java.lang.Character::toLowerCase (9 bytes)
                              @ 1   java.lang.CharacterData::of (120 bytes)   callee is too large
                              @ 5   java.lang.CharacterData::toLowerCase (0 bytes)   no static binding
    527    4       3       java.lang.CharacterData::of (120 bytes)
    528    7       3       java.lang.String::indexOf (70 bytes)
                              @ 66   java.lang.String::indexOfSupplementary (71 bytes)   callee is too large
    531   11     n 0       java.lang.System::arraycopy (native)   (static)
    531   10       1       java.lang.Object::<init> (1 bytes)
    535    2       3       java.lang.Object::<init> (1 bytes)   made not entrant
    536    8       3       java.lang.String::hashCode (55 bytes)
    537    5       3       java.lang.CharacterDataLatin1::toLowerCase (39 bytes)
                              @ 4      539   12       4       java.util.TreeMap::parentOf (13 bytes)
    541   14       4       java.lang.String::length (6 bytes)
 java.lang.CharacterDataLatin1::getProperties    544   13       4       java.lang.String::charAt (29 bytes)
 (11 bytes)
    546    6       3       java.lang.CharacterDataLatin1::getProperties (11 bytes)
    548    9       3       java.lang.Math::min (11 bytes)
    560   15       3       java.lang.Character::toUpperCase (6 bytes)
                              @ 1   java.lang.Character::toUpperCase (9 bytes)
                                @ 1   java.lang.CharacterData::of (120 bytes)   callee is too large
                                @ 5   java.lang.CharacterData::toUpperCase (0 bytes)   no static binding
    565   16       3       java.lang.Character::toUpperCase (9 bytes)
                              @ 1   java.lang.CharacterData::of (120 bytes)   callee is too large
                              @ 5   java.lang.CharacterData::toUpperCase (0 bytes)   no static binding
    572   18       3       java.lang.AbstractStringBuilder::ensureCapacityInternal (27 bytes)
                              @ 17   java.lang.AbstractStringBuilder::newCapacity (39 bytes)   callee is too large
                              @ 20   java.util.Arrays::copyOf (19 bytes)
                                @ 11   java.lang.Math::min (11 bytes)
                                @ 14   java.lang.System::arraycopy (0 bytes)   intrinsic
    581   19       3       java.lang.AbstractStringBuilder::append (29 bytes)
                              @ 7   java.lang.AbstractStringBuilder::ensureCapacityInternal (27 bytes)
                                @ 17   java.lang.AbstractStringBuilder::newCapacity (39 bytes)   callee is too large
                                @ 20   java.util.Arrays::copyOf (19 bytes)
                                  @ 11   java.lang.Math::min (11 bytes)
                                  @ 14   java.lang.System::arraycopy (0 bytes)   intrinsic
    640   21       3       java.io.WinNTFileSystem::isSlash (18 bytes)
    641   22  s    3       java.lang.StringBuffer::append (13 bytes)
                              @ 7   java.lang.AbstractStringBuilder::append (29 bytes)
                                @ 7   java.lang.AbstractStringBuilder::ensureCapacityInternal (27 bytes)
                                  @ 17   java.lang.AbstractStringBuilder::newCapacity (39 bytes)   callee is too large
                                  @ 20   java.util.Arrays::copyOf (19 bytes)
                                    @ 11   java.lang.Math::min (11 bytes)
                                    @ 14   java.lang.System::arraycopy (0 bytes)   intrinsic
    673   23 %     3       com.test.jvm.jit.JITTragerCondition1::doubleValue @ 2 (18 bytes)
    675   20       3       java.lang.StringBuilder::append (8 bytes)
                              @ 2   java.lang.AbstractStringBuilder::append (29 bytes)
                                @ 7   java.lang.AbstractStringBuilder::ensureCapacityInternal (27 bytes)
                                  @ 17   java.lang.AbstractStringBuilder::newCapacity (39 bytes)   callee is too large
                                  @ 20   java.util.Arrays::copyOf (19 bytes)
                                    @ 11   java.lang.Math::min (11 bytes)
                                    @ 14   java.lang.System::arraycopy (0 bytes)   intrinsic
    686   24       3       com.test.jvm.jit.JITTragerCondition1::doubleValue (18 bytes)
    687   17       3       java.lang.CharacterDataLatin1::toUpperCase (53 bytes)
    688   25 %     4       com.test.jvm.jit.JITTragerCondition1::doubleValue @ 2 (18 bytes)
                              @ 4   java.lang.CharacterDataLatin1::getProperties (11 bytes)
    707   23 %     3       com.test.jvm.jit.JITTragerCondition1::doubleValue @ -2 (18 bytes)   made not entrant
    742   26       4       com.test.jvm.jit.JITTragerCondition1::doubleValue (18 bytes)
    745   24       3       com.test.jvm.jit.JITTragerCondition1::doubleValue (18 bytes)   made not entrant
    747   27       3       com.test.jvm.jit.JITTragerCondition1::calcSum (26 bytes)
                              @ 12   com.test.jvm.jit.JITTragerCondition1::doubleValue (18 bytes)   inlining prohibited by policy
    750   28 %     4       com.test.jvm.jit.JITTragerCondition1::calcSum @ 4 (26 bytes)
                              @ 12   com.test.jvm.jit.JITTragerCondition1::doubleValue (18 bytes)   inline (hot)
```
如若还想进一步跟踪本地代码生成的具体过程，可基于debug版本虚拟机使用 -XX:PrintIdealGraphFile=ideal.xml -XX:PrintIdealGraphLevel=2 （Server模式，Client模式参数不一样）参数将编译过程各阶段数据输出到文件（
./java -XX:+PrintCompilation -XX:+PrintInlining -XX:+PrintOptoAssembly -XX:PrintIdealGraphFile=ideal.xml -XX:PrintIdealGraphLevel=2 JITTragerCondition1 ），之后基于工具 Ideal Graph Visualizer 可具体查看（下载，配置jdk路径,打开ideal.xml文件即可）进行分析（即使是图形化界面，仍然较难理解，此处跳过）。

#### 三. 编译优化技术
Java虚拟机在JDK1.3及之后几乎把对代码的所有优化措施集中在即时编译器之中，一般来说，即时编译器生产的代码会比Javac 产生的字节码更加优秀。
##### 1. 优化技术概览
包含经典编译器优化，也有针对Java语言本身进行的优化,挑选部分重要且经典列举：
- 编译器策略：延迟编译，分层编译，栈上替换，延迟优化，程序依赖图表示，静态单赋值表示。
- 基于性能监控的优化技术：乐观空值断言，乐观类型断言，乐观类型增强，乐观数组增强，裁剪未被选择的分支，乐观的多态内联。分支频率预测，调用频率预测
- 基于证据的优化技术：精确性推断，内存值推断，内存值跟踪，常量折叠，重组，操作符退化，空值检查消除。类型检测退化，类型检测消除，代数化简，公共子表达式消除
- 数据流敏感重写：条件常量传播，基于六承载的类型缩减转换，无用代码消除
- 语言相关的优化技术：类型继承关系分析，去虚拟机化，符号常量传播，自动装箱，消除逃逸分析，锁消除，锁膨胀，消除反射
- 内存及代码位置交换：表达式提升，表达式下沉，冗余存储消除，相邻存储合并，交汇点分离
- 循环变换：循环展开，循环剥离，安全点消除，迭代分离，范围检查消除
- 全局代码调整：内联，全局代码外提，基于热度的代码布局，Switch调整
- 控制流图变换：本地代码编排，本独代码封包，延迟槽填充，着色图寄存器分配，线性扫描寄存器分配，复写聚合，常量分裂，复写移除，地址模式匹配。指令窥孔优化，基于确定有限状态机的代码生成

方法内联一般会在各种编译器优化序列的最靠前位置，其可以除去方法调用成功，同时方法内联膨账之后代码以范围更广为后续其他优化建议良好的基础。之后则继续如冗余代码消除、复写传播、无用代码消除等。
> 增加内联函数的可能性可通过增加final修饰符、写方法时尽量不要写得太大
##### 2. 公共子表达式消除
公共表达式消除是普遍应用于各种编译器的经典优化技术，其含义是：如果一个表达式E已计算过，且从先前计算到现在E中所有变量的值都没有发生变化，那么E的这次出现就成为了公共表达式；对于这种表达式，直接用先前计算过的表达式结果代替E即可。若公共子表达式消除优化仅限于程序的基本块内，称为局部公共子表达式消除；若这种优化涵盖多个基本块，则称为全局公共子表达式消除。
##### 3. 数组边界检查消除
1. 思路：尽可能把运行期检查提前到编译期完成。
> 数组边界检查消除是即时编译器中的一项语言相关的经典优化技术。对于数组foo[],Java语言访问数组foo[i]系统会自动进行上下边界检查，若下标越界则抛出ArrayIndexOutOfBoundsException 异常。对开发而言是好事，无需专门编写防御代码也可避免大部分的溢出攻击。但对于虚拟机执行子系统来说，每次数组访问均会带一次隐含的条件判断，在拥有大量数据访问时则会有性能负担。
> 安全起见，数据边界检查是必须做，但非必须。如编译期根据数据流分析数组长度及下标判断是否越界，若明确没有越界则无需判断。对于循环内的数据访问，使用循环变量访问数组，通过数据流分析可判断循环变量的取值范围永远在[0,foo.length]之内，则可把整体循环中的数组上下边界检查消除，可节省很多次条件判断操作。
2. 隐式异常处理优化
> 隐式异常处理：Java中空指针检查和算术运算除数为0的检查。公认健壮的程序会先进行参数if非空判断，若不为空则正常运行；若为空则报出异常。而隐式优化则是用try catch()替换if非空判断，若不为这则正常运行，若为空则运行时报出异常。而从实际出来，若参数极少出现为空则try catch（）方式可极大减少非空判断是值得的；但若参数经常出现为空则更适合走if判断，因为异常抛出涉及到用户态与内核态切换，其时间无大于一次非空判断。而HotSpot则对此做了优化，即根据运行期收集到的Profile信息自动选择最优方案。
##### 4. 方法内联
方法内联是编译器最重要的优化手段之一，可消除方法调用成本，更重要的意义是为其他优化手段建议良好的基础。
```
	public static void foo(Object obj){
		if(obj != null){
			System.out.println("do something");
		}
	}
	
	public static void testInline(){
		Object obj = null ;
		foo(obj);
	}
```
整体来看，foo()方法执行没有任何意义；但若分开两个方法不做内联单独看，两个方法均有意义。而内联之后则是"Dead Code"。
- 实际，大多数方法是无法内联，其原因是第8章讲解Java方法解析和分派调用，只有使用invodespecial指令调用的私有方法、实例构造器、父类方法以及使用invokestatic指令进行调用的静态方法才可在编译期解析（即虚方法）；其他方法因涉及到多态选择且可能存在多于一个版本的方法接收者（final方法修饰符也特殊，虽使用invokevirtual指令但也是非虚方法）。
- 而实际大多数方法为虚方法，编译期时无法确定应该使用哪个方法版本即无法做内联。即内联与虎方法之间产生“矛盾”，该如何处理？是不是为了提高执行性能，到处使用final关键字修饰方法吗？
> 为解决该问题，引入了“类型继承关系分析“（Class Hierrarchy Analysis,CHA）技术，是可基于整个应用程序的类型分析技术，可用于确定已加载的类中某个接口是否有多于一种实现，某个类是否存在子类、子类是否为抽象类等信息。
编译器内联，如果为非虚方法则直接进行内联，此时的内联是稳定前提保障的。若是虚方法，则会向CHA查询此方法在当前程序是否有多个目标版本可供选择，如果查询只有一版本则也可进行内联，不过此种内联属于激进优化，需要预留一个逃生门，称为守护内联（Guarded Inlining)。若程序后续执行过程中，虚拟机未加载到会令这个方法的接收者的继承者关系发生变化的类，则该内联优化的代码可一直使用；若加载新类导致继承关系变化，则需抛弃已编译的代码，退回到解释状态执行或者重新进行编译。
> 若向CHA查询出来的结果是多个版本的目标方法可供选择，则编译器会尝试最后一次努力，使用内联缓存（Inline Cache）来完成方法内联。即在方法正常入口之前的缓存，其原理是：未发生方法调用之前，内联缓存状态为空，而第一次调发生后，缓存记录下方法接收者的版本信息，并且每次进行方法调用时都比较接收者版本。若以后进来的每次调用方法接收者版本一样，则内联可继续使用；若方法接收者不一致，则说明程序真正使用了虚方法的多态特性，此时会取消内联，查找虚方法表进行方法动态分派。
##### 5. 逃逸分析
- 逃逸分析（Escape Analysis）是目前Java虚拟机比较前沿的优化技术，其并不是直接优化代码的手段，而是为其他优化手段提供依据的分析技术（类似 类型继承关系分析）。
逃逸分析的基本行为是分析对象动态作用域：当一个对象在方法中被定义后，它可能被外部方法所引用，例如作为参数传递到其他方法，称为方法逃逸。甚至还有可能被外部线程访问，譬如赋值给类变量或者可以在其他线程访问的实例变量，称为线程逃逸。
- 如果能证明一个对象不会逃逸到方法或线程之外，即另的方法或线程无法通过任何途径访问到该对象 ，则可能为这个变量进行一些高效的优化：
1. 栈上分配（Stack　Allocation）：Java虚拟机，在Java堆分配创建对象被认为是常识，而Java堆中的对象各线程是共享和可见的，只要持有对象的引用就可访问存储在堆中的对象数据。而虚拟机的垃圾回收虽然可标识并回收不再使用的对象，但筛选、回收整理均比较耗时。若能确定一个对象不会逃逸出方法之外，那为该对象栈上分配内存则更好，对象所占用的空间可随栈帧出栈而销毁。一般应用，不会逃逸的对象占比很大，若能栈上分配，大量对象可随方法调用结束自动销毁，垃圾收集的压力则会小很多。
2. 同步消除(Synchronization Elimination):线程同步本身是相对耗时的过程，若逃逸分析能确定一个变量不会逃逸出线程，无法被其他线程访问，那变量的读写肯定不会有竞争，对这个变量实施的同步措施也可消除掉。
3. 标量替换(Scalar Replacement)：标量(Scalar）是指一个数据已经无法再分解成更小的数据时，Java虚拟机中的原始数据类型（int、long等数值类型及reference类型等）都不能再进一步分解，它们就可称为标量。相对，若一个数据可继续分析，则它称为聚合量（Aggregate），Java中的对象是最典型的聚合量。如果把一个Java对象拆散，根据程序访问情况 ，将其使用到的成员变量恢复原始类型来访问就叫做标量替换。若逃逸分析能个对象不会被外部访问且可以拆散，那程序真正执行时可能不创建该对象，而是改为直接创建它的若干被这个方法使用到的成员变量来替换。而对象拆散后，除了可让对象的成员变量在栈上分配（栈上存储的数据，有很大的概率会被虚拟机分配至物理机器的调整寄存器中存储）和读写，还可为后续进一步优化手段创建条件。
- 逃逸分析论文较早发表，但直到Sun JDK 1.6 才实现了逃逸分析，且目前这项优化尚未足够成熟。主要原因为逃逸分析性能收益必须高于其消耗，否则适得其反，且因为其分析准确不足可能效果不稳定，
> 通过 java -XX:+PintFlagsFinal -XX:+PrintFlagsInitial 查看sun jdk8 默认开启逃逸分析（-XX:+DoEscapeAnalysis），同时标量替换也默认开启（-XX:+EliminateAllocations）、，同步消除默认开启(-XX：+EliminateLocks）

#### 4. Java与C/C++的编译器比较
Java与C/C++的编译器比较实际代表最经典的即时编译器与静态编译器的对比：
1. 即时编译器运行占用程序运行时间，时间压力大且优化手段严重受制于编译成本。
2. Java语言是动态的类型安全语言，意味着虚拟机需确保程序不会违反语言语义或访问非结构化内存，即必须频繁进行动态检查、空指针、数组上下界、类型转换等。
3. Java虚方法频率高，运行时过多的多态选择。
4. Java语言可动态扩展，运行时加载的类可能改变程序类型的继承关系，使得编译器需时刻注意类型变化而在运行时撤销或重新优化。
5. Java语言对象大多数在堆上分配，仅方法中的局部变量才能在栈上分配（逃逸分析可部分在栈上分配）。C/C++可多种内存分配，且用户程序回收内存，不存在无用对象筛选过程。
但Java则在类型继承关系分析、运行期性能监控为基础的优化则是静态编译器无法做的，如调用频率预测、分支频率预测等。


参考资料：[JIT晚期(运行期)](https://www.cnblogs.com/wade-luffy/p/6050483.html) 
