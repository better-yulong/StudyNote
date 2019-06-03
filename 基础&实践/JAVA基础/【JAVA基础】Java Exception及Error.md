### 一. Java标准库内置了一些通用的异常类，这些类以Throwable为顶层父类，Throwable又派生出Error类和Exception类。
1. 错误： Error类以及它的子类的实例，代表了JVM本身的错误。错误不能被程序员通过代码处理，Error很少出现。
2. 异常： Exception以及它的子类，代表程序运行时发送的各种不期望发生的事件。可以被Java异常处理机制使用，是异常处理的核心。

### 二. 异常Exception
- Exception又分为运行时异常（Runtime Exception）和受检查的异常(Checked Exception )，但所有的的Excepiton都继承自Exception类，RuntimeException类只是Exception的其中一个直接子类。
1. RuntimeException：其特点是Java编译器不去检查它，也就是说，当程序中可能出现这类异常时，即使没有用try……catch捕获，也没有用throws抛出，还是会编译通过，如除数为零的ArithmeticException、错误的类型转换、数组越界访问和试图访问空指针等。处理RuntimeException的原则是：如果出现RuntimeException，那么一定是程序员的错误。
- 运行时异常：继承于 Runtime Exception ，Java 编译器允许程序不对它们做出处理（其实直接理解为有IDE如Eclipse编码时未显示try catch会直接提示错误；而编译时也会直接编译失败），下面列出了主要的运行时异常：
   1. ArithmeticException ： 一个非法算术运算产生的异常。
   2.  ArrayStoreException ： 存入数组的内容数据类型不一致所产生的异常。
   3. ArrayIndexOutOfBoundsException ： 数组索引超出范围所产生的异常。
   4. ClassCastException ： 类对象强迫转换造成不当类对象所产生的异常。
   5. IllegalArgumentException ： 程序调用时，返回错误自变量的数据类型。
   6. IllegalThreadStateException ： 线程在不合理状态下运行所产生的异常。
   7. NumberFormatException ： 字符串转换为数值所产生的异常。
   8. IllegalMonitorStateException ： 线程等候或通知对象时所产生的异常。
   9. IndexOutOfBoundsException ： 索引超出范围所产生的异常。
   10. NegativeException ： 数组建立负值索引所产生的异常。
   11. NullPointerException ： 对象引用参考值为 null所产生的异常。
   12. SecurityException ： 违反安全所产生的异常。
2. 受检查的异常（IOException等）：这类异常如果没有try……catch也没有throws抛出，编译是通不过的。这类异常一般是外部错误，例如文件找不到、试图从文件尾后读取数据等，这并不是程序本身的错误，而是在应用环境中出现的外部错误。
Exception的直接子类，除RuntimeExcepiton子类外，其他全部为受检查的异常，它们都在 java.lang 类库内定义；Java 编译器要求程序必须捕获或者声明抛弃这种异常（其实直接理解为有IDE如Eclipse编码时未显示try catch不会直接提示错误，编译可正常通过；即可显示try catch 也可不显示 try catch），下面列出了主要的检查异常：
   1. ClassNotFoundException ： 找不到类或接口所产生的异常。
   2. CloneNotSupportedException ： 使用对象的 clone( )方法但无法执行 Cloneable所产生的异常。
   3.  IllegalAccessException ： 类定义不明确所产生的异常。
   4. InstantiationException ： 使用 newInstance( )方法试图建立一个类 instance时所产生的异常。
   5.  InterruptedException ： 目前线程等待执行，另一线程中断目前线程所产生的异常。

### 三. 错误Error
Error 类与异常一样，它们都是继承自 java.lang.Throwable 类。 Error 类对象由 Java 虚拟机生成并抛出。 Error 类包括 LinkageError （结合错误）与 VitualmanchineError （虚拟机错误）两种子类。
#### LinkageError 类
LinkageError 类的子类表示一个类信赖于另一个类，但是，在前一个类编译之后，后一个类的改变会与它不兼容。常见的有 NoSuchMethodError
#### VitualmachineError类
当 Java 虚拟机崩溃了或用尽了它继续操作所需的资源时，抛出该错误：
> VitualmachineError 包含 InternalError 、 OutOfMemoryError 、 StackOverflow Error 和 UnknownError 。这些类所代表的意义如下所述。

### 四. 个人理解
1. 并非以Error结尾的都是Error错误，很多Exception类仍是以及Error结尾
2. 