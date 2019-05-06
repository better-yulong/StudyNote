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