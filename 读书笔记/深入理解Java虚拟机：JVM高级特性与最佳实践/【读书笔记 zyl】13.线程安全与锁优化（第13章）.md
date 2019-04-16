#### 一、概述
- 软件发展前期，程序编写以算法为核心，程序员会把数据和过程分别作为独立部分考虑，数据代表问题空间的客体，程序代码则用于处理这些数据，这种直接站在计算机的角度去抽象问题和解决问题的思维方式称之为面向过程的编程思想。与之相对，站在现实世界的角度去抽象和解决问题，把数据和行为都看做对象的一部分，则称之为面向对象的编程思想。
- 对于"高效并发"，需先保证并发的正确笥，之后基于此实现高效。

#### 二、线程安全
- "线程安全"定义较多，相对恰当的定义：当多个线程访问一个对象时，如果不用考虑这些线程在运行时环境下的调度和交替执行，也不需要考虑额外的同步，或者调用方进行任何其他的协调操作，调用这个对象的行为都可获得正确的结果，那这个对象线程安全。
- 该定义相对严谨，它要求线程安全的代码都必须具备一个特征：代码本身封装了所有必要的正确性保障手段（如互斥同步等），调用者无须关心多线程的问题，更无须自己采取任何任何措施是来保证多线程的正确调用。但实际很难做到，故大多数场景会把定义弱化：把"调用这个对象的行为"限定为"单次调用"，若满足该条件也可称为线程安全（见后面的相对线程安全）

