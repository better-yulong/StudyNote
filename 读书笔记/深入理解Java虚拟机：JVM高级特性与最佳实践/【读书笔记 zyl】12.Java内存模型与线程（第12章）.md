并发处理的广泛应用是使得Amdahl定律代替摩尔定律成为计算机性能发展源动力的要本原因。
#### 一、概述
衡量一个服务性能的高低好坏，每秒事务数（Transactions Per Second,TPS）是最重要的指标之一，它代表着一秒内服务端平均响应的请求总数。

#### 二、硬件的效率与一致性
1. CPU优化技术：CPU高速缓存、指令乱序执行
处理器性能 = 主频 X IPC（每周期执行的指令数），提升ICP的做法有两种：提高单核并行的度或者多核
- 高速缓存（Cache）是处理器与内存之间的缓冲。每个处理器都有自己的高速缓存，而它们又共享同一主内存，即引入新的问题：缓存一致性（Cache Coherence）。当多个处理哭器的运算任务都涉及同一主内存时，可能导致缓存数据不一致，为解决该问题，需各处理器访问高速缓存时遵循一些协议，读写必须根据协议来操作。而"内存模型"则可理解为特定的操作协议下，对特定内存或高速缓存进行读写访问的过程抽象。
> ![处理器、高速缓存、主存之间的交互关系](https://github.com/better-yulong/StudyNote-Resource/blob/master/StudyNote-Resource/12-001.PNG)
- 除增加高速缓存外，处理器内部也可能会对输入代码进行乱序执行（Out-Of-Order Execution）优化，仅保证同一处理结果与顺序执行一致，但若一个计算任务依赖中外一个计算任务的中间结果，则顺序性并不能靠代码先后顺序来保障（多线程明显，即不能单纯的代码顺序等价于执行顺序）。
与此类似，Java虚拟机的即时编译器也有类似的指令重排序（Instruction Reorder）优化。

#### 三、Java内存模型
Java虚拟机规范定义一种Java内存模型（Java Memory Model，JMM）来屏蔽各种硬件和操作系统的内存访问差异 ，以实现Java程序在各平台能达到一致的内存访问效果。而此之前，主流语言（如C/C++）等则直接使用物理硬件和操作系统的内存模型，因为会由于不同内存模型平台的差异，导致程序在不同的平台执行异常，使得需针对不同平台编写程序。
而Java内存模型必须足够严谨以避免Java的内存并发访问操作产生歧义；但也需足够宽松，使得虚拟机的实现有足够的自由空间去利用硬件各种特性（寄存器、高速缓存和指令集中某些特有指令）来获取更好的执行速度。直到JDK1.5，Java内存模型对相对完善和成熟。
##### 1. 主内存与工作内存
- Java内存模型的主要目标是定义程序中各个变量的访问规则，即在虚拟机将变量存储到内存和从内存读取变量这样的底层细节。此处的变量（Variables）与Java编程中所说的变量有所区别，它包括实例字段、静态字段和构成数组对象的元素，但不包括局部变量与方法参数（因局部变量与方法参数为线程私有，不会被共享也不存在竞争）。这了获得较好的执行效能，Java内存模型并没有限制执行引擎使用处理器的特定寄存器或缓存来和主内交互，也没有限制即时编译器进行代码执行顺序高速这类优化措施。
- Java内存模型规定所有的变量都存储在主内存（Main Memory，其是虚拟机内存的一部分但类似硬件的主内存）。每条线程还有各自的工作内存（Working Memory,其是虚拟机内存的一部分但类似硬件的处理器高速缓存），线程的工作内存保存了该线程使用到的变量的主内存副本拷贝（注意：局部就是表是一个reference类型时，它引用的对象在Java堆中是可被各线程共享的，但是reference本身在Java的局部变量表，是线程私有的）。线程对变量的所有操作（读取、赋值等）都必须在工作内存进行，而不能直接读写主内存中的变量。不同线程无法直接访问对方工作内存内的变量，线程间变量值的传递均需通过主内存完成，线程、主内存、工作内存三者关系如图：
![JMM内存模型交互](https://github.com/better-yulong/StudyNote-Resource/blob/master/StudyNote-Resource/12-002.PNG)
> 此处的主内存、工作内存与第2章Java内存区域的Java堆、栈、方法区等并非同一层次的内存划分，基本没有关系。对于JMM和JVM本身的内存模型，这两者本身没有关系。如果一定要勉强对应，则从变量，主内存，工作内存的定义来看，主内存主要是对应于Java堆中的对象实例数据部分，而工作内存则对应于虚拟机栈中的部分区域。从更低层次上说，主内存就是物理内存，而为了获取更好的执行速度，虚拟机（甚至是硬件系统本身的优化措施）可能会让工作内存优先存储于寄存器和高速缓存中，因为运行时主要访问–读写的是工作内存。
- JMM的目的是为了解决Java多线程对共享数据的读写一致性问题，通过Happens-Before语义定义了Java程序对数据的访问规则，修正之前由于读写冲突导致的Cache数据不一致的问题。具体到Hotspot VM的实现，主要是由OrderAccess类定义的一些列的读写屏障来实现JMM的语义；即是线程与内存之间的逻辑内存模型。
- JVM内存模型则是指JVM的内存分区，是JVM运行时数据区的内存划分的逻辑内存模型。
- 归根结底，JMM内存模型及JVM运行时数据区均是Java虚拟机规范定义的规则，仅是逻辑概念。
1. 拷贝副本：对象引用、对象中某个线程访问一的字段有可能拷贝，但虚拟机不会实现整个对象拷贝。
2. 根据Java虚拟机规范，volatitle变更依然有工作内存的拷贝，但由于它特殊的操作顺序性规定，只是其看起来如同直接在主内存读写访问。

##### 2. 内存间交互工作
主内存与工作内存之间的交互协议，即一个变量如何从主内存拷贝到工作内存、如何从工作内存同步回主内存的实现细节，Java内存模型定义了以下8种操作，虚拟机需保证下面提及的第一项都必须是原子、不可再分（double、long类型变量部分平台可能有例外）：
- lock（锁定）:作用于主内存的变量，把一个变量标识为一条线程独占状态。
- unlock（解锁）:作用于主内存，把一个处于锁定状态的变量释放出来，释放后的变量才可被其他线程锁定。
- read（读取）:作用于主内存变量，把一个变量的值从主内存传输到线程的工作内存，以便随后的load动作使用。
- load（载入）：作用于工作内存的变量，把read操作从主内存拷贝的变量值放入到工作内存的变量副本。
- use（使用）:使用于工作内存的变量，把工作内存中的一个变量的值传递给执行引擎，每当虚拟机遇到一个需要使用到变量的值的字节码指令时就会执行这个操作。
- assign（赋值）:使用于工作内存的变量，把一个从执行引擎接收到的值赋给工作内存的变量，每当虚拟机遇到一个给变量赋值的字节码指令时执行该操作。
- store（存储）：使用于工作内存的变量，把工作内存中的一个变量的值传送到主内存，以便随后的write操作使用。
- write（写入）：使用于主内存的变量，把store操作从工作内存中得到的变量的值放入主内存的变量。
> 如把一个变量从主内存复制到工作内存，则顺序执行read、load操作，如把一个变量从工作内存同步到主内存，则顺序执行store、write操作。Java内存模型要求上述两个操作必须顺序执行，但没有保证连续执行。即read与load之间、store与write之间可插入其他指令，如对主内存的变量a、b进行访问，一种可能出现的顺序是read a、read b、load b、load a。除此之外，Java内存模型还规定执行上述8种基本操作时必须满足如下规则：
- 不允许read和load、store和write操作之一单独出现，即不允许一个变量从主内存读取但工作内存不接收，或者从工作内存发起回写但主内存不接受的情况出现。
- 不允许一个线程丢弃它最近的assign操作，即变量在工作内存改变后必须把该变化同步回主内存。
- 不允许一个线程无原因地（没有发生过任何assign操作）把数据从线程的工作内存同步回主内存。
- 一个新的变量只能在主内存中诞生，不允许在工作内存中直接使用一个未被初始化(load或assign）的变量，换句话说，就是对一个变量的use、store操作之前，必须先执行过assign和load操作。
- 一个变量在同一时刻只允许一条线程对其进行lock操作，但lock操作可以被同一条线程重复执行多次，多次执行lock后，只有执行相同次数的unlock操作变量才会被解锁。
- 如果对一个变量执行lock操作，将会清空工作内存中此变量的值，在执行引擎使用这个变量前，需要重新load或assign操作初始化变量的值。
- 如果一个变量事先没有被lock操作锁定，那就不允许对它执行unlock操作，也不允许去unlock一个被其他线程锁定的变量。
- 对一个变量执行unlock操作之前，必须先把此变量同步回主内存（执行store、write操作）
> 这8种内存访问操作及规则限定，加上之后 volatile 特殊规定，即完全确定了Java程序中哪些内存访问操作在并发下安全。但烦琐且实现麻烦，后续介绍一种等效判断原则 ---- 先行发生原则，可确定一个访问在并发环境是否安全。
- 主内存、工作内存以及执行引擎，三者关系可以理解为:主内存 <---> 工作内存  <---> 执行引擎
- 
##### 3. volatile型变量的特殊规则
volatile是Java虚拟机提供的最轻量级的同步机制，但其并不容易完全被正确、完整的理解，所以需处理多线程同步时一律使用synchronized实现。
Java内存模型对volatile专门定义了特殊的访问规则，先简单介绍，当一个变量被定义为volatile后，它将具备两种特性:
1. 第一是保证此变量对所有线程的可见性，这里的"可见性"指当一条线程修改该变量的值，新值对于其他线程来说是可以立即得知；而普通变量则做不到这点，普通变量在线程间传递均需要通过主内存来完成。
> 即volatile变量在各个线程的工作线程可存在不一致，但是其他线程在使用前需根据主内存刷新工作内存，对于执行引擎而言可认为不存在一致情况。但是Java的运行并非原子操作，导致valatile变量的运行并发并不一定安全。
```
public class VolatileCalTest {
	public static volatile int a = 0;
	
	public final static int THREAD_COUNTS = 20 ;
	
	public static void increase(){
		a++;
	}
	
	public static void main(String[] args) {
		Thread[] threads = new Thread[THREAD_COUNTS];
		for(int i=0 ; i<threads.length; i++){
			threads[i] = new Thread(new Runnable(){

				@Override
				public void run() {
					for (int i=0 ;i<10000; i++){
						increase();
					}
				}
				
			});
			threads[i].start();
		}
		
		while(Thread.activeCount()>1){
			Thread.yield();
		}
		
		System.out.println(a);

	}

}

```
20个线程，每个线程运行10000次，如a++线程安全则结果应该为200000，但实际每次运行结果不一样且均远远小于200000。原因即在a++。便于分析理解，将该类编译为class文件，之后使用javap命令：javap -p -l VolatileCalTest.class
 ``` public static void increase();
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=0, args_size=0
         0: getstatic     #2                  // Field a:I
         3: iconst_1
         4: iadd
         5: putstatic     #2                  // Field a:I
         8: return
      LineNumberTable:
        line 9: 0
        line 10: 8

```
- 基于 主内存 <---> 工作内存  <---> 执行引擎 三者关系，volatile类型变量 a++ 运算从字节码层面可以可分析得出：getstatic 指令把a的值把a的值取到操作栈顶，volatile保证其正确（基于主内存刷新工作内存）；但是iconst_1、iadd指令执行时，其 他线程可能已经把a的值加大，所以之后putstatic指令则可能把基于操栈项旧值计算的较小的 a 值同步回主内存。（客观说，其实并来严谨，即使编译出来只有一条字节码也并不意味这条指令是原子操作，一条指令解释执行时解释器可能需要运行多条代码才能实现其语义，可使用-XX:+PrintAssembly参数输出反汇编分析更严格；但字节码已经说说明问题）。即只能保证volatile类型变量在 主内存 <---> 工作内存  之间线程安全。
- 因volatile变量只能保证可见性，只有符合以下两个条件的场景才可使用，其他场景仍然需要加锁（synchronized或java.util.concurrent 中的原子类）来保证原子性：
  1. 运算结果不依赖变量的当前值，或者能够确保只有单一的线程修改变量的值。
  2. 变量不需要与其他的状态变量共同参与不变约束。

- volatile变量第二个语义即禁止指令重排序优化，即Java内存模型描述“线程内表现为串行的语义。实际valoatile变量在运行时，对其修改时底层会加锁然后同步工作内存至主内存，而锁操作相关于内存屏障，即使多CPU也必须保证执行顺序。CPU的指令重排序是允许将多条指令不按程序规定的顺序分开发送给各相应电路单元处理。但本CPU内指令重排序运行结果不变可认为依然有序。synchronized因其消除与优化，与volatile性能难于比较。初步可认为，valoatile变量的读与普通变量无区别，但涉及写操作则因需要在本地代码插入许多内存屏障来保证处理器不发生乱序执行相对会慢一些；总的来说，大多数场景volatile总开销要比synchronized低；而volatile与synchronized 的选择依赖则是确认volatile语义是否满足场景需求。
- 个人理解：volatile变量涉及运运算、多线程执行时线程依赖volatile 修改控制线程运行逻辑时需谨慎使用。

##### 4. volatile型变量的特殊规则
- Java内存模型要求lock、unlock、read、load、assign、use、store、write这8个操作具有原子性，但对于64位数据类型(long和double)则规定相对宽松：允许虚拟机将没有被volatile修改的64位数据读写操作划分为2次操作进行，即允许虚拟机可不保证64位数据类型的load、store、read和write 4个操作的原子性，这就是 long和double的非原子协定。
- 若多个线程共享一个未声明为volatile的long或double类型的变量，并且同时对其进行读取和修改操作，那么某线程可能 会读取到一个即非原值也非新修改值"半个变量"的数值。虽然Java内存模型允许虚拟机不把long、double变量的读写实现成原子操作，但也同样允许并强烈建议把这些操作实现为原子性操作。目前实际开发中，各平台下的商用虚拟机几乎都将64位数据的读写操作作为原子操作对待，所以一般不需要把long和double变量场景声明为volatile（不会出现"半个变量"数据）。所以可认为所有对基本数据类型及reference的读写均为原子操作，但其运算则并不一定线程安全，除非使用其他方式保证。

##### 5. 原子性、可见性与有序性
Java内存模型主要围绕并发过程中原子性、可见性和有序性 3个特征建立：
- 原子性（Atomicity）：由Java内存模型直接保证原子性变量操作包括read、load、use、assign、store和write，可认为所所有基本数据类型的访问读写均具备原子性（long、double虽然非原子协定例外，但几乎不会发生例外）。如若应用场景需更大范围的保证原子性，Java内存模型则提供lock和unlock操作满足需求。但虚拟机并未直接把lock和unlock操作直接开放给用户，但提供了更高层次的字节码指令 monitorenter和monitorexit 隐式使用这两个操作，这两个字节码指令反映到Java代码就是同步块---synchronized。因此,synchronized块之间的操作也具备原子性。
- 可见性(Visibility）:可见性是指一个线程修改共享变量的值，其他线程能够立即得知这个修改。Java内存模型是通过变量修改后将新值同步回主内存，而变量读取前从主内存刷新新值至工作内存的方式不实现可见性。普通变量与volatile变量都是如此，但区别在于volatile的特殊规则保证修改为新值后能立即同步至主内存，以及每次使用前立即从主内存刷新。因此，volatile保证多线程操作时变量的可见性，而普通变量则不能保证这一点。除valitile外，Java还有两个关键字可实现可见性，即synchronized和final。同步块的可见性是"对一个变量执行unlock前必须先把变量从工作内存同步回主内存（执行store和write操作）"; 而final关键字的可见性则是指：被final修饰的字段在构造器中一旦初始化完成，并且构造器（类构造或对象构造函数、static代码段）没有把"this”的引用传递出去（this引用逃逸是一件很危险的事情，其他线程有可能通过这个引用访问到"初始化一半"的对象），那其他线程中就能看见final字段的值。
> this引用逃逸：参考  https://www.cnblogs.com/jian0110/p/9369096.html。那什么情况下会This逃逸？
1. 在构造器中很明显地抛出this引用提供其他线程使用（如上述的明显将this抛出）。
2. 在构造器中内部类使用外部类情况：内部类访问外部类是没有任何条件的，也不要任何代价，也就造成了当外部类还未初始化完成的时候，内部类就尝试获取为初始化完成的变量
   1. 在构造器中启动线程：启动的线程任务是内部类，在内部类中xxx.this访问了外部类实例，就会发生访问到还未初始化完成的变量
   2. 在构造器中注册事件，这是因为在构造器中监听事件是有回调函数（可能访问了操作了实例变量），而事件监听一般都是异步的。在还未初始化完成之前就可能发生回调访问了未初始化的变量。
- 有序性(Ordering):Java程序中天然的有序性可总结为：如果在本线程内观察所有操作都是有序；如果在一个线程中观察另外一个线程所有操作均是无序。前半句指"线程内表现为串行的语义"，后半句是指"指令重排序"现象和"工作内存与主内存同步延迟"现象。
Java提供了synchronized和volatile关键字来保证线程之间操作的有序笥，volatile关键字本身就包含了禁止指令重排序的语义，而synchronized则是由"一个变量在同一时刻只允许一条线程对其进行lock操作"规则获得，这条规则决定了同一时刻持有同一锁的两个同步块只能串行进入。
> 然而，正是因为synchronized在需要这3种特性时需可支持，故被当成一种万能的方案，使得大部分并发控制操作都能使用synchronized来完成才间接造成乱用，因其万能通常也会伴随越大的性能影响。

##### 6. 先行发生原则
若Java内存模型所有的有序性均依靠volatile和synchronized来完成，有些操作则会变更烦琐；但实际编译Java并发编码时并未感知，是因为Java语言的"先行发生"（happen-before）原则。这个原则非常重要，其是判断数据是否存在竞争、线程是否安全的主要依据，依靠该原则，可通过几条规则解决并发环境下两个操作之间是否可能存在冲突的所有问题。
先行发生是指Java内存模型中定义的两项操作之间的偏序关系。若两个操作的关系可基于下列规则推导白蘑菇则其执行有序，若无法推迟则顺序性无保障，虚拟机可对它们随意重排序：
1. 程序次序规则（Program Order Rule）：在一个线程内，按照控制流顺序（代码先后顺序，结合分支、循环等）前面的操作执行先于后面的操作。
2. 管程锁定规则（Monitor Lock Rule）：一个unlock操作先于发生于后面对同一个锁的lock操作。同一个锁，"后面是批时间上的先后顺序"。
3. volatile变量规则(Volatile Variable Rule):对一个volatile变量的写操作先行发生于后面这个变量的读操作，同样"后面是批时间上的先后顺序"。
4. 线程启动规则(Thread Start Rule):Thread对象的start()方法先行发生于此线程的每一个动作。
5. 线程终止规则(Thread Termination Rule)：线程中所有的操作都先行发生于此线程的终止检测，我们可通过Thread.join()方法结束、Thread.isAlive()的返回值等手段检测到线程已终止执行。
6. 线程中断规则(Thread Interruption Rule):对线程的interrupt()方法的调用先行发生于被中断线程的代码检测到中断事件的发生，可通过Thread.interrupted()方法检测到是否有中断发生。
7. 对象终结规则(Finalizer Rule):一个对象的初始化完成（构造函数执行结束）先行于发生于它的finalizer()方法的开始。
8. 传递性(Transitivity):如果操作A先行发生于操作B，操作B先行发生于操作C，那么可得出操作A先行发生于操作C的结论。

- Java无需任何同步手段保障就能保证的先行发生规则仅限上面8条。判断操作间是否具备顺序性（对于读写共享变量操作来说，线程是否安全），主要关注于"时间上的先后顺序"和"先行发生"。
1. "时间上的先发生"并不代表这个操作会"先行发生"，因代码层面的操作可能被编译成多条Java指令，而多条指令过程中若非原子操作则其他时间上后发生的线程有可能在任何时间点执行，即结果不可预测 ，线程不安全。
2. "操作的先行发生"也并非"时间上的先发生"，如涉及到"指令重排序"，即使同一线程内有明确先后顺序的代码，有可能基于指令后果排序顺序不可预测。
- 归根结底即为两点：
1. Java代码编译后即使简单代码也可能被拆分成多条指令执行，而多条指令的原子性则需根据变量及操作判断，非原子性操作则可能即时方法在执行是先执行，但仍有可能被时间上后执行的指令插入执行，导致执行顺序不可预测；
2. 因指令重排序，导致代码层面的并非其真实执行的先后顺序，即执行顺序不可预测。
- 即时间先后顺序与先行发生原则之间没有必然的关系，衡量并发安全问题不能受时间顺序的干扰，一切必须以先行发生原则为准。

#### 四、Java与线程
并发不一定依赖多线程（如PHP多进程并发），但Java谈论并发基于与线程脱不开关系。
##### 1.线程的实现
- 线程是比进程更轻量级的调度执行单位，线程的引入可把进程的资源分配和执行调度分开，各个线程即可共享进程资源（内存地址、文件I/O等），又可独立调用（线程是CPU调度的基本单元）。
- 主流的操作系统都提供了线程实现，Java语言则提供了在不同硬件和操作系统平台下对线程操作的统一处理，每个已经执行start()且还结束的java.lang.Thread类的实例代表一个线程。Thread类与大部分的Java API有显著的差别，其所有关键方法都是声明为Native的(currentThread、yield、sleep等)。在Java API中，一个Native方法往往意味着这个方法没有使用或无法使用平台无关的手段来实现（也可能是为了执行效率而使用Native方法，不过通常最高效率的手段也就是平台相关的手段。（此处不仅仅是针对Java，是线程实现的通用理解）
- 线程的实现主要有3种方式 ：使用内核线程实现、使用用户线程实现、使用用户线程加轻量级进程混合实现。
1. 使用内核线程实现
内核线程（Kernel-Level Thread，KLT）是直接由操作系统内核（Kernel，下称内核）支持的线程，这种线程由内核来完成线程切换，内核通过操作调度器（Scheduler）对线程进行调度，并负责将线程的任务映射到各处理器上。每个内核线程可视为内核的一个分身，这样操作系统就有能力同时处理多件事情，支持多线程的内核就叫做多线程内核（Multi-Treads Kernel）。
程序不般不会直接使用内核线程，而是去使用内核线程的高级接口---轻量级里程（Light Weigth Process，LWP），轻量级进程即是我们通常意义所讲的线程。因每个轻量级进程都有一个内核线程支持，因此只有先支持内核线程，才能有轻量级进程。这种轻量级进程与内核线程之间1：1的关系称为一对一的线程模型。
 ![轻量级进程与内核线程关系](https://github.com/better-yulong/StudyNote-Resource/blob/master/StudyNote-Resource/12-003.PNG)
  - 由于内核线程的支持，每个轻量级进程都会成为一个独立的调度单元，即使有一个轻量级进程在系统调用阻塞，也不会影响整个进程继续工作，但轻量级进程也有其局限性：因是基于内核线程实现，所以各种操作，如创建栈、析构及同步都需要进行系统调用；而每次系统调用的代价较高，需要在用户态（User Mode)和内核态(Kernel Mode）来回切换。其次，每个轻量级进程都个内核线程的支持，因此轻量级进程要消耗一定的内核资源（如内核线程的栈空间），因此每个系统支持的轻量级进程的数量是有限的。
2. 使用用户线程实现
广义来讲，一个线程只要不是内核线程即可认为是用户线程(User Thread,UT)，因此基于这个定义轻量级进程也属于用户线程。但实际轻量级进程的实现始终是建议在内核之上，其依赖系统调用，效率会受到限制。
而狭义上用户线程指的是完全建议在用户空间的线程库上，系统内核不能感知线程存在的实现。用户线程的建立、同步、销毁和调度完全在用户态完成，无需内核的帮助。若程序实现得当，用户线程无需切换到内核态，因此操作是快速且低消耗，也可支持规模更大的的线程数量，部分高性能数据库的多线程就是由用户线程实现。这种进程与用户线程之间1：N的关系称为一对多的线程模型。
 ![进程与用户线程1：N关系](https://github.com/better-yulong/StudyNote-Resource/blob/master/StudyNote-Resource/12-004.PNG)
- 优势在于不需要系统内核支援，劣势也在于没有系统内核的支援，所有的线程操作均需要用户程序自己处理（复杂的部分实现有线程库已封装支持）。线程的创建、切换和调度均需考虑，且操作系统只把处理器资源分配到进程，诸如"阻塞如何处理"、"多处理器系统如何将线程映射到其他处理器"这类问题解决则会异常困难，甚至不可能完成。因而使用多用户线程的程序一般相对比较复杂，现使用用户线程的程序很少。
3. 使用用户线程加轻量级进程混合实现
线程除了依赖内核线程实现和完全由用户程序实现外，还有一种将内核线程与用户线程一起使用的实现方式；这种混合实现，即存在用户线程，也存在轻量级进程。用户线程仍完全建立在用户空间，因此用户线程的创建、切换、析构等操作依然廉价,且可支持大规模的用户线程并发。而操作系统提供支持的轻量级进程则作为用户线程和内核线程之间的桥梁，这样则可使用内核提供的线程调度功能和处理器映射，并且用户线程的系统调用可通过轻量级进程来完成，可大大降低整个进程被阻塞的风险。在混合模式下，用户线程与轻量级进程的数量比不定，即为N:M的关系，即为多对多的线程模型。
![用户线程与轻量级进程N:M关系](https://github.com/better-yulong/StudyNote-Resource/blob/master/StudyNote-Resource/12-005.PNG)
4. Java线程的实现
JDK1.2之前，Java线程是基于"绿色线程"的用户线程实现；JDK1.2开始，线程模型替换为基于操作系统原生线程模型实现。故当前JDK版本中，Java虚拟机的线程映射到操作系统线程，而这点依赖平台实现。而线程模型只对线程的并发规模和操作成本有影响，但对Java程序的编码和运行过过程则是无感知。
- SunJDK,其Windows版本与Linux版均使用1对1线程模型，一条Java线程映射到一条轻量级进程，因Windows和Linux系统提供 线程即是一对一（部分内核有其他实现，但主流是一对一线程模型）。
- 而Solaris平台，因操作系统可同时支持一对一及多对多线程模型，故Solaris版本JDK有平台专有虚拟机参数来明确指定虚拟机使用哪种线程模型。
##### 2.Java线程调度
线程调度是指系统为线程分配处理器使用权的过程，主要调度方式有两种：协同式线程调度 和 抢占式线程调度。
- 协同式调度的多线程系统，线程的执行时间则线程本身来控制，线程工作执行完成需主动通知系统切换到另外一个线程。协同式多线程最大好处是实现简单，而且线程工作完成才进行线程切换，切换操作线程自己可知，无线程同步问题（串行执行，Lua语言的"协同例程"即这类实现）。坏处也相对明显：线程执行时间不可控，若程序线程编写问题，未正常通知系统切换线程，则程序会阻塞。若一个进程坚持不出让CPU执行时间可可能导致整个系统崩溃。
- 抢占式调度的多线程系统，那么每个线程由系统来分配执行时间，线程的切换不由线程本身决定 （Java中，Thread.yield()可让出执行时间，但要获取只能参与排除等待系统调度，没其他办法）。此咱实现线程调度的方式，线程执行时间系统可控，不会出现因一个线程导致整个进程阻塞，java使用的线程调度方式就是抢占式调度。而且即使进程出现阻塞，也可强制结束进程而不至于导致系统崩溃。
- 虽然Java线程调度由系统自动完成，但仍可通过设置线程优化级来“建议”系统给某些线程多分配一点执行时间，另外的线程则少分配一点时间。Java语言共设置10个级别的优先级（Thread.MIN_PRIORITY至Thread.MAX_PRIORITY)。当两个线程同时处于ready状态时，优先级越高的线路赵容易被系统选择执行。但线程优先级并不太靠谱，因Java线程是通过映射到系统的原生线程不实现，所以线程调度最终还取决于操作系统。虽然大多数操作系统提供线程优先级，但各操作系统实现有差异并不一定能与Java线程优先级一一对应（可能多，可能少），且优先级也有可能被系统自行改变，不能过于依赖。
##### 3.状态转换
Java语言语义了5种线程状态，在任意时间点，一个线程只能有且只有其中一种状态，5种状态分别为：
1. 新建（New）:创建尚未启动的线程处于这种状态。
2. 运行(Runnable):Runnable包括了操作系统线程状态的Running和Ready，也就是处于此状态的线程有可能正在运行，也有可能正在等待CPU为它分配执行时间。
3. 无限期等待(Waiting):处于这种状态的线程不会被分配CPU执行赶时间，需等待被其他线程显示唤醒。以睛Java方法会让线程陷入无限期的等待状态：
  - 没有设置Timeout参数的Object.wait（）方法。
  - 没有设置Timeout参数的Thread.join()方法。（用于线程同步使得串行执行，底层实现也是基于wait方法。如main方法调用t.join()方法，则main线程放弃cpu控制权，并返回t1线程继续执行直到线程t1执行完毕再返回main线程执行）
  - LockSupport.park()方法。（LockSupport其实是一个静态代理类，功能实现是用Unsafe类里面的native方法；起源于JDK1.6）
  > LockSupport是JDK中比较底层的类，用来创建锁和其他同步工具类的基本线程阻塞原语。Java锁和同步器框架的核心AQS:AbstractQueuedSynchronizer，就是通过调用LockSupport.park()和LockSupport.unpark()实现线程的阻塞和唤醒的。LockSupport很类似于二元信号量(只有1个许可证可供使用)，如果这个许可还没有被占用，当前线程获取许可并继续执行；如果许可已经被占用，当前线程阻塞，等待获取许可。
  > LockSupport中的park() 和 unpark() 的作用分别是阻塞线程和解除阻塞线程，而且park()和unpark()不会遇到“Thread.suspend 和 Thread.resume所可能引发的死锁”问题。因为park() 和 unpark()有许可的存在；调用 park() 的线程和另一个试图将其 unpark() 的线程之间的竞争将保持活性。
LockSupport的park/unpark和Object的wait/notify:面向的对象不同；跟Object的wait/notify不同LockSupport的park/unpark不需要获取对象的监视器；实现的机制不同，因此两者没有交集。虽然两者用法不同，但是有一点， LockSupport 的park和Object的wait一样也能响应中断.
  > 在java6之后在park系列方法新增加了入参Object blocker，用于标识阻塞对象，该对象主要用于问题排查和系统监控。（参考：https://blog.csdn.net/fenglllle/article/details/83049251、https://www.jianshu.com/p/ceb8870ef2c5）
4. 限期等待（Timed Waiting）:处于这种状态的线程也不会被分配CPU执行时间，不过无需等待被其他线程显示唤醒，在一定时间之后它们会由系统自由唤醒。以下方法会让线程进入限期等待状态：
  - Thread.sleep()方法。
  - 设置了Timeout参数的Object.wait()方法。
  - 设置了Timeout参数的Thread.join()方法。
  - LockSupport.parkNanos()方法。
  - LockSupport.parkUntil()方法。
5. 阻塞（Blocked）:线程被阻塞，"阻塞状态"与"等待状态"的区别是："阻塞状态"在等待着获取到一个排他锁，这个事件将在另外一个线程放弃这个锁的时候发生；而"等待状态"则是在等待一段时间，或者唤醒动作的发生。在程序等待进入同步区域，线程将进入阻塞状态。
- 这5种状态去到特定事件会发生状态转换：
  >![Java线程状态转换图](https://github.com/better-yulong/StudyNote-Resource/blob/master/StudyNote-Resource/12-006.PNG)