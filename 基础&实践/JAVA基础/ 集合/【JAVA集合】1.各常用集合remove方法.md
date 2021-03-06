- 背景
在研究Java NIO及Selector时：
```language
	Set<SelectionKey> selectionKeys = selector.keys();
	Iterator<SelectionKey> iteratorKeys = selectionKeys.iterator();
	while(iteratorKeys.hasNext()){
		SelectionKey selectionKey = iteratorKeys.next();
		iteratorKeys
		......
	}
```

```language
#### 示例1：
public class CollectionRemoveTest {

	public static void main(String[] args) {
		ArrayList list = new ArrayList<String>(); //ArrayList 底层实现为 数组
		list.add(String.valueOf("1"));
		list.add(String.valueOf("2"));
		
		list.remove(0); //result :OK
		Iterator listIterator = list.iterator();
		while(listIterator.hasNext()){
			listIterator.next();
			listIterator.remove(); //result: 不加上一行的next代码报错java.lang.IllegalStateException
		}
		System.out.println("ArrayList end.............");
		
		LinkedList<String> linkList= new LinkedList<String>(); //LinkedList 底层实现为 单链表
		linkList.add(String.valueOf("1"));
		linkList.add(String.valueOf("2"));
		linkList.remove();
		Iterator linkListIterator = list.iterator();
		while(linkListIterator.hasNext()){
			linkListIterator.next();
			linkListIterator.remove(); //result:不加上一行的next代码报错java.lang.IllegalStateException
		}
		System.out.println("linkList end.............");
		
		Set<String> set= new HashSet<String>(); //HashSet 底层实现为hashmap
		set.add(String.valueOf("1"));
		set.add(String.valueOf("2"));
		set.remove(new String("1"));//set.remove("1");  两种写法都可 remove 成功
		Iterator setIterator = set.iterator();
		while(setIterator.hasNext()){
			setIterator.next();
			setIterator.remove(); //result:不加上一行的next代码报错java.lang.IllegalStateException
			break;
		}
		System.out.println("set end............." + set.size());
	}

}
```
- Iterator 支持从源集合中安全地删除对象，只需在 Iterator 上调用 remove() 即可。这样做的好处是可以避免 ConcurrentModifiedException ，这个异常顾名思意：当打开 Iterator 迭代集合时，同时又在对集合进行修改。有些集合不允许在迭代时删除或添加元素，但是调用 Iterator 的remove() 方法是个安全的做法。
但如果多线程同时并发调用List的add和remove方法，则会抛出ConcurrentModifiedException，因为如果该集合迭代器就不再合法（即并发调用list的add和remove会异常，但可并发调用list的add和Iterator的remove方法）
- 循环中用下标删除多个元素的时候，它并不会正常的生效，如下示例：
```language
ArrayList<String> list = new ArrayList<String>(Arrays.asList("a","b","c","d"));
for(int i=0;i<list.size();i++){
    list.remove(i);
}
System.out.println(list); //结果：[b,d]
```
- 另外有两个ArrayList：java.util.ArrayList、java.util.Arrays.ArrayList，有差异需注意。
