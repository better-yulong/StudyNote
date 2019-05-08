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
b=112
```
然而实际呢？
```language
2
3
a=110,b=0
1
4
b=112
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
- 类的准备阶段需要做的是为类变量分配内存并设置默认值，因此类变量st为null、b为0；（需要注意的是如果类变量为final，编译时javac将会为value生成ConstantValue属性，在准备阶段虚拟机就会根据ConstantValue的设置将变量设置为指定的值，如果这里定义:static final int b = 112,那么在准备阶段b的值就是112,而不再是0了）
- 类的初始化阶段要做的是执行类构造器（类构造器是编译器收集所有静态语句块和类变量的赋值语句按在源码中的顺序合并生成类构造器），因此先执行第一条静态变量的赋值语句st = new ObjectStaticInitTest(),此时会触发对象的初始化，对象的初始化是先初始化成员变量再执行构造方法，因此先打印2 --> 设置a=110 --> 执行构造方法(打印3，此时a已经赋值为110，但是b只设置为了默认值0，并未完成赋值动作)，等对象的初始化完成之后继续执行之前的类构造器语句即会将b设置为112，最后才会执行到staticFunction()方法，因此 打印4 --> 打印b=112
- 这里牵扯到一个知识点：嵌套初始化有一个特别的逻辑。特别是内嵌的变量是静态成员且是本类的实例（单例模式或者静态工厂方法会出现的比较多），这会导致一个有趣的现象：实例初始化竟然在类初始化之前完成。可参考该简化代码来理解：
```language
public class Test {

	public static void main(String[] args) {
		func();
	}
	
	static Test st = new Test();
	static void func(){}

}

```
根据以上代码，关键

