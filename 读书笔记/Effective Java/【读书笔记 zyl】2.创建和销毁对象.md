### 第1条：考虑用静态工厂方法代替构造器
对于类而言，为了获取它自身的一个实现，最常用的方法是提供公有的构造器；但还有一种方法，则是类可提供一个公有的静态工厂方法（static factory method），它只是一个返回类的实例的静态方法。如：
```language
    public static final Boolean TRUE = new Boolean(true);
    public static final Boolean FALSE = new Boolean(false);
    public static Boolean valueOf(boolean b) {
        return (b ? TRUE : FALSE);
    }
```
> 注意：静态工厂方法与设计模式中的工厂方法模式不同，并不直接对应。
- 类通过静态工厂方法提供给外部使用而不是通过构造器，优势如下：
  1. 静态工厂方法有名称：一个类只能有一个带有自指定签名的构造器（后面实例理解）。虽然可通过重载方式，即调整参数顺序、个数等来避开限制。但这并非好主意，因为用户永远也记不住该用哪个构造器，往往可能调用错误（类似情况还真遇到过，同名类同方法但包不同，而两个参数类型顺序相反；踩了个坑...）。故当一个类需要多个带有相同签名的构造器时，可用静态方法代替构造器，并且需慎重运行名称以便突出区别、达到见名知意。
  ```language
public class ConstructorMulitTest {
	
	ConstructorMulitTest(){
		
	}
	ConstructorMulitTest(int a,String b){
		
	}
	ConstructorMulitTest(String b,int a){
		
	}

}

/*
 * 通过javap 命令可发现虽然有三个构造函数，但实际class常量池只有一个Object类init对应的Methodref，
 * 而三个构造函数实际在字节码指令中均指向该符号引用
 * 
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

  2. 不必在每次调用时都创建一个新对象。使得不可变类可以使用预先构建好的实例，或者将构建好的实例缓存进行重复利用，从而避免创建不必要的重复对象。如上Boolean.valueOf(boolean)方法，类似于Flyweight模式(享元模式)，适用于频繁请求创建相同对象且创建对象代价较高的场景，该项技术可极大提升性能。
  - 静态工厂方法能够为重复的调用返回相同对象，有助于类严格控制在某个时刻哪些实例存在，这种类称为实例受控的类(instance-controlled）。编写实例受控类可使得类可确保它是一个Singleton或者是不可实例化的；它还可使得不可变的类可以确保不会存在两个相等的实例，即当且仅当a==b的时候才有a.quals(b)为true。若如此，则可直接使用==操作符来代替equals（Object）方法，可显示提升性能。枚举（enum）类型保证了这一点。
  3. 可返回原返回类型的任何子类型对象，相对灵活。这种灵活的一种应用是：API可返回对象，同时又不会使对象的类变成公有（可私有构造函数）。使用该方法隐藏实现类会使API非常简洁，这项技术适用于基于接口的框架（interface-based-framework）。因为在这种框架中，接口为静态工厂方法提供了自然返回类型。接口不能有静态方法（原因：接口只能定义方法，而由其子类实现；静态方法则必须在声明的同时实现。接口中abstract 修饰的方法，这意味着该方法实现各不相同(即使你故意做一致实现)，则不能称之为类方法。这与static 作用想矛盾，即希望所有子类使用完全相同的实现。此种场景可考虑结合接口与抽象类共同实现），因此接口Type的静态工厂方法被放在一个名为Typesr的不可实例化的类中，如Java Conllections Framework API。
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
  - 服务提供者框架都着无数种变体。如服务访问API可利用适配器(Adapter)模式，返回比提供者
