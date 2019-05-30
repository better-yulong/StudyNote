https://blog.csdn.net/cws1214/article/details/52130688 
https://www.cnblogs.com/liuyun1995/p/8144706.html
https://www.liangzl.com/get-article-detail-129297.html
### 了解动态代理的前提是对JVM指令集及Class字节码文件结构有清晰的认识，不然很难去理解 

new关键字能调用任何构造函数。
Class.newInstance() 只能够调用无参的构造函数，即默认的构造函数； 
Constructor.newInstance() 可以根据传入的参数，调用任意构造构造函数。 

ew关键字是强类型的，效率相对较高。
newInstance()是弱类型的，效率相对较低。

- 既然使用newInstance()构造对象的地方通过new关键字也可以创建对象，为什么又会使用newInstance()来创建对象呢？
假设定义了一个接口Door，开始的时候是用木门的，定义为一个类WoodenDoor，在程序里就要这样写 Door door = new WoodenDoor() 。假设后来生活条件提高，换为自动门了，定义一个类AutoDoor，这时程序就要改写为 Door door = new AutoDoor() 。虽然只是改个标识符，如果这样的语句特别多，改动还是挺大的。于是出现了工厂模式，所有Door的实例都由DoorFactory提供，这时换一种门的时候，只需要把工厂的生产模式改一下，还是要改一点代码。
而如果使用newInstance()，则可以在不改变代码的情况下，换为另外一种Door。具体方法是把Door的具体实现类的类名放到配置文件中，通过newInstance()生成实例。这样，改变另外一种Door的时候，只改配置文件就可以了。示例代码如下：
String className = 从配置文件读取Door的具体实现类的类名; 
Door door = (Door) Class.forName(className).newInstance();
    再配合依赖注入的方法，就提高了软件的可伸缩性、可扩展性。




$Proxy0实现了Bird接口，当然可以类型转化
final class $Proxy0 extends Proxy implements Bird;
#### 结合Class文件结构和下面代码版本就可理解如上的继承、实现关系
结合class文件结构：
```language
ClassFile { 
    u4 magic; 
    u2 minor_version; 
    u2 major_version; 
    u2 constant_pool_count; 
    cp_info constant_pool[constant_pool_count-1]; 
    u2 access_flags; 
    u2 this_class; 
    u2 super_class; 
    u2 interfaces_count; 
    u2 interfaces[interfaces_count]; 
    u2 fields_count; 
    field_info fields[fields_count]; 
    u2 methods_count; 
    method_info methods[methods_count]; 
    u2 attributes_count; 
    attribute_info attributes[attributes_count]; 
}
```


```language
public static byte[] generateProxyClass(String paramString, Class[] paramArrayOfClass) {
		ProxyGenerator localProxyGenerator = new ProxyGenerator(paramString, paramArrayOfClass);
		byte[] arrayOfByte = localProxyGenerator.generateClassFile();
		if (saveGeneratedFiles)
			AccessController.doPrivileged(new PrivilegedAction(paramString, arrayOfByte) {
				public Object run() {
					try {
						FileOutputStream localFileOutputStream = new FileOutputStream(
								ProxyGenerator.access$000(this.val$name) + ".class");
						localFileOutputStream.write(this.val$classFile);
						localFileOutputStream.close();
						return null;
					} catch (IOException localIOException) {
						throw new InternalError("I/O exception saving generated file: " + localIOException);
					}
				}
			});
		return arrayOfByte;
	}
```

