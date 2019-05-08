先不说其他，来段示例代码：
```language
public class ObjectStaticInitTest {

	public static void main(String[] args) {
		staticFunction();
	}
	
	static ObjectStaticInitTest os = new ObjectStaticInitTest();
	
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
- 这里主要的点之下是：