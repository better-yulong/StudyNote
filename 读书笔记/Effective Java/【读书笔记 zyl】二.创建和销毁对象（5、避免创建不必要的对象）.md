一般来说，最好能重用对象而不是每次需要时都创建一个相同功能的新对象。重用方式即快速，又流行。若对象是不可变的（immutable），则其始终可被重用。
```language
	//
	String s = new String("string temp"); //DON'T DO THIS
```