```language
	private ProxyGenerator(String paramString, Class[] paramArrayOfClass) {
		this.className = paramString;
		this.interfaces = paramArrayOfClass;
	}
```
```language
	private byte[] generateClassFile() {
		addProxyMethod(hashCodeMethod, Object.class);
		addProxyMethod(equalsMethod, Object.class);
		addProxyMethod(toStringMethod, Object.class);
		for (int i = 0; i < this.interfaces.length; ++i) {
			localObject1 = this.interfaces[i].getMethods();
			for (int k = 0; k < localObject1.length; ++k)
				addProxyMethod(localObject1[k], this.interfaces[i]);
		}
		Iterator localIterator1 = this.proxyMethods.values().iterator();
		while (localIterator1.hasNext()) {
			localObject1 = (List) localIterator1.next();
			checkReturnTypes((List) localObject1);
		}
		Object localObject2;
		try {
			this.methods.add(generateConstructor());
			localIterator1 = this.proxyMethods.values().iterator();
			while (localIterator1.hasNext()) {
				localObject1 = (List) localIterator1.next();
				Iterator localIterator2 = ((List) localObject1).iterator();
				while (localIterator2.hasNext()) {
					localObject2 = (ProxyMethod) localIterator2.next();
					this.fields.add(new FieldInfo(((ProxyMethod) localObject2).methodFieldName,
							"Ljava/lang/reflect/Method;", 10));
					this.methods.add(((ProxyMethod) localObject2).generateMethod());
				}
			}
			this.methods.add(generateStaticInitializer());
		} catch (IOException localIOException1) {
			throw new InternalError("unexpected I/O Exception");
		}
		if (this.methods.size() > 65535)
			throw new IllegalArgumentException("method limit exceeded");
		if (this.fields.size() > 65535)
			throw new IllegalArgumentException("field limit exceeded");
		this.cp.getClass(dotToSlash(this.className));
		this.cp.getClass("java/lang/reflect/Proxy");//对应字节码常量：super_class
		for (int j = 0; j < this.interfaces.length; ++j)
			this.cp.getClass(dotToSlash(this.interfaces[j].getName()));
		this.cp.setReadOnly();
		ByteArrayOutputStream localByteArrayOutputStream = new ByteArrayOutputStream();
		Object localObject1 = new DataOutputStream(localByteArrayOutputStream);
		try {
			((DataOutputStream) localObject1).writeInt(-889275714);
			((DataOutputStream) localObject1).writeShort(0);
			((DataOutputStream) localObject1).writeShort(49);
			this.cp.write((OutputStream) localObject1);
			((DataOutputStream) localObject1).writeShort(49);
			((DataOutputStream) localObject1).writeShort(this.cp.getClass(dotToSlash(this.className)));
			((DataOutputStream) localObject1).writeShort(this.cp.getClass("java/lang/reflect/Proxy"));
			((DataOutputStream) localObject1).writeShort(this.interfaces.length);
			for (int l = 0; l < this.interfaces.length; ++l)
				((DataOutputStream) localObject1)
						.writeShort(this.cp.getClass(dotToSlash(this.interfaces[l].getName())));
			((DataOutputStream) localObject1).writeShort(this.fields.size());
			Iterator localIterator3 = this.fields.iterator();
			while (localIterator3.hasNext()) {
				localObject2 = (FieldInfo) localIterator3.next();
				((FieldInfo) localObject2).write((DataOutputStream) localObject1);
			}
			((DataOutputStream) localObject1).writeShort(this.methods.size());
			localIterator3 = this.methods.iterator();
			while (localIterator3.hasNext()) {
				localObject2 = (MethodInfo) localIterator3.next();
				((MethodInfo) localObject2).write((DataOutputStream) localObject1);
			}
			((DataOutputStream) localObject1).writeShort(0);
		} catch (IOException localIOException2) {
			throw new InternalError("unexpected I/O Exception");
		}
		return ((B) (B) localByteArrayOutputStream.toByteArray());
	}
```
但究竟是如何将实现调用Connection的方法时而转而调用代理类Proxy的invoke方法的呢？
如下代码的UserDao可以理解为被代理类的接口，比如MyBatis的Connection（ 
   return (Connection) Proxy.newProxyInstance(cl, new Class[]{Connection.class}, handler);）
