Singleton指仅仅被实例化一次的类。Singleton通常被用来代表那些本质上唯一的系统组件，比如窗口管理器或文件系统。使类成为Singleton会使它的客户端测试变得相对困难，因为无法给Singleton替换模拟实现，除非它实现一个充当其类型的接口。
1. Singleton 实现方法一：私有构造器+final static 成员
```language
public class SingletonTest1 {
	
	public final static SingletonTest1 st1 = new SingletonTest1();
	
	private SingletonTest1(){
		
	}

}
```
私有构造器仅被调用一次，用来实例化公有的静态final域 