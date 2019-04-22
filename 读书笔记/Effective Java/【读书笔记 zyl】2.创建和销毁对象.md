### 第一条：考虑用静态工厂方法代替构造器
对于类而言，为了获取它自身的一个实现，最常用的方法是提供公有的构造器；但还有一种方法，则是类可提供一个公有的静态工厂方法（static factory method），它只是一个返回类的实例的静态方法。如：
```language
    public static final Boolean TRUE = new Boolean(true);
    public static final Boolean FALSE = new Boolean(false);
    public static Boolean valueOf(boolean b) {
        return (b ? TRUE : FALSE);
    }
```
> 注意：静态工厂方法与设计模式中的工厂方法模式不同，并不直接对应。
- 类可通过静态工厂方法提供给外部使用而不是通过构造器，优势如下：
- 1. 静态工厂方法有名称：一个类只能有一个带有自指定签名的构造器。虽然可通过重载方式，即调整参数顺序、个数等来避开限制。但这并非好主意，因为用户永远也记不住该用哪个构造器，往往可能调用错误（类似情况还真遇到过，同名类同方法但包不同，而参数顺序均为String但顺序相反；踩了个坑...）。故当一个类需要多个带有相同签名的构造器时，可用静态方法代替构造器，并且需慎重运行名称以便