```language
public class Proxy0 extends Proxy implements UserDao {
  
      //第一步, 生成构造器
      protected Proxy0(InvocationHandler h) {
          super(h);
      }
  
      //第二步, 生成静态域
      private static Method m1;   //hashCode方法
     private static Method m2;   //equals方法
     private static Method m3;   //toString方法
     private static Method m4;   //...
     
     //第三步, 生成代理方法
     @Override
     public int hashCode() {
         try {
             return (int) h.invoke(this, m1, null);
         } catch (Throwable e) {
             throw new UndeclaredThrowableException(e);
         }
     }
     
     @Override
     public boolean equals(Object obj) {
         try {
             Object[] args = new Object[] {obj};
             return (boolean) h.invoke(this, m2, args);
         } catch (Throwable e) {
             throw new UndeclaredThrowableException(e);
         }
     }
     
     @Override
     public String toString() {
         try {
             return (String) h.invoke(this, m3, null);
         } catch (Throwable e) {
             throw new UndeclaredThrowableException(e);
         }
     }
     
     @Override
     public void save(User user) {
         try {
             //构造参数数组, 如果有多个参数往后面添加就行了
             Object[] args = new Object[] {user};
             h.invoke(this, m4, args);
         } catch (Throwable e) {
             throw new UndeclaredThrowableException(e);
         }
     }
     
     //第四步, 生成静态初始化方法
     static {
         try {
             Class c1 = Class.forName(Object.class.getName());
             Class c2 = Class.forName(UserDao.class.getName());    
             m1 = c1.getMethod("hashCode", null);
             m2 = c1.getMethod("equals", new Class[]{Object.class});
             m3 = c1.getMethod("toString", null);
             m4 = c2.getMethod("save", new Class[]{User.class});
             //...
         } catch (Exception e) {
             e.printStackTrace();
         }
     }
     
 }
```
如上静态变量m1、m2、m3、m4等实现为如下代码片断：
```language
//第二步, 组装要生成的class文件的所有的字段信息和方法信息
      try {
          //添加构造器方法
          methods.add(generateConstructor());
          //遍历缓存中的代理方法
          for (List<ProxyMethod> sigmethods : proxyMethods.values()) {
              for (ProxyMethod pm : sigmethods) {
                  //添加代理类的静态字段, 例如:private static Method m1;
                  fields.add(new FieldInfo(pm.methodFieldName,
                          "Ljava/lang/reflect/Method;", ACC_PRIVATE | ACC_STATIC));
                  //添加代理类的代理方法
                  methods.add(pm.generateMethod());
              }
          }
          //添加代理类的静态字段初始化方法
          methods.add(generateStaticInitializer());
      } catch (IOException e) {
          throw new InternalError("unexpected I/O Exception");
      }
```
1. 代码  fields.add(new FieldInfo(pm.methodFieldName, "Ljava/lang/reflect/Method;", ACC_PRIVATE | ACC_STATIC)); 完成上面m1、m2、m3、m4的static 常量定义，即：private static Method m1;
2. 代码：  methods.add(pm.generateMethod());其中的generateMethod()方法完成上面示例如下方法的实现，将代理类的方法实现定义 h.invoke(this, m4, args);
```language
     @Override
     public void save(User user) {
         try {
             //构造参数数组, 如果有多个参数往后面添加就行了
             Object[] args = new Object[] {user};
             h.invoke(this, m4, args);
         } catch (Throwable e) {
             throw new UndeclaredThrowableException(e);
         }
     }
```
其中generateMethod()方法中片段如下，其中code_aload、code_ipush等可以理解为Java字节码指令集的aload、sipus，即将指令及参数写入到DataOutputStream，这可是实打实的通过最原始的方法来实现一个方法。
```language

			DataOutputStream localDataOutputStream = new DataOutputStream(localMethodInfo.code);
			ProxyGenerator.this.code_aload(0, localDataOutputStream);
			localDataOutputStream.writeByte(180);
			localDataOutputStream.writeShort(ProxyGenerator.this.cp.getFieldRef("java/lang/reflect/Proxy", "h",
					"Ljava/lang/reflect/InvocationHandler;"));
			ProxyGenerator.this.code_aload(0, localDataOutputStream);
			localDataOutputStream.writeByte(178);
			localDataOutputStream.writeShort(
					ProxyGenerator.this.cp.getFieldRef(ProxyGenerator.access$000(ProxyGenerator.this.className),
							this.methodFieldName, "Ljava/lang/reflect/Method;"));
			if (this.parameterTypes.length > 0) {
				ProxyGenerator.this.code_ipush(this.parameterTypes.length, localDataOutputStream);
				localDataOutputStream.writeByte(189);
				localDataOutputStream.writeShort(ProxyGenerator.this.cp.getClass("java/lang/Object"));
				for (int k = 0; k < this.parameterTypes.length; ++k) {
					localDataOutputStream.writeByte(89);
					ProxyGenerator.this.code_ipush(k, localDataOutputStream);
					codeWrapArgument(this.parameterTypes[k], arrayOfInt[k], localDataOutputStream);
					localDataOutputStream.writeByte(83);
				}
			} else {
				localDataOutputStream.writeByte(1);
			}
			localDataOutputStream.writeByte(185);
			localDataOutputStream.writeShort(
					ProxyGenerator.this.cp.getInterfaceMethodRef("java/lang/reflect/InvocationHandler", "invoke",
							"(Ljava/lang/Object;Ljava/lang/reflect/Method;[Ljava/lang/Object;)Ljava/lang/Object;"));
			localDataOutputStream.writeByte(4);
			localDataOutputStream.writeByte(0);
			if (this.returnType == Void.TYPE) {
				localDataOutputStream.writeByte(87);
				localDataOutputStream.writeByte(177);
			} else {
				codeUnwrapReturnValue(this.returnType, localDataOutputStream);
			}
```
3. 代码： Class c2 = Class.forName(UserDao.class.getName());    m1 = c1.getMethod("hashCode", null); 
    1. 通过ProxyGenerator.this.codeClassForName(this.fromClass, paramDataOutputStream); 实现
