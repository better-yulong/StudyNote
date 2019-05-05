1. 模块：模块要尽可能小，但又不能太小（本书的模块：是指任何可重用的软件组件，从单个方法到包含多个包的复杂系统，均可是一个模块）。代码应该被重用，而不是被拷贝。模块之间的依赖应可能地降到最小。错误应该迟早被检测出来，最好是在编译时间。
2. 术语1：Java语言支持四种类型：接口(Interface)、类（class）、数组（array）和基本类型（premitive）。前三种通常称为引用类型（reference type），类实例和数组是对象（object），而基本类型的值则不是对象。类的成员（member）由它的域(field)、方法(method)、成员类(member class)和成员接口（member interface）组成。方法的签名（signature）由它的名称和所有参数类型组成；签名不包括它的返回类型。
3. 术语2（仅适用于本书，便于理解）：继承，同子类化，可理解为类的继承或实现。API（应用程序接口）是指类、接口、构造器、成员和序列化形式，并非等同于Interface；类、接口、构造器、成员和序列化形式被统称为API元素。
4. 本书主要基于约70条规则，会相互引用但可独立阅读。
5. 第三版在线阅读：
  - https://jiapengcai.gitbooks.io/effective-java/content/
  - https://sjsdfg.github.io/effective-java-3rd-chinese/#/README