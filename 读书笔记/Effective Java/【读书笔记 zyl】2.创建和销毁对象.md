### 第一条：考虑用静态工厂方法代替构造器
对于类而言，为了获取它自身的一个实现，最常用的方法是提供公有的构造器；但还有一种方法，则是类可提供一个公有的静态工厂方法（static factory method），它只是一个返回类的实例的静态方法。如：
```language
    public static final Boolean TRUE = new Boolean(true);
	
    public static Boolean valueOf(boolean b) {
        return (b ? TRUE : FALSE);
    }
```