遍历所有Class c2 = Class.forName(UserDao.class.getName());
    2. 通过 paramDataOutputStream.writeShort(ProxyGenerator.this.cp.getMethodRef("java/lang/Class", "getMethod","(Ljava/lang/String;[Ljava/lang/Class;)Ljava/lang/reflect/Method;")); 实现m1 = c1.getMethod("hashCode", null); 
```language
while (localIterator1.hasNext()) {
			List localList = (List) localIterator1.next();
			Iterator localIterator2 = localList.iterator();
			while (localIterator2.hasNext()) {
				ProxyMethod localProxyMethod = (ProxyMethod) localIterator2.next();
				localProxyMethod.codeFieldInitialization(localDataOutputStream);
			}
		}
```

```language

		private void codeFieldInitialization(DataOutputStream paramDataOutputStream) throws IOException {
			ProxyGenerator.this.codeClassForName(this.fromClass, paramDataOutputStream);
			ProxyGenerator.this.code_ldc(ProxyGenerator.this.cp.getString(this.methodName), paramDataOutputStream);
			ProxyGenerator.this.code_ipush(this.parameterTypes.length, paramDataOutputStream);
			paramDataOutputStream.writeByte(189);
			paramDataOutputStream.writeShort(ProxyGenerator.this.cp.getClass("java/lang/Class"));
			for (int i = 0; i < this.parameterTypes.length; ++i) {
				paramDataOutputStream.writeByte(89);
				ProxyGenerator.this.code_ipush(i, paramDataOutputStream);
				if (this.parameterTypes[i].isPrimitive()) {
					ProxyGenerator.PrimitiveTypeInfo localPrimitiveTypeInfo = ProxyGenerator.PrimitiveTypeInfo
							.get(this.parameterTypes[i]);
					paramDataOutputStream.writeByte(178);
					paramDataOutputStream.writeShort(ProxyGenerator.this.cp
							.getFieldRef(localPrimitiveTypeInfo.wrapperClassName, "TYPE", "Ljava/lang/Class;"));
				} else {
					ProxyGenerator.this.codeClassForName(this.parameterTypes[i], paramDataOutputStream);
				}
				paramDataOutputStream.writeByte(83);
			}
			paramDataOutputStream.writeByte(182);
			paramDataOutputStream.writeShort(ProxyGenerator.this.cp.getMethodRef("java/lang/Class", "getMethod",
					"(Ljava/lang/String;[Ljava/lang/Class;)Ljava/lang/reflect/Method;"));
			paramDataOutputStream.writeByte(179);
			paramDataOutputStream.writeShort(
					ProxyGenerator.this.cp.getFieldRef(ProxyGenerator.access$000(ProxyGenerator.this.className),
							this.methodFieldName, "Ljava/lang/reflect/Method;"));
		}
	}
```


