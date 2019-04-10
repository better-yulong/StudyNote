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
2. Java语言中，大部分的线程安全类都属于相对线程安全，如Vector、HashTable、Collections的synchronizedCollection()方法包装的集合。
###### 4. 线程兼容
线程兼容是指对象本身并不是线程安全（即使单个操作也非线程安全），但可通过在调用端正确的使用同步手段来保证对象在并发环境中可安全使用。平常所说一个类线程不安全，大多数指此种情况。Java API中大部分类都是属于线程兼容，如Vector和HashTable对应的集合类ArrayList和HashMap等。
###### 5. 线程对立
线程对立是指无论调用端是否采取同步措施，但都无法在多线程环境中并发使用。因Java语言天生具备多线程特性，线程对立这种排斥多线程的情况较少且通常有害应该避免。
其中如Thread.suspend()、Thread.resume() 持有同一对象锁而并发执行时则极容易发生死锁，因为suspend方法并不会释放锁,既然suspend和resume使用同一锁那么resume执行前需获得锁才可，即容易造成死锁。除此之外，常见的线程对立操作还有System.setIn()、System.setOut()和System.runFinalizersOnExit()等.
- 既然说到这儿，那对于Thread相关常见的 wait、notify、notifyAll、sleep、suspend、resume、yield、join这些方法的作用进一步理解并汇总：
1. wait、notify、notifyAll是Object类中的方法，而锁即所谓对象的 monitor，所以基于此也可以理解，wait、notify、notifyAll三个方法必须在同步方法(块)中调用，且wait时会释放锁而notify、 notifyAll之后优先获得锁的线程将重新执行。若在非同步方法(块)中调用wait、notify、notifyAll运行时，因当前线程不是对象锁的持有者，则该方法抛出一个java.lang.IllegalMonitorStateException 异常；因一切皆对象，理论上也意味着任何Java代码随时随地均可调用wait、notify、notifyAll方法。
2. sleep(long)、suspend、resume、yield、join这些方法均为Thread类的方法。
   - sleep是当将前被服务的线暂停一段时间之后可恢复继续服务，不会对锁进行操作。
   - suspend、resume方法则对应当前被服务线程的暂停、恢复，不会对锁操作（
