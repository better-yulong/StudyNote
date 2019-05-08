Object所有的非final方法(equals、hashCode、toString、clone和finalize)都有明确的通用约定，因为他们被设计成是可覆盖的；故覆盖这些方法有责任遵守约定。

### 第8条：覆盖equals时请遵守通用约定
- equals方法看似简单但容易出错且后果严重，尽量避免覆盖equqls方法。
1. 类的每个实例本质上都是唯一的。对于代表活动实体而不是值(value）的类来说确实如此，如Thread。Object类提供的默认equals实现对于这些类来说正是正确的行为，故不应该覆盖。
2. 不关心类是否提供了"逻辑相等"(logical equality)的测试功能。例如，java.util.regex.Pattern 可以覆盖 equals 来检查两个 Pattern 实例是否表示完全相同的正则表达式，但设计人员认为客户端不需要或不需要这个功能。在这种情况下，从 Object 继承的 equals 实现是理想的。其实也就是说设计人员认为某些类不应该存在逻辑相等判断时，也就不会特意去重写equals方法，而是使用Object继承的equals.
3. 超类已经覆盖了equals，从超类继承过来的行为对子类也是合适的。如大多数Set实现都是AbstraceSet继承Set实现，List实现从AbstractList继承equals实现，Map实现从AbstractMap继承equals实现。另外发现个新知识点，Set、List、Map这些继承了equals方法，而其实实现并非与默认理解是根据当前对象hashcode来比较，这几个均是比较内部存储的对象是否相对。
```language
    AbstractList代码示例：
    public boolean equals(Object o) {
        if (o == this)
            return true;
        if (!(o instanceof List))
            return false;

        ListIterator<E> e1 = listIterator();
        ListIterator e2 = ((List) o).listIterator();
        while (e1.hasNext() && e2.hasNext()) {
            E o1 = e1.next();
            Object o2 = e2.next();
            if (!(o1==null ? o2==null : o1.equals(o2)))
                return false;
        }
        return !(e1.hasNext() || e2.hasNext());
    }
```
验证示例：
```language
public class EqualsTest {

	public static void main(String[] args) {
		Object o = new Object();
		ArrayList<Object> a1 = new ArrayList<Object>();
		a1.add(o);
		ArrayList<Object> a2 = new ArrayList<Object>();
		a2.add(o);
		System.out.println(a1.equals(a2)); // 结果为:true
		
		
		Object o3 = new Object();
		ArrayList<Object> a3 = new ArrayList<Object>();
		a3.add(o3);
		Object o4 = new Object();
		ArrayList<Object> a4 = new ArrayList<Object>();
		a4.add(o4);
		System.out.println(a3.equals(a4));// 结果为:false
	}

}

```
4. 当你确定类是私有的或包私有的,且可确定它的equals方法不被调用。
