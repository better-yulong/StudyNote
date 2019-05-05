Singleton指仅仅被实例化一次的类。Singleton通常被用来代表那些本质上唯一的系统组件，比如窗口管理器或文件系统。使类成为Singleton会使它的客户端测试变得相对困难，因为无法给Singleton替换模拟实现，除非它实现一个充当其类型的接口。
1. Singleton 实现方法一：私有构造器+final static 成员
```language
public class SingletonTest1 {
	
	public final static SingletonTest1 st1 = new SingletonTest1();
	
	private SingletonTest1(){
		
	}

}
```
私有构造器仅被调用一次，用来实例化公有的静态final域  SingletonTest1.st1 。因缺少公有或受保护的构造器，可保证SingletonTest1全局唯一性：一旦SingletonTest1 类被实例化只会存在一个SingletonTest1实例。但是客户端仍有可能借助 AccessibleObjet.setAccessible方法，通过反射机制调用私有构造器，即破坏单例模式。如若想抵御此种攻击，则可修改构造器，使其在被要求第二次创建实例时抛出异常。
2. Singleton 实现方法二：私有构造器 + 私有final static成员 + 公有静态工厂方法
```language
public class SingletonTest2 {
	
	private final static SingletonTest2 st2 =  new SingletonTest2();
	
	private SingletonTest2(){
	}

	public SingletonTest2 getInstance(){
		return st2 ;
	}
}
```
- 对于SingletonTest2.getInstance()的所有调用都会返回同一个对象，不会合建新的实例（但上述AccessibleObjet.setAccessible 也同样存在）。公有域方法的好处是组成类的成员声明表明该类是一个Singleton：final的静态域，所以该域总是包含相同的对象引用。公有域方法在性能上不再有任何优势：现代JVM实现都能够将静态工厂方法的调用内联化。
- 但工厂方法优势之一是提供了灵活性：在不改变API的前提下，可改变该类是否应该为Singleton的想法，即工厂方法返回该类的唯一实例，但也可容易修改为每个调用方法的线程返回一个唯一实例，这点在实现方法一基于公有fianl的静态域则很难做到。优势二与泛型相关（后续再理解）
- 为了确保Singleton类是可序列化的（Serializable),仅仅在声明中加上"implements Serializable"是不够的。为了维护并保证Singleton，必须声明所有实例域都是瞬时(transient)，并提供一个readResolve方法。否则，每次反序列化一个序列化的实例时，都会创建一个新的实例。即为了解决该序列化导致的单例破坏，需在类中加入readResolve方法：