```language
反编译class文件：

public class ObjectStaticInitTest
{
  static ObjectStaticInitTest os = new ObjectStaticInitTest();
  int a;
  
  static { System.out.println("1"); }
  
  ObjectStaticInitTest()
  {
    System.out.println("2");
    
    this.a = 110;System.out.println("3");System.out.println("a=" + this.a + ",b=" + b);
  }
  
  public static void staticFunction() {
    System.out.println("4");
  }
  
  static int b = 112;
  
  public static void main(String[] paramArrayOfString) {}
}

```
```language
javap -v -l -c -s ObjectStaticInitTest.class结果：
Classfile /D:/work/workspace/work2/effective-demo/src/main/java/com/zyl/effective/demo/other/ObjectStaticInitTest.class
  Last modified 2019-5-8; size 1053 bytes
  MD5 checksum 44653e2b8f3cec8358b9c41c36bb91f3
  Compiled from "ObjectStaticInitTest.java"
public class com.zyl.effective.demo.other.ObjectStaticInitTest
  SourceFile: "ObjectStaticInitTest.java"
  minor version: 0
  major version: 51
  flags: ACC_PUBLIC, ACC_SUPER

Constant pool:
   #1 = Methodref          #17.#37        //  com/zyl/effective/demo/other/ObjectStaticInitTest.staticFunction:()V
   #2 = Methodref          #21.#38        //  java/lang/Object."<init>":()V
   #3 = Fieldref           #39.#40        //  java/lang/System.out:Ljava/io/PrintStream;
   #4 = String             #41            //  2
   #5 = Methodref          #42.#43        //  java/io/PrintStream.println:(Ljava/lang/String;)V
   #6 = Fieldref           #17.#44        //  com/zyl/effective/demo/other/ObjectStaticInitTest.a:I
   #7 = String             #45            //  3
   #8 = Class              #46            //  java/lang/StringBuilder
   #9 = Methodref          #8.#38         //  java/lang/StringBuilder."<init>":()V
  #10 = String             #47            //  a=
  #11 = Methodref          #8.#48         //  java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
  #12 = Methodref          #8.#49         //  java/lang/StringBuilder.append:(I)Ljava/lang/StringBuilder;
  #13 = String             #50            //  ,b=
  #14 = Fieldref           #17.#51        //  com/zyl/effective/demo/other/ObjectStaticInitTest.b:I
  #15 = Methodref          #8.#52         //  java/lang/StringBuilder.toString:()Ljava/lang/String;
  #16 = String             #53            //  4
  #17 = Class              #54            //  com/zyl/effective/demo/other/ObjectStaticInitTest
  #18 = Methodref          #17.#38        //  com/zyl/effective/demo/other/ObjectStaticInitTest."<init>":()V
  #19 = Fieldref           #17.#55        //  com/zyl/effective/demo/other/ObjectStaticInitTest.os:Lcom/zyl/effective/demo/other/ObjectStaticInitTest;
  #20 = String             #56            //  1
  #21 = Class              #57            //  java/lang/Object
  #22 = Utf8               os
  #23 = Utf8               Lcom/zyl/effective/demo/other/ObjectStaticInitTest;
  #24 = Utf8               a
  #25 = Utf8               I
  #26 = Utf8               b
  #27 = Utf8               main
  #28 = Utf8               ([Ljava/lang/String;)V
  #29 = Utf8               Code
  #30 = Utf8               LineNumberTable
  #31 = Utf8               <init>
  #32 = Utf8               ()V
  #33 = Utf8               staticFunction
  #34 = Utf8               <clinit>
  #35 = Utf8               SourceFile
  #36 = Utf8               ObjectStaticInitTest.java
  #37 = NameAndType        #33:#32        //  staticFunction:()V
  #38 = NameAndType        #31:#32        //  "<init>":()V
  #39 = Class              #58            //  java/lang/System
  #40 = NameAndType        #59:#60        //  out:Ljava/io/PrintStream;
  #41 = Utf8               2
  #42 = Class              #61            //  java/io/PrintStream
  #43 = NameAndType        #62:#63        //  println:(Ljava/lang/String;)V
  #44 = NameAndType        #24:#25        //  a:I
  #45 = Utf8               3
  #46 = Utf8               java/lang/StringBuilder
  #47 = Utf8               a=
  #48 = NameAndType        #64:#65        //  append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
  #49 = NameAndType        #64:#66        //  append:(I)Ljava/lang/StringBuilder;
  #50 = Utf8               ,b=
  #51 = NameAndType        #26:#25        //  b:I
  #52 = NameAndType        #67:#68        //  toString:()Ljava/lang/String;
  #53 = Utf8               4
  #54 = Utf8               com/zyl/effective/demo/other/ObjectStaticInitTest
  #55 = NameAndType        #22:#23        //  os:Lcom/zyl/effective/demo/other/ObjectStaticInitTest;
  #56 = Utf8               1
  #57 = Utf8               java/lang/Object
  #58 = Utf8               java/lang/System
  #59 = Utf8               out
  #60 = Utf8               Ljava/io/PrintStream;
  #61 = Utf8               java/io/PrintStream
  #62 = Utf8               println
  #63 = Utf8               (Ljava/lang/String;)V
  #64 = Utf8               append
  #65 = Utf8               (Ljava/lang/String;)Ljava/lang/StringBuilder;
  #66 = Utf8               (I)Ljava/lang/StringBuilder;
  #67 = Utf8               toString
  #68 = Utf8               ()Ljava/lang/String;
{
  static com.zyl.effective.demo.other.ObjectStaticInitTest os;
    Signature: Lcom/zyl/effective/demo/other/ObjectStaticInitTest;
    flags: ACC_STATIC



  int a;
    Signature: I
    flags: 



  static int b;
    Signature: I
    flags: ACC_STATIC



  public static void main(java.lang.String[]);
    Signature: ([Ljava/lang/String;)V
    flags: ACC_PUBLIC, ACC_STATIC

    LineNumberTable:
      line 6: 0
      line 7: 3
    Code:
      stack=0, locals=1, args_size=1
         0: invokestatic  #1                  // Method staticFunction:()V
         3: return        
      LineNumberTable:
        line 6: 0
        line 7: 3

  com.zyl.effective.demo.other.ObjectStaticInitTest();
    Signature: ()V
    flags: 

    LineNumberTable:
      line 19: 0
      line 16: 4
      line 29: 12
      line 20: 18
      line 21: 26
      line 22: 65
    Code:
      stack=3, locals=1, args_size=1
         0: aload_0       
         1: invokespecial #2                  // Method java/lang/Object."<init>":()V
         4: getstatic     #3                  // Field java/lang/System.out:Ljava/io/PrintStream;
         7: ldc           #4                  // String 2
         9: invokevirtual #5                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
        12: aload_0       
        13: bipush        110
        15: putfield      #6                  // Field a:I
        18: getstatic     #3                  // Field java/lang/System.out:Ljava/io/PrintStream;
        21: ldc           #7                  // String 3
        23: invokevirtual #5                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
        26: getstatic     #3                  // Field java/lang/System.out:Ljava/io/PrintStream;
        29: new           #8                  // class java/lang/StringBuilder
        32: dup           
        33: invokespecial #9                  // Method java/lang/StringBuilder."<init>":()V
        36: ldc           #10                 // String a=
        38: invokevirtual #11                 // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
        41: aload_0       
        42: getfield      #6                  // Field a:I
        45: invokevirtual #12                 // Method java/lang/StringBuilder.append:(I)Ljava/lang/StringBuilder;
        48: ldc           #13                 // String ,b=
        50: invokevirtual #11                 // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
        53: getstatic     #14                 // Field b:I
        56: invokevirtual #12                 // Method java/lang/StringBuilder.append:(I)Ljava/lang/StringBuilder;
        59: invokevirtual #15                 // Method java/lang/StringBuilder.toString:()Ljava/lang/String;
        62: invokevirtual #5                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
        65: return        
      LineNumberTable:
        line 19: 0
        line 16: 4
        line 29: 12
        line 20: 18
        line 21: 26
        line 22: 65

  public static void staticFunction();
    Signature: ()V
    flags: ACC_PUBLIC, ACC_STATIC

    LineNumberTable:
      line 25: 0
      line 26: 8
    Code:
      stack=2, locals=0, args_size=0
         0: getstatic     #3                  // Field java/lang/System.out:Ljava/io/PrintStream;
         3: ldc           #16                 // String 4
         5: invokevirtual #5                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
         8: return        
      LineNumberTable:
        line 25: 0
        line 26: 8

  static {};
    Signature: ()V
    flags: ACC_STATIC

    LineNumberTable:
      line 9: 0
      line 12: 10
      line 30: 18
    Code:
      stack=2, locals=0, args_size=0
         0: new           #17                 // class com/zyl/effective/demo/other/ObjectStaticInitTest
         3: dup           
         4: invokespecial #18                 // Method "<init>":()V
         7: putstatic     #19                 // Field os:Lcom/zyl/effective/demo/other/ObjectStaticInitTest;
        10: getstatic     #3                  // Field java/lang/System.out:Ljava/io/PrintStream;
        13: ldc           #20                 // String 1
        15: invokevirtual #5                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
        18: bipush        112
        20: putstatic     #14                 // Field b:I
        23: return        
      LineNumberTable:
        line 9: 0
        line 12: 10
        line 30: 18
}

```