##### 1.Java语言的线程安全
- 若对象不会被多个线程共享访问，则无论是串行执行还是多线程执行没有区别，均是线程安全的。
- 若多个线程之间存在共享数据访问，那么可把Java语言中各种操作共享的数据分为以下5类：不可变、绝对线程安全、相对线程安全、线程兼容和线程对立。（按照线程"安全程度"划分，而非安全与否的二选一）。
###### 1. 不可变
- Java语言中（特指JDK1.5之后，即Java内存模型被修正之后的Java语言），不可变(Immutable)的对象一定是线程安全的，无论是对象的方法实现还是方法的调用者，都不需要采取任何的线程安全保障措施。如final关键字提到的可见性，只要一个不可变的对象被正确的构造出来（即未发生this引用逃逸的情况），其外部的可见状态永远不会改变，永远也不会出现多个线程之中处于不一致的状态。"不可变"带来的安全性是最简单和最纯粹的。
- Java语言中，若共享数据为基本数据类型，那么定义时使用final关键字修饰它就可保证不可变。若共享数据是一个对象，若可保证的行为不会对其本身的状态产生任何影响，可认为也其也不可变。（如java.lang.String，其不可变组线程安全，为什么呢？是因为String被定义为final且String内部存储字符串的char数组以及和char数组相关的信息都是final的）
- 保证对象行为不影响自身状态的途径有多种，但最简单的则是把对象中带状态的变量均声明为final，这样构造函数执行结束它即不可变（此处的自身状态较难理解，个人理解即是上面说的数据及行为均未改变）。如java.lang.Integer则是通过构造函数，通过将Integer对象的字面量（内部状态变量）value字义为final来保障其不可变。
- Java/API中不可变类型：String、Integer、Enum枚举、java.lang.Number部分子类（Long和Double等数值包装类型、BigInteger和BigDecimal等大数据类型不可变；但是Number的子类AtomicInteger、AtomicLong则非不可变---查看源码，个人理解可发现其对着状态变量并非final类型，如value仅是volatile类型的int)
###### 2. 绝对线程安全
绝对线程安全完全满足"不管运行时环境如何，调用者都不需要任何额外的同步措施"，但需要付出很大且有时不切实际。Java API中标注自己是线程安全的类，大多数都不是绝对的线程安全。
- 如java.util.Vector是一个线程安全的类（doc文档有说明该类是否线程安全），也许大家对此均不会有意义，因为其add()、get()和size()这类方法被synchronized修饰，虽然效率低但安全。
```
package com.test.jvm.concurrent;

import java.util.Vector;

public class VectorUnSafeTest {
	
	private static Vector<Integer> vector = new Vector<Integer>();

	public static void main(String[] args) {
		while(true){
			for(int i=0 ; i<10; i++){
				vector.add(i);
			}
			
			Thread removeThread = new Thread(new Runnable(){

				@Override
				public void run() {
					for(int i=0 ; i<10; i++){
						vector.remove(i);
					}
				}
				
			});
			
			Thread printThread = new Thread(new Runnable(){

				@Override
				public void run() {
					for(int i=0 ; i<10; i++){
						System.out.println(vector.get(i));
					}
				}
			});
			
			removeThread.start();
			printThread.start();
			
			while(Thread.activeCount()>5);
		}

	}

}

```
运行结果：
```
Exception in thread "Thread-49037" java.lang.ArrayIndexOutOfBoundsException: Array index out of range: 9
	at java.util.Vector.get(Vector.java:744)
	at com.test.jvm.concurrent.VectorSafeTest$2.run(VectorSafeTest.java:31)
	at java.lang.Thread.run(Thread.java:722)
```
原因分析源码可发现：Vector中主要有两个重要的成员变量emelentData[]、elementCount（其它如容量控制不暂时忽略），可发现add会基于elementCount在emelentData尾部插入新对象并同步elementCount加1、remove方法调用后会基于System.arraycopy移动emelentData之后同步elementCount减一且置空尾部元素。而get方法则会根据传入index与当前elementCount比较，如上示例因add、remove会导致elementCount动态变化，所以remove、add的for循环而使用add时默认长度去遍历则正好可能遇到i元素不可用抛出数组下标越界异常。那如何修改呢？首先因明确elementCount会随插入或移除动作变化，故for循环需根据elementCount值（私有变量，可通过size()获取）而不是固定值。那这样就可以了吗？显然不可以，虽然get、remove方法虽然以synchronized修改，但仍然可能出现printThread 的for循环刚取到i在get之前，而remove方法正常移除了该元素，所以还需要确保的是for循环开始后emelentData[]、elementCount不能再出现状态改变，故需对remove、get的for及add加入同步。修改如下：
```
//加入同步保证Vector访问的线程安全
public class VectorSafeTest {
	
	private static Vector<Integer> vector = new Vector<Integer>();

	public static void main(String[] args) {
		while(true){
			for(int i=0 ; i<10; i++){
				vector.add(i);
			}
			
			Thread removeThread = new Thread(new Runnable(){

				@Override
				public void run() {
					synchronized(vector){
						for(int i=0 ; i<vector.size(); i++){
							vector.remove(i);
						}
					}
				}
			});
			
			Thread printThread = new Thread(new Runnable(){

				@Override
				public void run() {
					synchronized(vector){
						for(int i=0 ; i<vector.size(); i++){
							System.out.println(vector.get(i));
						}
					}
				}
			});
			
			removeThread.start();
			printThread.start();
			
			while(Thread.activeCount()>5);
		}
	}
}

```
###### 3. 相对线程安全
1. 相对的线程是我们通常意义所讲的线程安全，它需保证对这个对象单独的操作是线程安全，我们在调用的时候不需要做额外的保障措施；但对于一些特定顺序的连续调用，则可能需要调用端使用额外的同步手段保证调用的正确性。
2. Java语言中，大部分的线程安全类都属于相对线程安全，如Vector、HashTable、Collections的synchronizedCollection()方法包装的集合、StringBuffer。
###### 4. 线程兼容
线程兼容是指对象本身并不是线程安全（即使单个操作也非线程安全），但可通过在调用端正确的使用同步手段来保证对象在并发环境中可安全使用。平常所说一个类线程不安全，大多数指此种情况。Java API中大部分类都是属于线程兼容，如Vector和HashTable对应的集合类ArrayList和HashMap、StringBuilder 等。
###### 5. 线程对立
线程对立是指无论调用端是否采取同步措施，但都无法在多线程环境中并发使用。因Java语言天生具备多线程特性，线程对立这种排斥多线程的情况较少且通常有害应该避免。
其中如Thread.suspend()、Thread.resume() 持有同一对象锁而并发执行时则极容易发生死锁，因为suspend方法并不会释放锁,既然suspend和resume使用同一锁那么resume执行前需获得锁才可，即容易造成死锁。除此之外，常见的线程对立操作还有System.setIn()、System.setOut()和System.runFinalizersOnExit()等.
- 示例验证代码1：
```language
package com.test.jvm.concurrent;

/***
 * 验证同步情况下：
 * 1、先suspend后resume线程：可正常唤醒
 * 2、先suspend后不resume线程：线程阻塞
 * 
 * */
public class ThreadLockTest {

	public static void main(String[] args) {
		final SynchronizedObject sobject = new SynchronizedObject();
		try{
			final Thread t1 = new Thread(new Runnable(){

				@Override
				public void run() {
					sobject.printString();
				}
				
			});
			
			t1.setName("t1");
			t1.start();
			Thread.sleep(2000);
			
			Thread t2 = new Thread(new Runnable(){

				@Override
				public void run() {
					System.out.println("t2 start ...");
					t1.resume();  
					//若把 t1.resume();注释掉，则会因t1线程暂停，t2线程调用 sobject.printString（）会无法获取到锁被阻塞，通过jstack 根据线程栈分析可发现
					//"Thread-1" prio=6 tid=0x04e56000 nid=0x174 waiting for monitor entry [0x0523f000]  ; java.lang.Thread.State: BLOCKED (on object monitor)
					System.out.println("t2 start ,resume success ...");
					sobject.printString();
					
				}
				
			});
			t2.setName("t2");
			t2.start();
			
		}catch(Exception e){
			
		}
		
	}

}

class SynchronizedObject{
	synchronized public void printString(){
		System.out.println("printString...");
		if(Thread.currentThread().getName().equals("t1")){
			System.out.println("t1 start...");
			Thread.currentThread().suspend();
			System.out.println("t1 resume...");

		}
	}
}


/***
 * 运行结果（未注释：t1.resume()则方法正常线束；若注释t1.resume()则线程阻塞，通过jstack -l可发现t2为BLOCK状态）：
 * 
printString...
t1 start...
t2 start ...
t2 start ,resume success ...
t1 resume...
printString....

 * 
 * 
 * */

```
```
package com.test.jvm.concurrent;

/***
 * 验证非同步情况先resume线程，后suspend线程，是否如预期发生死锁。
 * 结果：线程假死无法正常结束，jstack日志显示t1、t2均为 WAIT状态，即无限期等待
 * 
 * */
public class ThreadLockTest2 {
	
	public static boolean resumeExeFlag = false ;

	public static void main(String[] args) {
		final SynchronizedObject sobject = new SynchronizedObject();
		
		
		try{
			final Thread t1 = new Thread(new Runnable(){

				@Override
				public void run() {
					sobject.printString();
				}
				
			});
			
			t1.setName("t1");
			t1.start();
			
			Thread t2 = new Thread(new Runnable(){

				@Override
				public void run() {
					System.out.println("t2 start ...");
					t1.resume();  
					resumeExeFlag = true;
					System.out.println("t2 start ,resume status ...");
					sobject.printString();
					
				}
				
			});
			t2.setName("t2");
			t2.start();
			
		}catch(Exception e){
			
		}
		
	}
	
	static class SynchronizedObject{
		 public void printString(){
			System.out.println("printString...");
			if(Thread.currentThread().getName().equals("t1")){
				System.out.println("t1 start...");
				while(!resumeExeFlag);
				Thread.currentThread().suspend();
				System.out.println("t1 resume...");

			}
		}
	}


}


/***
 * 注：static class 静态类(Java) 一般情况下是不可以用static修饰类的。如果一定要用static修饰类的话,通常static修饰的是匿名内部类。
 * 
 * 运行结果：
 * 
printString...
t1 start...
t2 start ...
t2 start ,resume status ...
printString...
之后
 * 
 * 
 * */

```
```
package com.test.jvm.concurrent;

/***
 * 验证同步发生死锁:
 * 运行后发生无限等待，而通过jstack -l 输入线程栈，发现t1、t2为BLOCKED，而日志下方有 Found one Java-level deadlock
 * 
 * */
public class ThreadDeadLockTest {
	
	public static Object obj1 = new Object();
	
	public static Object obj2 = new Object();
	
	public static boolean switchFlag1 = false ;

	public static void main(String[] args) {
		final SynchronizedObject sobject = new SynchronizedObject();
		
		try{
			final Thread t1 = new Thread(new Runnable(){

				@Override
				public void run() {
					synchronized(obj1){
						System.out.println(Thread.currentThread().getName() + "get obj1 lock....");
						while(!switchFlag1);
						synchronized(obj2){
							System.out.println(Thread.currentThread().getName() + "get obj2 lock....");
						}
					}
				}
				
			});
			
			t1.setName("t1");
			t1.start();
			Thread.currentThread().sleep(2000);
			
			Thread t2 = new Thread(new Runnable(){

				@Override
				public void run() {
					synchronized(obj2){
						switchFlag1 = true ;
						System.out.println(Thread.currentThread().getName() + "get obj1 lock....");
						synchronized(obj1){
							System.out.println(Thread.currentThread().getName() + "get obj2 lock....");
						}
					}
				}
				
			});
			t2.setName("t2");
			t2.start();
			
		}catch(Exception e){
			
		}
		
	}
	

}


/***
 * 运行结果：
 * 
t1get obj1 lock....
t2get obj1 lock....
之后无限等待...
 * 
 * 
 * */
```
```language
/***
 * 验证  wait、notify、notifyAll的正确用法是 synchronized(obj){ ...; obj.wait();...}，而并不是调用Thread对象的wait。
 * 
 * */
public class ThreadWaitNotifyTest {
	
	public static void main(String[] args) {
		final Object sobject = new Object();
		try{
			final Thread t1 = new Thread(new Runnable(){

				@Override
				public void run() {
					synchronized(sobject){
						try {
							System.out.println("t1 wait start ....");
							 //Thread.currentThread().wait(); 此处是尝试暂停是t1，而Thread.currentThread()获取的是t1对象的锁标记并未被任何线程持有，
							//会抛出异常IllegalMonitorStateException
							sobject.wait(100000);
							System.out.println("t1 restart ....");
						} catch (InterruptedException e) {
							// TODO Auto-generated catch block
							e.printStackTrace();
						}
					}
				}
				
			});
			
			t1.setName("t1");
			t1.start();
			t1.sleep(2000);
			Thread t2 = new Thread(new Runnable(){

				@Override
				public void run() {
					synchronized(sobject){
						try {
							System.out.println("t1 notify start ....");
							// Thread.currentThread().notify(); 此处是通知唤醒错误，而Thread.currentThread()获取的是t1对象的锁标记并未被任何线程持有，
							//会抛出异常IllegalMonitorStateException
							sobject.notifyAll();
							System.out.println("t1 notify end ....");
						} catch (Exception e) {
							// TODO Auto-generated catch block
							e.printStackTrace();
						}
					}
					
				}
				
			});
			t2.setName("t2");
			t2.start();
			
		}catch(Exception e){
			
		}
		
	}

}



/***
 * 运行结果：
 * 若使用Thread.currentThread().wait(); 此处是尝试暂停是t1，而Thread.currentThread()获取的是t1对象的锁标记并未被任何线程持有，
 * 若使用sobject.wait()，则正常运行完成：
t1 wait start ....
t1 notify start ....
t1 notify end ....
t1 restart ....
 * 
 * */

```

