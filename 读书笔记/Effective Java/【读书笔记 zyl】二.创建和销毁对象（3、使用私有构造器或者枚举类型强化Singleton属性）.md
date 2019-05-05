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
对于SingletonTest2.