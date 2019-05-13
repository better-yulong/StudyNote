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
4. 当你确定类是私有的或包私有的,且可确定它的equals方法不被调用时，为防止意外调用产生风险，无疑是应该覆盖equals方法。
```language
@Override
public boolean equals(Object o){
	throw new AssertionError();// Method is nerver called
}
```
- 那么，什么时候应该覆盖Object.equals呢？如果类具有自己特有的"逻辑相等"概念（不等同于对象相等的概念），而且超类还没有覆盖equals以实现预期的行为（如上面说的AbstractList、AbstractMap、AbstractSet类），此时我们就需要覆盖equals方法，这通常属于"值类（value class）"的情形。值类仅仅是一个表示值的类，例如Integer或者Date（个人理解大部分集合类也是）。即程序员利用equals方法比较对象时，希望知道它们在逻辑上是否相等（内部存储的数据）而不是想知道它们是否指向同一对象。
- 有一种"值类"不需要覆盖equals方法，即用实例受控确保"每个值至多只存在一个对象"的类，枚举类型即属于这种类。对于这种类而言，逻辑相同与对象等同是一回事，因此Object的equals方法等同于逻辑意义上的equals方法。怎么理解呢？可以理解为安全的单例场景（注意是完全，即绝对不会发生单例破坏），通过静态工厂方法或枚举返回的始终是同一个对象，即无需担忧同一个类会存在多个实例对象，所以无需覆盖equals方法。
#### 覆盖equals方法时必须遵守它的通用约定
- equals方法实现了等价关系(equivalence relation):
1. 自反性(reflexive):对于任何非null的引用值x，x.equals（x）必须返回为true
2. 对称性(symmetric)：对于任何非null的引用值x和y，x.equals(y)与y.equals(x）的结果必须始终保持一致，同时为true或同时为false。
3. 传递性(transitive):对于任何非null的引用值x、y和z，如果x.equals(y)为true且y.equals(z)也为true,那么x.equals(z)也必须为true。
4. 一致性(consistent):对于任何非null的引用值x和y，只要equals的比较操作在对象中所用的信息没有被修改，多次调用x.equals(y)结果必须返回一致，即多次全部为true或者多次全部为false.
5. 对于任何非null的引用x，x.equals(null)必须返回false。
- 涉及继承非抽象类时，若子类增加值组件，父子类的实例或同一父类不同的实例，若在使用equals方法可能会违背里式替换原则，则可能会违背如上通用约定。而为解决该问题，则可考虑建议：复合优先于继承。但实际中确实有部分类违反如对称性，例如java.sql.Timestamp对java.util.Date进行了扩展，并增加了nanoseconds域，Timestamp的equals实际违反了对称性，如果Timestamp和Date对象被用于同一集合，或者以其他方式被混合，则可能引起不正确的行为。Timestamp为在有一个免责声明，告诫不要混合使用，但实际此种实现并不推荐。
#### 实现高质量equals方法的诀窍:
1. 使用==操作符检查"参数是否为这个对象的引用"
2. 使用instanceof操作符检查"参数是否为为正确的类型"
3. 把参数转换成正确的类型
4. 对于该类中的每个"关键(significant)"域，检查参数中的域是否与该对象中对应的域相匹配。---小技巧：：域的比较顺序可能会影响到equals方法的性能；为了获得最佳性能，应该优先比较最有可能不一致的域或者是开销最低的域，最理想的情况是两个条件同时满足的域。
5. 不你编写完equals方法后，应该问自己三个问题：它是否是对称的，传递的，一致的（自反性、非空性也需满足，但一般会自动满足）？同时还得编写单元测试来检测这些特性。
#### 最后的告诫
- 覆盖equals方法时总要覆盖hashCode（见第9条）
- 不要企图让equals方法过于智能
- 不要将equals声明中的Object对象替换为其他的类型，因为equals方法默认参数类型是Object，如：
```language
	public boolean equals（MyClass o){
		...
	}
```

### 第9条：覆盖equals时总要覆盖hashCode
- 原则：每个覆盖了equals方法的类中也必覆盖hashCode方法。如若违反该规则，会使得该类无法结合所有基于散列的集合一起正常运行，如HashMap、HashSet和Hashtable.
- 同样hashCode覆盖也需新遵循规范（第二版翻译有些晦涩，此处采用第三版本内容）：
  1. 应用程序执行期间，只要对象的equals方法的比较操作所用到的信息没有被修改，那么对同一对象调用多次hashCode方法都必须始终如一返回同一个整数；但在同一程序的多次执行过程中，每次执行所返回整数可以不一致。
  2. 如果两个对象根据 equals(Object) 方法比较是相等的，那么在两个对象上调用 hashCode 就必须产生的结果是相同的整数。
  3. 如果两个对象根据 equals(Object) 方法比较并不相等，则不要求在每个对象上调用 hashCode 都必须产生不同的结果。 但是，程序员应该意识到，为不相等的对象生成不同的结果可能会提高散列表（hash tables）的性能。
 - 如若没有覆盖hashCode方法而违反的关键约定是第二条：相等的对象必须具有相等的散列码(hashCode)。根据类的equals方法，两个截然不同的实例在逻辑上有可能是相等的；但是根据Object类的hashCode方法，它们仅仅是两个没有任何共同之处的对象。
- 基于如上理角提到的逻辑相等，个人也想到==是否可认为物理相等。那==对于引用类型的比较一定是只有指向同一个对象才会返回true；而Object类的equals方法默认其实也是使用==判断，但是因为不同的类、场景需要，很多类都对equals&hashCode方法进行覆盖重写，使得其支持逻辑相等的判断。下面来看下String类的equals及hashCode方法源码，再结合简单的示例应该就比较好理解了。
```language
    public boolean equals(Object anObject) {
        if (this == anObject) {
            return true;
        }
        if (anObject instanceof String) {
            String anotherString = (String)anObject;
            int n = count;
            if (n == anotherString.count) {
                char v1[] = value;
                char v2[] = anotherString.value;
                int i = offset;
                int j = anotherString.offset;
                while (n-- != 0) {
                    if (v1[i++] != v2[j++])
                        return false;
                }
                return true;
            }
        }
        return false;
    }

    public int hashCode() {
        int h = hash;
        if (h == 0 && count > 0) {
            int off = offset;
            char val[] = value;
            int len = count;

            for (int i = 0; i < len; i++) {
                h = 31*h + val[off++];
            }
            hash = h;
        }
        return h;
    }

    /** Cache the hash code for the string */
    private int hash; // Default to 0
```
```language
	String a = new String("1");
	String b = new String("1");
	System.out.println(a==b); //false
	System.out.println(a.equals(b));//true
	System.out.println(a.hashCode());//49
	System.out.println(a.hashCode()==b.hashCode());//true
```
- 对于hashCode散列码除了遵循如上规则，但也需考虑避免hashCode避免每个对象具有相同的散列码。如HashMap中，若所有对象散列码均相同，则所有对象被映射到同一散列桶中使散列退化为链表，这样会导致本该线性时间运行的程序变成了以平方级时间在运行，规模越大影响越明显。
- 如若一个类不可变以且计算散列码开销较大可考虑把散列码缓存在对象内部，避免每次请求都重新计算散列码；如果对象大多数时候被用来做散列键(hash keys)，则应该创建例时就计算散列码。否则，则可选择"延迟初始化"散列化，直到首次调用hashCode时才初始化。
- 不要试图从散列码计算中排除一个对象的关键部分来提高性能。虽然可能导致散列码计算更快，但可能导致散列表缓慢不利于查找。

### 第10条：始终要覆盖toString
- 虽然java.lang.Object 提供了toString方法的一个实现，但它返回的字符串通常并不是类的用户所期望看到的:类名@散列码16进制。而toString的通用约定指出：被返回的字符串应该是一个简洁的，但信息丰富，并且易于阅读的表达形式，故建议所有的子类都覆盖这个方法。
- 遵守toString的约定并不像遵守equals和hashCode的约定那么重要，但是 提供好的toString实现可以使类用起来更加舒适。实际应用中，toString方法应该返回对象中包含的所有值得关注的信息，如若对象太大包含信息难以简单表达可考虑返回摘要信息。同时也存在问题，即可能toString的格式可能被用来作为解析使用，而后续格式的变化则可能破坏原有的代码和数据。故无论是否指定指定格式，均应该在文档中明确说明。

### 第11条：谨慎覆盖clone
Cloneable接口的目的是作为对象的一个mixin接口（mixin interface，后续会讲到），表明这样的对象允许克隆(clone)。但实际其并未达到该目的，其主要缺陷在缺少一个clone方法，Object的clone方法是受保护的（protected native Object clone()），如果不借助于反射就不仅仅因为实现了Cloneable就可调用clone方法。即使反射也可能会失败，因为无法保证该对象一定具有可访问的clone方法。虽然如此，但该方法仍一直被保留，所以也需值得进一步了解，以便知道如何实现一个行为良好的clone方法，并知道何时适合这样做，是否有可替换做法。
- Cloneable接口并没有包含任何方法，那么它的作用是什么呢？它决定了Object受保护的clone方法实现的行为：如果一个类实现了Cloneable，Object的clone方法就返回该对象的逐域拷贝，否则抛出CloneSupportedException，这是接口的一种极端非典型的用法。（感觉类似于java.io.Serializable接口，其也不包含任何方法）。
- 总的来说，其实关于clone方法相关的描述还是相对晦涩，可简单理解为：clone方法可理解为另一个构造器，必须确保它不会伤害到原始的对象，并确保正确的创建被克隆对象中的约束条件，即clone方法是禁止给原对象赋新值的。Clone架构与引用可变对象的final域的正常用法是不相兼容的，除非在原始对象和克隆对象之前可安全的共享此可变对象；或者说为了使类可克隆，可能有必要从某些域中去掉fianl修饰域。
1. 克隆如集合类的对象，常见的方法递归调用每个对象的clone方法，但如hashtable如果桶是空的则为null，出于性能考虑其实实际是重新创建buckets数组并遍历原始的bucket数组对第一个非空散列桶进行深度拷贝。可参考hasttable的clone方法源码：
```language
    public synchronized Object clone() {
        try {
            Hashtable<K,V> t = (Hashtable<K,V>) super.clone();
            t.table = new Entry[table.length];
            for (int i = table.length ; i-- > 0 ; ) {
                t.table[i] = (table[i] != null)
                    ? (Entry<K,V>) table[i].clone() : null;
            }
            t.keySet = null;
            t.entrySet = null;
            t.values = null;
            t.modCount = 0;
            return t;
        } catch (CloneNotSupportedException e) {
            // this shouldn't happen, since we are Cloneable
            throw new InternalError();
        }
    }

```
2、clone方法的使用需注意几点：a、避免在构造过程中调用新对象的非final方法，以避免新对象与原对象不一致；b、clone方法代价有可能比较高，需谨慎使用；3、Ojbect类的clone方法默认非线程安全，若希望线程安全的使用，需考虑编写同步的clone方法不调用super.clone（）
- 简而言之，所有实现了Cloneable接口的类都应该用一个公有的方法覆盖clone。但实际并不推荐去这么使用，其中一个实现对象拷贝的好办法是拷贝构造器或者拷贝工厂来代替继承Cloneable接口并实现clone方法。同时拷贝构造器或者拷贝工厂 可带参数，可允许客户选择拷贝的实现类型，而不是强近客户接受原始的实现类型。例如一个HashSet，希望将其拷贝成一个TreeSet; clone方法无法提供该功能，但是使用转换构造器很容易实现：new TreeSet(s)。
- 总的来说，强烈不建议通过clone方法来实现对象拷贝。
> 补充知识点：
1. 浅拷贝（浅克隆）复制出来的对象的所有变量都含有与原来的对象相同的值，而所有的对其他对象的引用仍然指向原来的对象。
2. 深拷贝（深克隆）复制出来的所有变量都含有与原来的对象相同的值，那些引用其他对象的变量将指向复制出来的新对象，而不再是原有的那些被引用的对象。换言之，深复制把要复制的对象所引用的对象都复制了一遍。

### 第12条：考虑实现Comparable接口
与之前的方法不同，compareTo方法并没有声明在Object，而Comparable接口唯一的方法。类实现了Comparable接口，就表明它的实例具有内存的排序关系(natural ordering)，compareTo方法不但允许进行简单的等同性比较，而且还允许执行顺序比较，同时其参数是泛型。
```language
public interface Comparable<T> {
    public int compareTo(T o);
}
```
而对于实现了Comparable接口的对象数组进行排序则很简单： Arrays.sort(a); Arrays类的sort方法源码：
```language
    public static void sort(Object[] a) {
        if (LegacyMergeSort.userRequested)
            legacyMergeSort(a); //legacyMergeSort后续可能被废弃
        else
            ComparableTimSort.sort(a);
    }
```
> java.util.ComparableTimSort中的sort()方法简单分析:TimSort 是一个归并排序做了大量优化的版本。对归并排序排在已经反向排好序的输入时表现O(n2)的特点做了特别优化。对已经正向排好序的输入减少回溯。对两种情况混合（一会升序，一会降序）的输入处理比较好。
> 在jdk1.7之后，Arrays类中的sort方法有一个分支判断，当LegacyMergeSort.userRequested为true的情况下，采用legacyMergeSort，否则采用ComparableTimSort。并且在legacyMergeSort的注释上标明了该方法会在以后的jdk版本中废弃，因此以后Arrays类中的sort方法将采用ComparableTimSort类中的sort方法。(https://blog.csdn.net/bruce_6/article/details/38299199)
- 归并排序后序研究算法时针对理解。
- 示例1：```language
public class WordList {

	public static void main(String[] args) {
		 	Set<String> s = new TreeSet<>();
	        Collections.addAll(s, new String[]{"a","e","c"});
	        System.out.println(s);

	}

}
/**
 * 结果：[a, c, e]
 * */

```
通过实现Comparable接口，可使类与所依赖该类的集合进行操作；几乎Java平台类的所有值类及所有枚举类型都实现了Comparable接口。如若正在编写的类有明显自然顺序(如字母顺序，数字顺序或时间顺序)的值类，则应该实现Comparable接口。
- compareTo 方法的通用约定与 equals 相似：将此对象与指定的对象按照排序进行比较。 返回值可能为负整数，零或正整数，因为此对象对应小于，等于或大于指定的对象。 如果指定对象的类型与此对象不能进行比较，则引发 ClassCastException 异常。
-  与 equals 方法不同，equals 方法在所有对象上施加了全局等价关系（并不要求对象类型相同，只认hashcode结果）；而compareTo 不必跨越不同类型的对象：当遇到不同类型的对象时，compareTo 被允许抛出 ClassCastException 异常。涉及相关类：排序后的集合 TreeSet 和 TreeMap 类，以及包含搜索和排序算法的实用程序类 Collections 和 Arrays。
- compareTo 约定的最后一段是一个强烈的建议而并非真正要求，只是声明 compareTo 方法施加的相等性测试，通常应该返回与 equals 方法相同的结果。（如BigDeimal类其compareTo方法与equals不一致）