```
在线分析工具：https://www.fastthread.io
```
- 既然说到这儿，那对于Thread相关常见的 wait、notify、notifyAll、sleep、suspend、resume、yield、join这些方法的作用进一步理解并汇总：
1. wait、notify、notifyAll是Object类中的方法，而锁即所谓对象的 monitor，所以基于此也可以理解，wait、notify、notifyAll三个方法必须在同步方法(块)中调用，且wait时会释放锁而notify、 notifyAll之后优先获得锁的线程将重新执行。若在非同步方法(块)中调用wait、notify、notifyAll运行时，因当前线程不是对象锁的持有者，则该方法抛出一个java.lang.IllegalMonitorStateException 异常；因一切皆对象，理论上也意味着任何Java代码随时随地均可调用wait、notify、notifyAll方法。而WAIT方法则会释放锁，线程进入等待（有限期或无限等待）状态。----所以wait、notify、notifyAll的正确用法是 synchronized(obj){ ...; obj.wait();...}，而并不是调用Thread对象的wait；且如wait 10s，但实际可提前notify通知唤醒成功的。
2. sleep(long)、suspend、resume、yield、join、interrupt、interrupted、isInterrupted这些方法均为Thread类的方法，并不要求必须运行于同步方法（代码块）
   - sleep(long)是当将前被服务的线暂停一段时间之后可恢复继续服务，若当前线程持有锁也不会对锁进行任何操作。即重新进入就绪队列等待再次获取CPU资源（经验证，sleep中的线程状态为 TIMED_WAITING）。另Thread.sleep()和Thread.currentThread().sleep()有点儿让人迷惑，但实际效果一样，因
   - suspend、resume方法则对应当前被服务线程的暂停、恢复，若当前线程持有锁也不会对锁进行任何操作；所以会出现上面场景，持有同一锁并发可能造成死锁（验证并未出现 DeadLock，而是WAIT状态或BLOCK）。----（suspend、resume方法已废弃）
   - yield()让当前正在运行的线程回到可运行状态，以允许具有相同优先级的其他线程获得运行的机会。因此，使用yield()的目的是让具有相同优先级的线程之间能够适当的轮换执行。但是，实际中无法保证yield()达到让步的目的，因为让步的线程可能被线程调度程序再次选中。
   - join()定义在Thread.java类，join()的作用:让“主线程”等待“子线程”结束之后才能继续运行。
   - interrupt 中断线程。若线程在调用 Object 类的 wait()、wait(long) 或 wait(long, int) 方法，或者该类的 join()、join(long)、join(long, int)、sleep(long) 或 sleep(long, int) 方法过程中受阻，则其中断状态将被清除，它还将收到一个 InterruptedException。
    1. 若线程t1处于sleep、wait状态，那么在线程t2中运行 t1.interrupt时，t2线程正常向后运行，而t1线程内部会抛出 InterruptedException并直接结束该线程。
    2. 若线程t1处于运行状态，则t2线程内运行 t1.interrupt时对t1无任何影响即t1会正常运行；但若之后出现t1.sleep、Obj.wait时，t1线程则会从内部会抛出 InterruptedException并直接结束该线程。。
    3. 有资料说interrupt可中断join，但经验证线程t一旦进入join后只会先执行完t1线程的内容才会重新执行main.
    4. 分析Thread.interrupt 方法只是修改线程的中断状态，而并不是正直断线线程，需要配合wait、sleep等才有效。
   - interrupted 返回当前线程是否已经中断，线程的中断状态由该方法清除。线程中断被忽略，因为在中断时不处于活动状态的线程将由此返回 false 的方法反映出来。
   - isInterrupted 测试线程是否已经中断。线程的中断状态不受该方法的影响。线程中断被忽略，因为在中断时不处于活动状态的线程将由此返回 false 的方法反映出来。
