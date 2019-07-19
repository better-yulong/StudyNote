- 可参考资料：https://www.guitu18.com/se/java/2019-02-22/29.html、https://docs.oracle.com/javase/7/docs/api/java/lang/ClassLoader.html、https://blog.csdn.net/gzuimis/article/details/44198737

- Spring加载resource时classpath*:与classpath:的区别：https://www.jianshu.com/p/5bab9e03ab92
```language
package com.test.jvm.classloader;

import java.io.IOException;

//当前工程需引入spring-beans 的jar包，同时当前类依赖  org.springframework.beans。PropertyValue 类验证
//主要验证当jar与当前工程出现完全同名类，不同的方法调用结果有什么差异

//javac -encoding UTF-8 org/springframework/beans/PropertyValue.java
//study-code-effective-java\JVMDemo\src>javac -encoding UTF-8 com/test/jvm/classloader/ClassLoader1Test.java
//java -verbose:class com/test/jvm/classloader/ClassLoader1Test

public class ClassLoader1Test {

	public static void main(String[] args) throws ClassNotFoundException, IOException {
		System.out.println("A:" + ClassLoader1Test.class.getResource("org/springframework/beans/PropertyValue.class")); 
		System.out.println("B:" + ClassLoader1Test.class.getClassLoader().getResource("org/springframework/beans/PropertyValue.class")); 
		System.out.println("......................");
		System.out.println("C:" + ClassLoader1Test.class.getClassLoader().loadClass("org.springframework.beans.PropertyValue")); 
		
/*		[Loaded sun.misc.IOUtils from C:\Program Files\Java\jre1.8.0_112\lib\rt.jar]
				A:null
				B:file:/D:/work/workspace/study-code-effective-java/JVMDemo/src/org/springframework/beans/PropertyValue.class
				......................
				[Loaded org.springframework.beans.PropertyValue from file:/D:/work/workspace/study-code-effective-java/JVMDemo/src/]
				C:class org.springframework.beans.PropertyValue
				[Loaded java.lang.Shutdown from C:\Program Files\Java\jre1.8.0_112\lib\rt.jar]
				[Loaded java.lang.Shutdown$Lock from C:\Program Files\Java\jre1.8.0_112\lib\rt.jar]*/
		
		//通过验证可发现getResource并不会调用clssloader load class
		
		
		System.out.println("......................");
		System.out.println("A:" + ClassLoader1Test.class.getResource("")); 
		System.out.println("A:" + ClassLoader1Test.class.getResource("/")); 
		System.out.println("B:" + ClassLoader1Test.class.getClassLoader().getResource("")); 
		System.out.println("B:" + ClassLoader1Test.class.getClassLoader().getResource("/")); 
/*		......................
		A:file:/D:/work/workspace/study-code-effective-java/JVMDemo/src/com/test/jvm/classloader/
		A:file:/D:/work/workspace/study-code-effective-java/JVMDemo/src/
		B:file:/D:/work/workspace/study-code-effective-java/JVMDemo/src/
		B:null*/
	}
}

```
