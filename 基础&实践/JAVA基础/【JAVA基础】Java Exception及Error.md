### 一. Java标准库内置了一些通用的异常类，这些类以Throwable为顶层父类，Throwable又派生出Error类和Exception类。
1. 错误： Error类以及它的子类的实例，代表了JVM本身的错误。错误不能被程序员通过代码处理，Error很少出现。
2. 异常： Exception以及它的子类，代表程序运行时发送的各种不期望发生的事件。可以被Java异常处理机制使用，是异常处理的核心。

### 二. 异常Exception
Exception可能可通过编码解决；而Error极少数可通过编码解决，如内存泄漏、递归过深导致栈溢出）
Exception又分为运行时异常（Runtime Exception）和受检查的异常(Checked Exception )。
RuntimeException：其特点是Java编译器不去检查它，也就是说，当程序中可能出现这类异常时，即使没有用try……catch捕获，也没有用throws抛出，还是会编译通过，如除数为零的ArithmeticException、错误的类型转换、数组越界访问和试图访问空指针等。处理RuntimeException的原则是：如果出现RuntimeException，那么一定是程序员的错误。

受检查的异常（IOException等）：这类异常如果没有try……catch也没有throws抛出，编译是通不过的。这类异常一般是外部错误，例如文件找不到、试图从文件尾后读取数据等，这并不是程序本身的错误，而是在应用环境中出现的外部错误。