##### 2.线程安全的实现方法
该部分包含代码编写如何实现线程安全和虚拟机如何实现同步与锁两部分。
###### 1.互斥同步
- 互斥同步（MutualExclusion & Synchronization）是常见的一种并发正确性保障手段。同步是指多个线程并发访问共享数据时，保证共享数据同一时刻只被一个（或者是一些，使用信号量的时候）线程使用。而互斥是实现同步的一种手段，临界区（Critical Section）、互斥量（Mutex）和信号量（Semaphore）都是主要的互斥实现方式。因此，互斥是因同步是果；互斥是方法，同步是目的。
- Java中，最基本的互斥同步手段就是使用synchronized关键字，synchronized关键字经编译后，会在同步块前后分别形成monitorenter和monitorexit两个字节码指令，这两个字节码都需一个reference类型的参数来指明锁定和解锁的对象。若synchronized明确指定对象，则即为该对象的reference；若未明确指定，则根据synchronized所在方法是实例方法还是类方法，则取对应的实例对象或Class对象来作为锁对象。
- 根据虚拟机规范，在执行monitorenter指令时，先需尝试获取对象的锁。如果对象未被锁定或者当前线程已经拥有了锁，则把锁的计数器加1；相应的在执行monitorexit指令时，则把锁的计数器减1，若计数器变为0则释放锁。若获取对象的锁失败，则当前线程要阻塞等待（jstack可看到对应等待锁的线程状态为BLOCKED）,直到对象锁被另一个线程释放并重新尝试获取锁成功。
- 基于虚拟机规范对monitorenter和monitorexit的行为描述，有两点需注意：
 - 1. 是synchronized同步块对于同一线程来说可重入，不会出现把自己锁死的问题；
 - 2. 是同步块在已进入的线程执行完之前会阻塞后面其他线程的进入（其实可通过如wait方式释放锁）。而之前讲过，Java的线程是映射到操作系统的原生线程，若要阻塞或唤醒一个线程，需操作系统帮忙从用户态切换到内核态，而该操作需要消耗很多的处理器时间。对于简单的同步块(如synchronized修饰的setter()或getter()方法），状态转换消耗的时间有可能与用户代码执行的时间还要长。所以，synchronized是Java语言中一个重量级(Heavyweight）操作，需要在确实必要的情况下才使用这种操作。而虚拟机本身也会进行一些优化，譬如在通知操作系统阻塞线程之前加入一段自旋等待过程，避免频繁的切入到核心态。
- 除synchronized，还可使用java.util.concurrent（下方称J.U.C）包中的重入锁(ReentrantLock)来实现同步，ReentrantLock基本用法与synchronized相似都具备线程重入特性，但写法有些区别：ReentrantLock表现为API层面的互斥锁(lock()和unlock()方法配合try/finally语句块实现)，而synchronized表现为原生语法层面的互斥锁（稍后通过字节码层面理解）。除此之外，ReentrantLock增加了一些高级功能，主要为3项：等待可中断、可实现公平锁，以及锁可绑定多个条件。
  1. 等待可中断是指当持有锁的线程长期不释放锅的时候，正在等待的线程可选择放弃等待，改为执行其他逻辑，可中断特性对处理执行时间非常长的同步块有很大帮助。
  2. 公平锁是指多个线程在等待同一锁时，必须按照申请锁的时间顺序来依次获得锁，而非公平锁则不保证这一点，在锁被释放时任何一个等待的锁的线程都有机会获得锁。synchronized的锁是非公平，ReentrantLock的锁默认也是非公平，但可通过带布尔值的构造函数要求使用公平锁。
  3. 绑定多个条件是指一个ReentrantLock对象可同时绑定多个Conditions对象，而在synchronized，锁对象的wai()和notify()或notifyAll()只可实现一个隐含条件，如果要和多个条件关联，则不得不额外添加一个锁，而ReentrantLock则无样做，只需多次调用newCondition()方法即可。
  - 如若基于功能需要，选择ReentrantLock个比较好的选择，那若基于性能考虑呢？关于synchronized和ReentrantLock的性能问题，基于验证多线程环境synchronized的吞吐量下载非常严重，而ReentrantLock则能基本保持一个比较稳定的水平(基于JDK1.5)。与其说ReentrantLock性能好，反而可认为synchronized在非常大的优化余地；从JDK1.6则加入了针对锁较多的优化措施，使得ReentrantLock与synchronized性能基本持平。所以无特殊需求，仍优先建议使用synchronized（后期性能改进应该会更加偏向于synchronized）。
 ###### 2.非阻塞同步
互斥同步最主要的问题即是线程阻塞和唤醒所带来的性能问题，因此此种同步也称为阻塞同步(Blocking Synchronization)。从处理问题的方式看，互斥同步属于一种悲观锁的并发策略，即无论共享数据是否会真的出现竞争，均需加锁（此处是概念模型，但实际虚拟机会优化掉很大一部分不必要的锁）、用户态核心态转换、维护锁计数器和检查是否有被阻塞的线程需要唤醒等来确保正常的同步。随着硬件指令集的发展，新曾一种选择：基于冲突检测乐观并发策略。即先进行操作，如果没有其他线程争用共享数据则操作成功；若共享数据有争用产生冲突，则采取其他补偿措施（最早常见的则是不断重试直到成功）。这种乐观的并发策略许多实现均不需要把线程挂起，因此此种操作称为非阻塞同步（Non-Blocking Synchronization）。
- 为什么，只是在指令集发展之后才能进行呢？因为需要操作和冲突检测两个步骤具有原子性。若再使用同步互斥保证就没有意义，所以只能依靠硬件完整，硬件需保证从语义上来看需要多次操作的行为只通过一条处理器指令就能完成，这类指令常用的有：
  1. 测试并设置(Test-and-Set)
  2. 获取并增加(Fetch-and-Increment)
  3. 交换(Swap）
  4. 比较并交换（Compare-and-Swap,下文称CAS）
  5. 加载连接/条件存储（Load-Linked/Store-Conditional，下文称LL/SC）。
- 其中前3条已存在于指令集，而后面两条则是现代处理器新增，且这两条指令的目的和功能类似（不同指令集会采用不同的指令实现）。
-CAS指令需3个操作数，分别是变量内存位置（Java中可简单理解为变量的内存地址，用V表示）、旧的预期值（用A表示）和新值（用B表示）。CAS指令执行时，当且仅当V符合旧预期值A时，处理器用新值B更新V的值，否则它就不执行更新。但无论是否更新了V的值，都会返回V的旧值，上述的处理过程是一个原子操作。
- JDK1.5之后，Java程序才可使用CAS操作，该操作由sun.misc.Unsafe类的compareAndSwapInt()和compareAndSwapLong()等几个方法包装提供，虚拟在内部对这些方法做了特殊处理，即时编译出来的结果是一条平台相关的CAS指令，无方法调用过程，可认为无条件内联（称为固有函数，Math.sin（）类似）。
- 之前提到过jdk的保护机制，默认是不程序使用使用sun.misc包的接口，且因其平台相关性，Unsafe类并不直接提供给用户程序调用（Unsafe.getUnsafe()方法中限制仅启动类加载器加载的Class才能访问）。因此，若不采用反射手段，只能通过其他Java API间接访问，如JUC包的整数原子类，其中compareAndSet()和compareIncrement()等方法都使用Unsafe类的CAS操作。
```language
package com.test.jvm.concurrent;

import java.util.concurrent.atomic.AtomicInteger;

public class AtomicTest {
	
	public static int normao_i = 0 ;
	
	public static AtomicInteger atomic_i = new AtomicInteger(0);
	
	private final static int THREAD_COUNT = 20 ;	
	
	public static void increase(){
		normao_i++;
		atomic_i.incrementAndGet();
	}
	

	public static void main(String[] args) {
		Thread[] ts = new Thread[THREAD_COUNT];
		for(int i=0;i<ts.length;i++){
			ts[i] = new Thread(new Runnable(){

				@Override
				public void run() {
					for(int i=0;i<1000;i++){
						increase();
					}
				}
				
			});
			ts[i].start();
		}
		
		while(Thread.activeCount()>1){
			Thread.yield();
		}
		
		System.out.println("normao_i:" + normao_i);
		System.out.println("atomic_i:" + atomic_i);
	}
}

/***
 * 运行结果：
 *  normao_i:19931
    atomic_i:20000
 * */

```
- 通过对比结果那可发现 i++与 AtomicInteger 差异。而icrementAndGet方法源码如下：
```language
    /**
     * Atomically increments by one the current value.
     *
     * @return the updated value
     */
    public final int incrementAndGet() {
        for (;;) {
            int current = get();
            int next = current + 1;
            if (compareAndSet(current, next))
                return next;
        }
    }
```
- 比如简单，即重复获取-更新，compareAndSet方法只有在原子CAS 更新成功时返回true。基于乐观锁，无限重试（若更新失败则重新获取并比较更新）直到成功。
- 单从线程安全的自增实现来看确实非常好，但这咱
