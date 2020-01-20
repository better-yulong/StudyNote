- 参考资料：Lambda表达式详解https://www.cnblogs.com/haixiang/p/11029639.html
- Lambda 表达式是 JDK8 的一个新特性，可以取代大部分的匿名内部类，写出更优雅的 Java 代码，尤其在集合的遍历和其他集合操作中，可以极大地优化代码结构。

### 简单示例
#### 接口
```language
public interface DemoInterf1 {
	
	String objBuilder(String a,int b);

}

```
#### Lamdba简单示例
```language
public class Test{
    public boolean someLibraryMethod() {
        return true;
    }
    public static void main(String[] args) {
    	DemoInterf1 demoInterf1 = (String c,int b) -> {
    		System.out.println("NoReturnNoParam");
    		return "abc";
    	};
    	
    	System.out.println(demoInterf1.objBuilder("a",1));
    	
    	
    	DemoInterf1 demoInterf2 = (c,b) -> {
    		System.out.println("NoReturnNoParam");
    		return "abc";
    	};
    	
    	System.out.println(demoInterf2.objBuilder("?",1));
	}
}
```
详细可查阅页首参考资料