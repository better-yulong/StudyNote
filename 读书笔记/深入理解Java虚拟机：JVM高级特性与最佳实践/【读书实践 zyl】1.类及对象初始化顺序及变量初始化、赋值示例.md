先不说其他，来段示例代码：
```language
public class ObjectStaticInitTest {

	public static void main(String[] args) {
		staticFunction();
	}
	
	static ObjectStaticInitTest st= new ObjectStaticInitTest();
	
	static{
		System.out.println("1");
	}
	
	{
		System.out.println("2");
	}
	
	ObjectStaticInitTest(){
		System.out.println("3");
		System.out.println("a=" + a + ",b=" + b);
	}

	public static void staticFunction(){
		System.out.println("4");
	}
	
	
	int a = 110 ;
	static int b = 112;
}
```
基于之前的理解，分析运行结果应该是：
```language
1
2
3
a=0,b=112
4
```
然而实际呢？
```language
2
3
a=110,b=0
1
4
```
- 其实，整体来说，还是之前理解的不够，基于规则：
1. 父类的静态变量赋值
2. 自身的静态变量赋值
3. 父类成员变量赋值和父类块赋值
4. 父类构造函数赋值
5. 自身成员变量赋值和自身块赋值
6. 自身构造函数赋值
- 这里主要的点之下是：实例初始化不一定要在类初始化结束之后才开始初始化。
- 类的生命周期是：加载--->验证--->准备--->解析--->初始化---->使用--->卸载（其中，偶尔会笼统定义：连接过程包括验证、准备和解析这三个子步骤），只有准备阶段和初始化阶段才会涉及类变量的初始化和赋值，因此只针对这两个阶段进行分析。
- 类的准备阶段需要做的是为类变量分配内存并设置默认值，因此类变量st为null、b为0；（需要注意的是如果类变量为final，编译时javac将会为value生成ConstantValue属性，在准备阶段虚拟机）

