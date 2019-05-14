类的接口是Java程序设计语言的核心，它们也是Java语言抽象的基本单元。

### 第13条：使类和成员的可访问性最小化
- 设计良好的模块会隐藏所有的实现细节，把它的API与它的实现清晰隔离开。模块之前只通过它们的API通知，一个模块不需要关注其他模块的内部工作。这个概念称为"信息隐藏"或封装，是软件设计的基本原则之一。
- 封装之所以重要，是因为：它可以有效的解除组成系统的各模块之间的耦合关系，使得模块可独立的开发、测试、优化、使用、理解和修改；可提供软件的可重用性，降低系统风险。
1. 第一规则：尽可能使每个类或者成员不被外界访问，即尽可能最小的访问级别。
对于顶层（非嵌套的）的类和接口，保有两个可能的访问级别:包级私有(package-private)和公共的(public）（外部类不能用private修饰，不然直接就报错，private只能修饰内部类）。----(顶层类如何理解？即是当前java文件名对应的类；其实在eclipse中通过界面新建class、interface 时会发现可选的访问修饰符只有public和package(即默认default)，对于protected、private是无法选中的；也就是class、interface说要么public要么啥也不用即默认包级私有；注意包级私有类在其子package包中也无法访问)
```language
//顶层（非嵌套的）的类，只允许包级私有(package-private)和公共的(public）
public class AccessTest1 { 
	
	 //内部类
	 //can private/protected/public or default(package), if private ,just be used in this file.
	 static class InnerClass{ 
		public void a(){
		}
	}

}

//can't private/protected/public,only default(package)--只允许默认包级私有
class PackageClass{ 
	
}

```
```language
public class AccessTest2 {
	
	public  class InnerClass{
		public void a(){
			// 1. InnerClass must be static .else： if not No enclosing instance of type AccessTest1 is accessible.
			//即只有内部类为static，才可使用外部类AccessTest1.直接访问，类似于静态变量
			// 2. AccessTest1.InnerClass 必须为public或者默认或者明确标明protected，若其为private此处无法访问。
			//private 修改符修饰类时，只能作用于内部内
			AccessTest1.InnerClass a = new AccessTest1.InnerClass(); 
			
			PackageClass pc = new PackageClass();
		}
	}

}

```
- 类修饰为包级私有，则说明其仅作为内部实现一部分，而不是导出API，可在后续版本中修改、替换甚至消除，且不必担心影响现有的客户端；但若类定义为公开public，则就有义务永远支持并保持兼容性。
- 如果一个包级私有顶级类或接口只被一个类使用，那么可考虑这个类作为使用它的唯一类的私有静态嵌套类（或者即私有静态内部类）。但是，减少不必要的公共类的可访问性要比包级私有的顶级类更重要：公共类是包的API的一部分，而包级私有的顶级类是该包实现的一部分。
- 对于成员（属性、

