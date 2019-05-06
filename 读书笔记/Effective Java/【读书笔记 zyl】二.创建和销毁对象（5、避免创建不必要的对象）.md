一般来说，最好能重用对象而不是每次需要时都创建一个相同功能的新对象。重用方式即快速，又流行。若对象是不可变的（immutable），则其始终可被重用。
```language
	//反面示例
	String s = new String("string temp"); //DON'T DO THIS
```
该语句会使得每次执行时都创建一个新的String实例，但是这些创建动作全都是不必要。传递给String构造器的参数（"string temp"）本身就是一个String实例，功能方面等同于构造器所创建的