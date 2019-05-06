- 有时，可能只需要编写包含静态方法的静态域的类，甚至很容易被滥用。但尽管如此，其也确有用处，如很多的工具类。抑或如java.lang.Math或者java.util.Arrays的方式，把基本类型的值或者数组类型上的相关方法组织起来；也可通过java.util.Collections，把实现特定接口的对象的静态方法（如工厂方法）组织起来。最后，还可利用这种类把final类上的方法组织起来，以取代扩展该类的做法。
- 此类工具类（utility class）不希望被实例化，实例对它没有意义。然而，在缺少显示构造器的情况下编译器会自动提供一个公有、无参的缺省构造器（default constructor）。
- 故，需强制该类不可被实例化。使用抽象类行不通，因其可被子类化，而子类可被实例化；同时此种方式更可能误导用户，以为这种类是专门为继承而设计。相对简单的习惯用法是让该类包含私有构造器（且防止被破坏可在构造方法中抛出错误），因只有当类不包含显示构造器，编译器才会生成缺省构造器。
```language
public class PrivateConstructor {

	private PrivateConstructor(){
		//防止通过序列化或反射实例化或类内部其他方法调用该构造器
		throw new AssertionError();
	}
}
```
- 此处无疑指出了一个习惯，即所有类最好都显示构造器，且不可实例化的类需针对性防范被实例化。AssertionError不是必须的，但是它可避免不小心在类的内部调用构造器，可保证该类在任何情况下不会被实例化。这种习惯会让人感觉不友好，明智的做法则在代码中增加明确的注释。但此种方式也有副作用，即该类不能被子类化，因子类构造器都必须显示或隐式的调用超类(superclass)构造器，此种情况即会导致子类没有可可访问的超类构造器，报错类：Implicit super constructor PrivateConstructorParent() is not visible for default constructor. Must define an explicit constructor。即超类的默认构造器不可见，需定义一个明确清晰可见的构造器。
```language
class PrivateConstructorParent {
	
	private PrivateConstructorParent(){
		
	}

}

//此处在IDE
class PrivateConstructorChild extends PrivateConstructorParent{
	
}
```
