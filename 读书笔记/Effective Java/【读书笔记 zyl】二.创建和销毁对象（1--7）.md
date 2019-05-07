### 第1条：考虑用静态工厂方法代替构造器
对于类而言，为了获取它自身的一个实现，最常用的方法是提供公有的构造器；但还有一种方法，则是类可提供一个公有的静态工厂方法（static factory method），它只是一个返回类的实例的静态方法。如：
```
    public static final Boolean TRUE = new Boolean(true);
    public static final Boolean FALSE = new Boolean(false);
    public static Boolean valueOf(boolean b) {
        return (b ? TRUE : FALSE);
    }
```
> 注意：静态工厂方法与设计模式中的工厂方法模式不同，并不直接对应。
- 类通过静态工厂方法提供给外部使用而不是通过构造器，优势如下：
#### 静态工厂方法优势
##### 1. 静态工厂方法有名称
一个类只能有一个带有自指定签名的构造器（后面实例理解）。虽然可通过重载方式，即调整参数顺序、个数等来避开限制。但这并非好主意，因为用户永远也记不住该用哪个构造器，往往可能调用错误（类似情况还真遇到过，同名类同方法但包不同，而两个参数类型顺序相反；踩了个坑...）。故当一个类需要多个带有相同签名的构造器时，可用静态方法代替构造器，并且需慎重运行名称以便突出区别、达到见名知意。
```language
public class ConstructorMulitTest {
	
	ConstructorMulitTest(){
		
	}
	ConstructorMulitTest(int a,String b){
		
	}
	ConstructorMulitTest(String b,int a){
		
	}

}
```
```language
 //通过javap 命令可发现虽然有三个构造函数，但实际class常量池只有一个Object类init对应的Methodref，
 //而三个构造函数实际在字节码指令中均指向该符号引用
D:\work\workspace\work2\effective-demo\src\main\java\com\zyl\effective\demo>javap -s -v -p ConstructorMulitTest.class
Classfile /D:/work/workspace/work2/effective-demo/src/main/java/com/zyl/effective/demo/ConstructorMulitTest.class
  Last modified 2019-4-23; size 385 bytes
  MD5 checksum 474067a831b8aa43e87e157aaaa530c4
  Compiled from "ConstructorMulitTest.java"
public class com.zyl.effective.demo.ConstructorMulitTest
  SourceFile: "ConstructorMulitTest.java"
  minor version: 0
  major version: 51
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
   #1 = Methodref          #3.#12         //  java/lang/Object."<init>":()V
   #2 = Class              #13            //  com/zyl/effective/demo/ConstructorMulitTest
   #3 = Class              #14            //  java/lang/Object
   #4 = Utf8               <init>
   #5 = Utf8               ()V
   #6 = Utf8               Code
   #7 = Utf8               LineNumberTable
   #8 = Utf8               (ILjava/lang/String;)V
   #9 = Utf8               (Ljava/lang/String;I)V
  #10 = Utf8               SourceFile
  #11 = Utf8               ConstructorMulitTest.java
  #12 = NameAndType        #4:#5          //  "<init>":()V
  #13 = Utf8               com/zyl/effective/demo/ConstructorMulitTest
  #14 = Utf8               java/lang/Object
{
  com.zyl.effective.demo.ConstructorMulitTest();
    Signature: ()V
    flags:
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 5: 0
        line 7: 4

  com.zyl.effective.demo.ConstructorMulitTest(int, java.lang.String);
    Signature: (ILjava/lang/String;)V
    flags:
    Code:
      stack=1, locals=3, args_size=3
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 8: 0
        line 10: 4

  com.zyl.effective.demo.ConstructorMulitTest(java.lang.String, int);
    Signature: (Ljava/lang/String;I)V
    flags:
    Code:
      stack=1, locals=3, args_size=3
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 11: 0
        line 13: 4
}
*/
```
##### 2. 不必在每次调用时都创建一个新对象。
使得不可变类可以使用预先构建好的实例，或者将构建好的实例缓存进行重复利用，从而避免创建不必要的重复对象。如上Boolean.valueOf(boolean)方法，类似于Flyweight模式(享元模式)，适用于频繁请求创建相同对象且创建对象代价较高的场景，该项技术可极大提升性能。
  - 静态工厂方法能够为重复的调用返回相同对象，有助于类严格控制在某个时刻哪些实例存在，这种类称为实例受控的类(instance-controlled）。编写实例受控类可使得类可确保它是一个Singleton或者是不可实例化的；它还可使得不可变的类可以确保不会存在两个相等的实例，即当且仅当a==b的时候才有a.quals(b)为true。若如此，则可直接使用==操作符来代替equals（Object）方法，可显示提升性能。枚举（enum）类型保证了这一点。
##### 3. 可返回原返回类型的任何子类型对象，相对灵活。
这种灵活的一种应用是：API可返回对象，同时又不会使对象的类变成公有（可私有构造函数）。使用该方法隐藏实现类会使API非常简洁，这项技术适用于基于接口的框架（interface-based-framework）。因为在这种框架中，接口为静态工厂方法提供了自然返回类型。接口不能有静态方法（原因：接口只能定义方法，而由其子类实现；静态方法则必须在声明的同时实现。接口中abstract 修饰的方法，这意味着该方法实现各不相同(即使你故意做一致实现)，则不能称之为类方法。这与static 作用想矛盾，即希望所有子类使用完全相同的实现。此种场景可考虑结合接口与抽象类共同实现），因此接口Type的静态工厂方法被放在一个名为Typesr的不可实例化的类中，如Java Conllections Framework API。
  - 公有的静态工厂方法所返回的对象的类不仅可是非公有的，而且该类还可随着每次调用而发生变化，这取决于静态工厂参数值。只要是已声明的返回类型的子类都是允许的，同时为提升软件的可维护性和性能，返回对象也可随着发行版不同而不同。
  - 如java.util.EnumSet类noneOf方法可获得EnumSet实例，但底层根据枚举元素个数，会分别有不同的实现：
  ```language
public static <E extends Enum<E>> EnumSet<E> noneOf(Class<E> elementType) {
        Enum[] universe = getUniverse(elementType);
        if (universe == null)
            throw new ClassCastException(elementType + " not an enum");

        if (universe.length <= 64)
            return new RegularEnumSet<>(elementType, universe);
        else
            return new JumboEnumSet<>(elementType, universe);
    }
```
  - 而底层根据元素长度的不同实现用户无感知且无需关注，而在后期的版本则轻易去除其中一种实现或添加其他更好的实现，而使用时不用关注实际对象的类类型，因其都是EnumSet的子类。
  - 静态工厂方法返回的对象所属的实际类，在编写包含该静态工厂方法的类时可不必存在，这种灵活的静态工厂方法构成了服务提供者框架（Service Provider Framework），如JDBC API。服务提供者框架指的是一个系统：多个服务提供者实现一个服务，系统为服务提供者的客户端提供多个实现，并把他们从多个实现中解耦出来。
  - 服务提供者框架有三个重要组件：服务接口(Service Interface），提供给服务提供者实现的；提供者注册API（Provider Registration API），系统用来注册实现，让客户端访问；服务访问API（Service Access API），用户用来获取服务实例。服务访问API一允许但是不要求客户端指定某种选择提供者的条件；若没有这样的规定，则API返回默认实现的一个实例。服务访问API是"灵活的静态工厂"，它构成了服务提供者框架的基础。
  - 服务提供者框架的第四个组件是可选的：服务提供者接口（Service Prodiver Interface，SPI），这些提供者负责创建其服务实现的实例。如果没有服务提供者接口，实现就按类名称注册，并通过反射方式进行实例化。对于JDBC而言，Connection是它的服务接口，DriverManager.registerDrive是提供者注册API，DriverManger.Connection是服务访问API，Driver就是服务提供者接口。
  - 服务提供者框架都着无数种变体。如服务访问API可利用适配器(Adapter)模式，返回比提供者更丰富的服务接口。

###### 3.1 深入研究服务提供者框架（SPF）及服务提供者接口(SPI):
因服务提供者框架（SPF）及服务提供者接口(SPI)其非常实用 ，故该篇深入研究，SPF 包括以下组件:
 ![SFP组件](https://github.com/better-yulong/StudyNote-Resource/blob/master/StudyNote-Resource/effective-java-pic/2-001.PNG)
可参考理解：https://www.jianshu.com/p/72d1b41f7cde、https://juejin.im/post/59ffbe8ef265da4309448ae2
 ![SFP组件交互图](https://github.com/better-yulong/StudyNote-Resource/blob/master/StudyNote-Resource/effective-java-pic/2-002.PNG)
如何理解呢？
- 服务接口（指interface）：服务接口中定义一些提供具体服务的方法，如java.sql.Connection，其中提供了熟悉的createStatement()、prepareStatement()、commit()等方法（使用过jdbcTemplate会比较熟悉））（对应Oracle数据库的OracleConnection）
- 服务提供者接口(指interface)：即服务提供者的接口。按服务接口理解，那就是Connection提供者接口，即可以提供Connection实例，可理解为java.sql.Driver,其方法(String url, java.util.Properties info)可通过指定的数据库连接URL及属性配置返回Connection实例（对应Oracle数据库的OracleDriver）
- 提供者注册API（指方法）：服务提供者接口的具体实现类里面去注册这个API，如java.sql.DriverManager的registerDriver()。既然服务服务提供者接口是Driver，那用来注册和管理的类自然就是DriverManager，其中registeredDriver()方法即将当前第三方服务的服务提供者的实现OracleDriver注册到服务列表registeredDrivers。而第三方服务实现可同时支持多个，所以registeredDrivers为列表。
- 服务访问API（指方法）：服务访问API是“灵活的静态工厂”，它构成了服务提供者框架的基础; 获得具体服务的实例。即对应
DriverManager的 getConnection()方法。
- 具体demo示例可参考"【实践2】SPI 入门demo1 "
###### 3.2 SPI原理深究
在SPI示例中，提到需在服务接口具体实现类所在jar的资源目录META-INF/services中放置提供者配置文件 来标识服务提供者。而对于为何简单配置即可？ ServiceLoader是在 jdk1.6开始引入的，它主要的功能是用来完成对SPI的provider的加载。SPI 是JDK 内置的服务发现机制，父类加载器请求子类加载器去完成类加载的动作，其违反了双亲委派模型的一般性原则。因为双亲委派模式虽然很好的解决了各类加载器的基础类的统一问题，但是当你JDK定义的基础类需要去调用（加载）对应的自适应性扩展的类的时候，双亲委派就无法实现，需依赖于SPI。
##### 4.创建参数化类型实例，代码更简洁
如大家常用的创建HashMap的代码：
```language
Map<Sting,List<String>> m = new HashMap<Sting,List<String>>();
```
但若HashMap有提供静态工厂方法，即编译器可帮忙找到类型，可理解为类型推导（type inference），假设HashMap提供如下静态工厂方法，则使用会更简单：
```language
public static <K,V> HashMap<K,V> newInstance(){
 	return new HashMap<K,V>();
}

使用调整为：Map<Sting,List<String>> m = HashMap.newInstance();
```
实际HashMap 并未提供 newInstance方法哈，仅用于举例哈。但是从JDK 1.7开始，基于类型推导与泛型方法的改进，如上的实例化代码可简化为：Map<Sting,List<String>> m = new HashMap<>(); 注意后面的<>必须，否则认为是默认无泛型的实例化。

#### 静态工厂方法缺点
##### 1.类如果没有公有或受保护的构造器，则不能被子类化
具体怎么理解呢？即若类的默认构造函数为private，则子类无法继承：Implicit super constructor ConstructorPrivateTest() is not visible for default constructor. Must define an explicit constructor。
这个地方需要注意，是指默认的无参构造函数为私有则无法继承。虽然基于多态可有多个有参的构造函数，但影响继承的仅默认无参构造函数。
##### 2.用于控制实例生成，但与其他静态方法实际无任何区别，即目前API及DOC文档并未像类构造函数那样明确标识。但基于习惯，静态工厂方法仍有一些惯用名称：valueOf、of、getInstance、newInstance、getType、newType.

- 总而言之，静态工厂方法与公有构造器各有用处，需理解各自长处。静态工厂通常更加合适，因此切忌第一反应是提供公有构造器，而不优先考虑静态工厂。

### 第2条：遇到多个构造器参数时要考虑用构建器
- 静态工厂与构造器有共同的局限：不能很好的扩展到大量的可选参数。
1. 如构造器对应4个必选参数，至多20个多组的非必选参数。那么习惯会采用重叠构造器（telescoping constructor），即先提供一个只有必要参数的构造器，之后提供多个包含可选参数的构造器，最后提供一个包含所有参数的构造器。而使用此种方式会导致通常需给很多本不想设置的参数传递值，且参数数目增加很容易失去控制。即：重叠构造器模式可行，但当参数过多，客户端代码会很难编写，且难以阅读。
2. 那么针对该情况，有种替代方法，即JavaBeans模式：无参构造器创建对象，然后调用setter()方法来设置必要参数及可选参数。但是该模式亦有严重缺点：构造过程被拆分成多次调用，使得构造过程中JavaBean可能处于不一致状态；类无法通过参数检验构造器参数的有效性来保证一致性。同时JavaBeans模式则使得类不能为不可变，需要额外考虑其线程安全。
3. 而第三种替换方法，即能保证像重叠构造器模式的安全性，也可保证JavaBeans模式好的可读性，即Builder模式的一种形式。即让客户端利用所有必要的参数调用构造器（或静态工厂），得到builder对象；之后客户端通过builder对象上调用类似于setter方法来设置相关可选参数。
 - 如用一个类表示包含食品外的营养成分标签，其中有必须域：每份含量、每罐含量、每份卡路里，还有超过20个可选域：总脂肪量、饱和脂肪量、转化脂肪、胆固醇、钠等。如若采用重叠构造器或JavaBeans模式均不太好编写。
 ```language

public class NutritionFacts {

	private final int servingSize ;
	private final int servings ;
	private final int calories;
	private final int fat ;
	private final int sodium ;
	private final int carbohydrate ;
	
	public static class Builder{
		//required paremeters
		private final int servingSize ;
		private final int servings ;
		
		//Optional paremeters
		private int calories = 0 ;
		private int fat = 0 ;
		private int sodium = 0 ;
		private int carbohydrate = 0 ;
		
		public Builder(int servingSize,int servings){
			this.servingSize = servingSize ;
			this.servings = servings;
		}
		
		public Builder calories(int val){
			this.calories = val;
			return this;
		}
		
		public Builder fat(int val){
			this.fat = val;
			return this;
		}
		
		public Builder sodium(int val){
			this.sodium = val;
			return this;
		}
		public Builder carbohydrate(int val){
			this.carbohydrate = val;
			return this;
		}
		
		public NutritionFacts build(){
			//optional: add paremeter check --- 风险
			return new NutritionFacts(this);
		}
	}
	
	private NutritionFacts(Builder builder){
		//optional: add paremeter check --- 风险
		this.servingSize = builder.servingSize ;
		this.servings = builder.servings ;
		this.calories = builder.calories ;
		this.fat = builder.fat ;
		this.sodium = builder.sodium ;
		this.carbohydrate = builder.carbohydrate ;
		//optional: add paremeter check --- 正确（参考第39条：Time-Of-Check）

	}
	
	public static void main(String[] args){
		NutritionFacts cacaCola = new NutritionFacts.Builder(240, 8).calories(100).sodium(20).carbohydrate(10).build();
		System.out.println(cacaCola.servingSize);
		System.out.println(cacaCola.sodium);
		System.out.println(cacaCola.fat);
	}
}

/***
 * Builder构建器模式，区别于重叠构造器、JavaBeans模式，可保证线程全亦可有较好的可读性.
 * 
 * 运行结果：
 *  240
	20
	0

 * */

```
- builder类似于构造器，可对参数强加约束条件，build方法可检验这些约束条件。将参数从builder拷贝到对象中之后，并在对象域而不是builder域对它们进行检验。如上注释三个地方"optional: add paremeter check"，但建议最优的是是对象构造函数最后检查参数，且若参数不合法，需显示报出IllegalStateException（且异常信息应该显示出违反了哪个约束条件，见第63条）；另一个对参数强加约束的方法则是在setter方法中对参数进行检查。若约束条件未满足，则setter方法抛出IllegalStateException，好处是一旦传递参数无效可立即发现约束失败，而不是调用builder方法才抛出。
- 与构造器相比，builder微略优势在于，builder 可有多个可变(varargs)参数；而构造器只能有一个可变参数。因为builder方法利用单独的方法设置参数，甚至每个setter方法均可有一个可变参数。
- Builder模式十分灵活，可利用单个Builder构建多个对象；builder参数可在创建对象期间进行调整，也可随着不同对象而改变。builder可自动填充某些域，例如每次创建对象时自动增加序列号。
- 设置了参数的builder生成一个很好的抽象工厂。换种说法，客户端可将builder对象传给方法，而该方法可根据需要需要创建一个或多个。如基于泛型：
```language
	// A Builder for objects of type T
	public interface Builder<T>{
		public T build();
	}
```
- Builder实例方法通常利用有限制的通配符类型来约束构建器的类型参数。如 tree buildTree(Builder<？ extends Node> nodeBuilder){...}
- Java传统的抽象工厂实现是Class对象，利用其newInstance方法充当build方法的一部分，但隐含诸多问题：newInstance方法总是企图调用类的无参构造器，而该构造器可能都不存在，同时也不会收到编译错误；客户端代码必须运行时处理InstantiationException或者IllegalAccessException，不雅观也不方便；newInstance方法还会传播无参构造器抛出的所有异常，即使newInstance缺乏相应throws子句。总的来说，Class.newInstance 破坏了编译时的异常检查，而Builder模式可弥补这些不足。
- 但Builder模式也有不足。为了创建对象，必须先创建其构造器；该部分开销一般不明显，但在十分注重性能的情况下，可能成为问题。Builder模式与重叠构造器模式更加冗长，其只适合在参数很多的场景使用，比如4个或更多。但参数可能未来会增加，所以需要权衡是否一开始就使用构造器。

### 第3条：使用私有构造器或者枚举类型强化Singleton属性
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
```language

public class SingletonTest2 implements Serializable{
	
	private final static SingletonTest2 st2 =  new SingletonTest2();
	
	private SingletonTest2(){
	}

	public SingletonTest2 getInstance(){
		return st2 ;
	}
	
	//实现了Serializable接口的单例，为避免反序列化破坏单例，需添加该方法
	private Object readResolve(){
		return st2 ;
	}
	
	
}
```
2. Singleton 实现方法三（Java 1.5开始） ：枚举
```language
public enum SingletonEnumSimple {
	INSTANCE;
}

```
枚举方式相对更加简洁，可防范如上实现序列化或者反射攻击导致的单例破坏。可以说，单元素的枚举类型已经成为Singleton的最佳方法。至于如下相对复杂的枚举实现，具体合理性等还有待后面理解，先示例：
```language
public enum SingletonEnum {
	
	INSTANCE_SUCC("00","SUCCESS"),
	INSTANCE_FAIL("99","FAIL");
	
	private String key ;
	private String value;
	
	SingletonEnum(String key,String value){
		this.key = key;
		this.value = value;
	}
	
	static String getValue(String key){
		for(SingletonEnum se:SingletonEnum.values()){
			if(se.key.equals(key))
					return se.value;
		}
		return null;
	}

}
```
```language

```

### 第4条：通过私有构造器强化不可实例化的能力
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

//此处在IDE会提示错误：Implicit super constructor PrivateConstructorParent() is not visible for default constructor. Must define an explicit 
class PrivateConstructorChild extends PrivateConstructorParent{
	
}
```

### 第5条：避免创建不必要的对象
一般来说，最好能重用对象而不是每次需要时都创建一个相同功能的新对象。重用方式即快速，又流行。若对象是不可变的（immutable），则其始终可被重用。
```language
	//反面示例
	String s = new String("string temp"); //DON'T DO THIS
```
#### 1.字面量、静态工厂方法
该语句会使得每次执行时都创建一个新的String实例，但是这些创建动作全都是不必要。传递给String构造器的参数（"string temp"）本身就是一个String实例，功能方面等同于构造器所创建的所有对象。如果这种方法在一个循环或者是在一个被频繁调用的方法中，就会创建出成千上万不必要的String实例，可考虑改进为：
```language
    String s = "string temp" ; 
    //改进版本，基于字面量、字符串常量池。String.intern()也可避免多次创建新String对象，但若对其了解不充分则不建议使用。
```
此种方式只生成一个String实例，而不是每次执行都创建新的实例。其可保证，对于所有在同一台虚拟机中运行的代码，只要它们包含相同的字符串字面量，该对象就会被重用。
- 对于同时提供了静态工厂方法和构造器的不可变类，通常可优先使用静态工厂方法而不是构造器，以避免合建不必要的对象。例如，静态工厂方法Boolean.valueOf(String)几乎总是优先于构造器new Boolean(String)。构造器在每次被调用时都会创建一个新的对象，而静态工厂方法则不要求这样做，实际也不会如此做。
#### 2.局部变量改为final静态域
- 除了重用不可变对象外，也可重用那些已知不会被修改的可变对象。即把已知不会被修改的局部变量改为final静态域，将其作为常量对象，代码更易于理解，且对于频繁调用性能可能会有显著提高（效果依赖于创建对象成本，即该对象创建代价，如Calendar对象相对创建代价较大--- 人个理解：如若涉及native调用应该代价会相对较高）
- 延迟初始化(lazily initializing)虽可消除这些不必要的初始化工作，但实际不建议，因延迟初始化会使方法的实现更加复杂，从而可能使得性能的提升效果打折扣。
- 如适配器模式，适配器是指这样一个对象：把功能委托给一个后备对象(backing object)，从而为后备对象提供一个可替代的接口。其无状态，无需合建多个适配器实例。（Adapter(适配器类):它可以调用另一个接口,作为一个转换器,对Adaptee和Target进行适配。它是适配器模式的核心。）
- 例如Map接口的keySet()方法返回该Map对象的Set视图，其中包含该Map中所有的键(key），而实际每次调用keySet()方法都会返回同一个Set实例（Set实例对象本身可能随操作发生变化，但对象始终为同一个）
#### 3.自动装箱（Java 1.5开始）
自动装箱使得程序员可将基本类型与装箱基本类型（Boxed Primitive Type）混用，从语义上使得基本类型与装箱基本类型差别更模糊，但在性能上则可能有明显的差别（第49条）
```language
public class BoxPerformTest {

	public static void main(String[] args) {
		Date startDate = new Date();
		System.out.println("start: " + startDate.getTime());
		
		Long sum = 0L ;
		for(long i=0 ;i<Integer.MAX_VALUE;i++){
			sum += i ;
		}
		System.out.println("sum:" + sum);
		Date middleDate = new Date();
		System.out.println("first: " + (middleDate.getTime() - startDate.getTime()));
		
		
		long sum2 = 0L ;
		for(long j=0 ;j<Integer.MAX_VALUE;j++){
			sum2 += j ;
		}
		System.out.println("sum2:" + sum2);
		
		Date endDate = new Date();
		System.out.println("end: " + (endDate.getTime() - middleDate.getTime()));
	}

}
```
运行结果：
```language
start: 1557132832635
sum:2305843005992468481
first: 32906
sum2:2305843005992468481
end: 5906

```
可发现，唯一的区别在于sum、sum2数据类型的差异（sum为Long类型，sum2为基本数据类型），而每次往Long sum 中增加i时均城构造一个新的Long实例，使得整体会多构造约2的31次方个多余的Long实例。即使多次运行，sum的计算始终与sum2的计算需多花数倍的时间（32秒与6秒）。
- 结论1：要优先使用基本数据类型而不是装箱基本类型，避免无意识的自动装箱。
- 结论2：不能错误的认为"创建对象的代价非常昂贵，就应尽可能的避免重复创建对象"。相反，小对象的构造器可能只做少量工作且其创建、回收动作非常廉价。有时，通过创建附加对象，提升程序的清晰性、简洁性和功能性，反面会是件好事。反之，通过维护自己的对象池(object pool)来避免创建对象并不是一种好的做法，除非池中的对象是非常重量级的。正确使用对象池的典型示例就是创建数据库连接池，因创建数据库的连接代价是非常昂贵的，且数据库可能会限制只允许使用一定数量的连接。
- 该规则与第39条："当你应该创建新对象时，请不要重用现有的对象"，该规则仅仅是针对"保护性拷贝"，因保护性拷贝重用对象的代价要远大于因创建重复对象而付出的代价。

### 第6条：消除过期的对象引用
先看简单的示例：
```language
import java.util.Arrays;
import java.util.EmptyStackException;

public class ObjectExpireStackMemoryLeak {
	
	private Object[] elements ;
	
	private final static int CAPACITY_INITIAL = 16 ;
	
	private static int size = 0 ;
	
	public ObjectExpireStackMemoryLeak(){
		elements = new Object[CAPACITY_INITIAL];
	}
	
	public Object pop(){
		if(size ==0 ){
			throw new EmptyStackException();
		}
		return elements[--size] ;
	}
	
	public void push(Object obj){
		ensureCapacity();
		elements[size++] = obj ;
	}
	
	
	/**
	 * ensure space for at least one more element,roughly doubling the capacity each time the array needs to grow
	 * */
	private void ensureCapacity(){
		if(elements.length == size){
			elements  = Arrays.copyOf(elements, 2*size + 1);
		}
		
	}

}
```
```language
public class ObjectExpireStackMemoryLeakTest {

	public static void main(String[] args) throws Exception {
		Thread.currentThread().sleep(1000*50l);
		ObjectExpireStackMemoryLeak stack = new ObjectExpireStackMemoryLeak();
		System.out.println("start push.......");
		for(int i=0 ; i<10000000; i++){
			 stack.push(new Object());
		}
		System.out.println("end push.......");
		Thread.currentThread().sleep(1000*10l);
		System.out.println("start pop.......");
		for(int i=0 ; i<10000000-1; i++){
			 stack.pop();
		}
		System.out.println("end pop.......");
		Thread.currentThread().sleep(1000*1000l);

	}

}
```
运行ObjectExpireStackMemoryLeakTest ，并打开jvm内存监控：jstat -gcutil 8524 1000 1000000 ，生成日志片断如下：
```language
S0     S1     E      O      P     YGC     YGCT    FGC    FGCT     GCT   
  0.00   0.00  11.65   0.00   1.08      0    0.000     0    0.000    0.000
  0.00   0.00  11.65   0.00   1.08      0    0.000     0    0.000    0.000
  0.00   0.00  11.65   0.00   1.08      0    0.000     0    0.000    0.000
  0.00   0.00  11.65   0.00   1.08      0    0.000     0    0.000    0.000
  0.00   0.00  11.65   0.00   1.08      0    0.000     0    0.000    0.000
  0.00   0.00  11.65   0.00   1.08      0    0.000     0    0.000    0.000
  0.00   0.00  11.65   0.00   1.08      0    0.000     0    0.000    0.000
  0.00   0.00  11.65   0.00   1.08      0    0.000     0    0.000    0.000
  0.00   0.00  11.65   0.00   1.08      0    0.000     0    0.000    0.000
  0.00   0.00  11.65   0.00   1.08      0    0.000     0    0.000    0.000
  0.00   0.00  11.65   0.00   1.08      0    0.000     0    0.000    0.000
  0.00   0.00  11.65   0.00   1.08      0    0.000     0    0.000    0.000
  0.00   0.00  11.65   0.00   1.08      0    0.000     0    0.000    0.000
  0.00   0.00  11.65   0.00   1.08      0    0.000     0    0.000    0.000
  0.00   0.00  11.65   0.00   1.08      0    0.000     0    0.000    0.000
  0.00   0.00  11.65   0.00   1.08      0    0.000     0    0.000    0.000
  0.00   0.00  11.65   0.00   1.08      0    0.000     0    0.000    0.000
  0.00   0.00  11.65   0.00   1.08      0    0.000     0    0.000    0.000
  0.00   0.00  11.65   0.00   1.08      0    0.000     0    0.000    0.000
  0.00   0.00  11.65   0.00   1.08      0    0.000     0    0.000    0.000
  0.00   0.00  11.65   0.00   1.08      0    0.000     0    0.000    0.000
  0.00   0.00  11.65   0.00   1.08      0    0.000     0    0.000    0.000
  0.00   0.00  11.65   0.00   1.08      0    0.000     0    0.000    0.000
  0.00 100.00   0.00  99.84   1.09      7    0.353     3    0.214    0.567
  7.35   0.00  82.14  99.73   1.09     10    0.691     4    0.839    1.530
  0.00 100.00   0.00  99.91   1.09     11    1.077     5    0.839    1.916
  3.68   0.00  11.75  99.86   1.09     12    1.155     5    1.469    2.624
  3.68   0.00  11.75  99.86   1.09     12    1.155     5    1.469    2.624
  3.68   0.00  11.75  99.86   1.09     12    1.155     5    1.469    2.624
   ..... 后面数十行，但与上面一行完全一样，直到程序运行结束仍然一样
```
- 从上面的日志可看出，在push期间年轻代使用逐渐增多，后续触发YGC年轻代使用减小老年代增加至稳定；而在调用pop方法后，内存使用并没有任何变化（如若了解该场景的同学分析如上代码其实可发现原因，是因为pop方法并未释放掉之前push对象的引用），即并没有发生GC而使得内存如预期回收。
- 原因其实很简单：Object[]数组对象elements仍然持有之前push所有的对象的引用，pop方法仅是调整了elements数组的长度（可理解为状态）。其实有分析过集合类ArrahList等集合类、ThreadLocal源码的同学会发现该示例似曾相识？先贴段源码：
```language
ArrayList源码片段：
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable
{
	    private transient Object[] elementData;
	    private int size;

	public ArrayList(int initialCapacity) {
        	super();
        	if (initialCapacity < 0)
            		throw new IllegalArgumentException("Illegal Capacity: "+
                                               initialCapacity);
        	this.elementData = new Object[initialCapacity];
    	}

	public E remove(int index) {
        	rangeCheck(index);

        	modCount++;
        	E oldValue = elementData(index);

        	int numMoved = size - index - 1;
        	if (numMoved > 0)
            	System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
        	elementData[--size] = null; // Let gc do its work

        	return oldValue;
    	}	
      ........
}
```
```language
HashMap源码片段：
public class HashMap<K,V>
    extends AbstractMap<K,V>
    implements Map<K,V>, Cloneable, Serializable
{
	 static final int DEFAULT_INITIAL_CAPACITY = 16;
	 transient Entry[] table;
	 transient int size;
           public V remove(Object key) {
        Entry<K,V> e = removeEntryForKey(key);
        return (e == null ? null : e.value);
    }

    /**
     * Removes and returns the entry associated with the specified key
     * in the HashMap.  Returns null if the HashMap contains no mapping
     * for this key.
     */
    final Entry<K,V> removeEntryForKey(Object key) {
        int hash = (key == null) ? 0 : hash(key.hashCode());
        int i = indexFor(hash, table.length);
        Entry<K,V> prev = table[i];
        Entry<K,V> e = prev;

        while (e != null) {
            Entry<K,V> next = e.next;
            Object k;
            if (e.hash == hash &&
                ((k = e.key) == key || (key != null && key.equals(k)))) {
                modCount++;
                size--;
                if (prev == e)
                    table[i] = next;
                else
                    prev.next = next;
                e.recordRemoval(this);  
                return e;
            }
            prev = e;
            e = next;
        }

        return e;
    }
	//remove方法间接由上面的recordRemoval调用
       private void remove() {
            before.after = after;
            after.before = before;
        }
```
- 至于上面ArrayList、HashMap将elementData、table 定义为transient 类型后面分析。通过ArrayList、HashMap代码可以发现与上面的例子何其相似，那其如如何解决上面例子pop方法后内存并未如预期回收的呢？
- ArrayList中对应 elementData[--size] = null; // Let gc do its work ；而HashMap则对应 remove() 方法移除指针的引用（HashMap基于数组+链表方式实现），那该如何修改上面示例可解决内存泄漏呢？看懂了ArrayList源码应该就知道相当简单了。但实际验证还是尝试了多次修改才达到预期。

 
