- 参考资料：Lambda表达式详解https://www.cnblogs.com/haixiang/p/11029639.html
- Lambda 表达式是 JDK8 的一个新特性，可以取代大部分的匿名内部类，写出更优雅的 Java 代码，尤其在集合的遍历和其他集合操作中，可以极大地优化代码结构。

### 简单示例
#### 接口
```language
public interface DemoInterf1 {
	
	String objBuilder(String a,int b);

}

```
#### Lamdba
