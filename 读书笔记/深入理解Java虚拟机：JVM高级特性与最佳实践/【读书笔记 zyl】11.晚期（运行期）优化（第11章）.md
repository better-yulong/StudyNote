
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







参考资料：[JIT晚期(运行期)](https://www.cnblogs.com/wade-luffy/p/6050483.html) 