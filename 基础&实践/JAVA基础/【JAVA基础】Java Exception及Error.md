一Java标准库内置了一些通用的异常类，这些类以Throwable为顶层父类，Throwable又派生出Error类和Exception类。
1. 错误： Error类以及它的子类的实例，代表了JVM本身的错误。错误不能被程序员通过代码处理，Error很少出现。
2. 异常： Exception以及它的子类，代表程序运行时发送的各种不期望发生的事件。可以被Java异常处理机制使用，是异常处理的核心。


Exception可能可通过编码解决；而Error极少数可通过编码解决，如内存泄漏、递归过深导致栈溢出）