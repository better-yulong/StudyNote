- 静态工厂与构造器有共同的局限：不能很好的扩展到大量的可选参数。
1. 如构造器对应4个必选参数，至多20个多组的非必选参数。那么习惯会采用重叠构造器（telescoping constructor），即先提供一个只有必要参数的构造器，之后提供多个包含可选参数的构造器，最后提供一个包含所有参数的构造器。而使用此种方式会导致通常需给很多本不想设置的参数传递值，且参数数目增加很容易失去控制。即：重叠构造器模式可行，但当参数过多，客户端代码会很难编写，且难以阅读。
2. 那么针对该情况，有种替代方法，即JavaBeans模式：无参构造器创建对象，然后调用setter()方法来设置必要参数及可选参数。但是该模式亦有严重缺点：构造过程被拆分成多次调用，使得构造过程中JavaBean可能处于不一致状态；类无法通过参数检验构造器参数的有效性来保证一致性。同时JavaBeans模式则使得类不能为不可变，需要额外考虑其线程安全。
3. 而第三种替换方法，即能保证像重叠构造器模式的安全性，也可保证JavaBeans模式好的可读性，即Builder模式的一种形式。即让客户端利用所有必要的参数调用构造器（或静态工厂），得到builder对象；之后客户端通过builder对象上调用类似于setter方法来设置相关可选参数。
 - 如用一个类表示包含食品外的营养成分标签，其中有必须域：每份含量、每罐含量、每份卡路里，还有超过20个可选域：总脂肪量、饱和脂肪量、转化脂肪、胆固醇、钠等。如若采用重叠构造器或JavaBeans模式均不太好编写。
 ```language

public class NutritionFacts {

	private final int servingSize ;
	private final int servings ;
	private final int calories;
	private final int fat ;
	private final int sodium ;
	private final int carbohydrate ;
	
	public static class Builder{
		//required paremeters
		private final int servingSize ;
		private final int servings ;
		
		//Optional paremeters
		private int calories = 0 ;
		private int fat = 0 ;
		private int sodium = 0 ;
		private int carbohydrate = 0 ;
		
		public Builder(int servingSize,int servings){
			this.servingSize = servingSize ;
			this.servings = servings;
		}
		
		public Builder calories(int val){
			this.calories = val;
			return this;
		}
		
		public Builder fat(int val){
			this.fat = val;
			return this;
		}
		
		public Builder sodium(int val){
			this.sodium = val;
			return this;
		}
		public Builder carbohydrate(int val){
			this.carbohydrate = val;
			return this;
		}
		
		public NutritionFacts build(){
			//optional: add paremeter check --- 风险
			return new NutritionFacts(this);
		}
	}
	
	private NutritionFacts(Builder builder){
		//optional: add paremeter check --- 风险
		this.servingSize = builder.servingSize ;
		this.servings = builder.servings ;
		this.calories = builder.calories ;
		this.fat = builder.fat ;
		this.sodium = builder.sodium ;
		this.carbohydrate = builder.carbohydrate ;
		//optional: add paremeter check --- 正确（参考第39条：Time-Of-Check）

	}
	
	public static void main(String[] args){
		NutritionFacts cacaCola = new NutritionFacts.Builder(240, 8).calories(100).sodium(20).carbohydrate(10).build();
		System.out.println(cacaCola.servingSize);
		System.out.println(cacaCola.sodium);
		System.out.println(cacaCola.fat);
	}
}

/***
 * Builder构建器模式，区别于重叠构造器、JavaBeans模式，可保证线程全亦可有较好的可读性.
 * 
 * 运行结果：
 *  240
	20
	0

 * */

```
- builder类似于构造器，可对参数强加约束条件，build方法可检验这些约束条件。将参数从builder拷贝到对象中之后，并在对象域而不是builder域对它们进行检验。如上注释三个地方"optional: add paremeter check"，但