Socket类代表一个客户端套接字;ServerSocket是等待客户端的请求，一旦获得一个连接请求，就创建一个Socket示例来与客户端进行通信。 
```language
	ServerSocket serverSocket = new ServerSocket(port,3); 	
```
把连接请求队列的长度设为 3。这意味着当队列中有了 3 个连接请求时，如果 Client 再请求连接，就会被 Server拒绝，因为服务器队列已满。
#### 一. 基础示例：
```language
public class SocketClient {

	public static void main(String[] args) throws Exception, IOException {
		Socket socket = new Socket("127.0.0.1",9005);
		OutputStream out = socket.getOutputStream();
		long times = System.currentTimeMillis();
		for(int i=0;i<1000000;i++){
			out.write(("hello..." + times).getBytes());
			System.out.println("hello..." + times);
			out.flush();
		}
		out.write(("EOF").getBytes());
		out.flush();
		out.close();
		socket.close();
	}

}
```
```language

public class SocketServer {

	public static void main(String[] args) throws Exception {
		SocketServer socketServer =  new SocketServer();
		socketServer.init();
	}
	
	public void init() throws Exception{
		ServerSocket serverSocket = new ServerSocket(9005,2);
		serverSocket.setReuseAddress(true);
		while(true){
			Socket client = serverSocket.accept();
			InputStream in= client.getInputStream();
			Reader reader = new InputStreamReader(in);
			BufferedReader br = new BufferedReader(reader);
			String line ;
			while((line=br.readLine())!=null){
				System.out.println(line);
			}
		    }
		
	}

}
```
运行SocketServer后，依次分别运行3次SocketClient 结果分析下几点：
1. SocketServer连接请求队列的长度设为 2（new ServerSocket(9005,2)，第二个参数），第3次运行SocketClient时因前2次运行的请求仍在运行中使得服务器队列已先被前2次SocketClient占用，直接拒绝：
```language
Exception in thread "main" java.net.ConnectException: Connection refused: connect
	at java.net.DualStackPlainSocketImpl.connect0(Native Method)
	at java.net.DualStackPlainSocketImpl.socketConnect(Unknown Source)
	at java.net.AbstractPlainSocketImpl.doConnect(Unknown Source)
```
2. SocketClient每次写入后均flush，SocketServer打印为System.out.println，预期是期望SocketServer每次打印"hello...1557996285019"能自动换行，并在最后一行打印出"EOF"; 然而实际呢？实际SocketServer会把同一SocketClient的所有输入的"hello...1557996285019"及最后的"EOF"合并一行输出，类似：
```language
hello...1557995541663hello...1557995541663hello...1557995541663 中间还有无数个hello....EOF
```
其原因呢？时OutputStream为abstract 抽象类，而通过debug发现SocketClient类的socket.getOutputStream()实际返回的是 java.net.SocketOutputStream实例， 查看源码发现SocketOutputStream类及其父类并没有重写 OutputStream类的flush()方法（空方法实现）；最终只有SocketClient的out.close()方法：会自动刷新缓冲区然后关闭流对象。（官文解释：flush方法会将缓存中的数据立即强制刷新，而这些被强制刷新的数据会交给操作系统，至于操作系统什么时候讲这些输入写到管道中，这并不是我们能控制的）
```language
   OutputStream类flush默认空实现
   public void flush() throws IOException {
   }
```
> 合并一行输出呢？其实是因为SocketClient缓冲，直到close方法调用时flush才会作为一个完整的数据包发送到SocketServer。
3. 异常场景：
   1. 如若未先运行SocketServer而直接执行SocketClient报错：Connection refused: connect ；
   2. 如若如SocketClient正在运行的请求已达到SocketServer请求队列长度，报错：Connection refused: connect；
   3. 如若SocketClient在write循环执行（还未执行到close()方法）， SocketServer突然关闭SocketClient会报错：Connection reset by peer: socket write error；但若相反是SocketClient突然关闭SocketServer则会报错java.net.SocketException: Connection reset。
 
- 针对异常场景第2点，SocketClient与 SocketServer 仍一方中断均会导致另一方中断；考虑实际场景，若SocketServer 突然关闭SocketClient异常中断正常；但SocketClient异常中断而SocketServer异常 则不可接受。其实愿意很简单，就是SocketServer 在获取SocketClient后目前若有异常是直接往外抛的，适当调整加上异常捕获处理即可。
```language
public class SocketServer {

	public static void main(String[] args) throws Exception {
		SocketServer socketServer =  new SocketServer();
		socketServer.init();
	}
	
	public void init() throws Exception{
		ServerSocket serverSocket = new ServerSocket(9006,2);
		serverSocket.setReuseAddress(true);
		while(true){
			Socket client = serverSocket.accept();
			try{
				
				InputStream in= client.getInputStream();
				Reader reader = new InputStreamReader(in);
				BufferedReader br = new BufferedReader(reader);
				String line ;
				while((line=br.readLine())!=null){
					System.out.println(line);
				}
			}catch(Exception e){
				System.out.println("client exception:" + client.hashCode());
		}
		}

	}

}

```
如若 SocketClient 异常中断，则服务端报错： client exception:1311053135，但仍正常运行。

#### 二. 粘包（半包）问题
https://www.cnblogs.com/f-zhao/p/7502075.html   
https://blog.csdn.net/nongfuyumin/article/details/78298380
https://blog.csdn.net/m0_37739193/article/details/78738253
- 那么针对上面的示例，在实际应用场景中，常有如长连接，一旦连接建立可分多次传送数据且希望服务可正常识别，不会出现示例1的所有数据拼接成一行的情况？其实就是粘包。
- TCP是个“流”协议，所谓流，就是没有界限的一串数据。大家可以想象河里的流水，他们是连成一片的，其间并没有分界线。TCP底层并不了解上层业务数据的具体含义，他会根据TCP缓冲区的实际情况进行包的划分，所以在业务上认为，一个完整的包可能会被TCP拆分成多个包进行发送，也有可能把多个小的包封装成一个大的数据包发送。这就是TCP所谓的拆包和粘包的问题。
- java socket半包、粘包问题解决方案
   1. 以特殊字符串比如/r、/n作为数据的结尾，这样就可以区分数据包了。
   2. 发送请求包的时候只发送固定长度的数据包，这样在服务端接收数据也只接收固定长度的数据，这种方法效率太低，不太合适频繁的数据包请求。
   3. 在tcp协议的基础上封装一层数据请求协议，既数据包=数据包长度+数据包内容，这样在服务端就可以知道每个数据包的长度，也就可以解决半包、粘包问题
- 1.特殊字符串/n 分隔
```language
/*
 * 解决粘包：通过/n分隔
 * **/
public class SocketClient2 {

	public static void main(String[] args) throws Exception, IOException {
		Socket socket = new Socket("127.0.0.1",9006);
		OutputStream out = socket.getOutputStream();
		BufferedOutputStream bop = new BufferedOutputStream(out);
		long times = System.currentTimeMillis();
		for(int i=0;i<30;i++){
			if(i>0 && i%1000==0) Thread.sleep(30000l);
			System.out.println("hello..." + times);
			System.out.println("/n");
		}
		bop.close();
		socket.close();
	}

}

```
```language
/*解决粘包：通过/n分隔
 * 
 * **/
public class SocketServer2 {

	public static void main(String[] args) throws Exception {
		SocketServer2 socketServer =  new SocketServer2();
		socketServer.init();
	}
	
	public void init() throws Exception{
		ServerSocket serverSocket = new ServerSocket(9006,2);
		serverSocket.setReuseAddress(true);
		/*boolean eof = false ;*/
		while(true){
			Socket client = serverSocket.accept();
			InputStream in= client.getInputStream();
			Reader reader = new InputStreamReader(in);
			BufferedReader br = new BufferedReader(reader);
			String line ;
			while((line=br.readLine())!=null){
				String[] lines = line.split("/n");
				for(String str:lines)
					System.out.println(str);
			}
		}
	}

}

/***
 * 运行结果：
hello...1558332560890
/n
hello...1558332560890
/n
hello...1558332560890
/n
hello...1558332560890
 */
```
此种方案相对简单，但不够通用，比如客户端请求数据就在/n 串则会导致服务端处理异常；可考虑对客户端请求数据编码避免出现/n类似字符串。
- 2.发送请求包发送固定长度的数据包
该方案不做考虑，一方面为了固定方案必须对短的字符串填充空白字符串；另外对于客户端请求报文长短差异较大的场景不适合。
- 3.封装请求协议，即数据包=数据包长度 + 数据包内容（简易示例）
```language
package com.zyl.base.io;

import java.io.BufferedOutputStream;
import java.io.IOException;
import java.io.OutputStream;
import java.net.Socket;

/*
 * 解决粘包：自定义协议  数据包 =  数据包长度+数据包内容
 * **/
public class SocketClient3 {

	public static void main(String[] args) throws Exception, IOException {
		Socket socket = new Socket("127.0.0.1",9006);
		OutputStream out = socket.getOutputStream();
		BufferedOutputStream bop = new BufferedOutputStream(out);
		long times = System.currentTimeMillis();
		for(int i=0;i<30;i++){
			if(i>0 && i%1000==0) Thread.sleep(30000l);
			String inStr = "hello..." + times ;
			byte[] inByte = inStr.getBytes("UTF-8");
			byte[] inLen = int2ByteArray(inByte.length);
			bop.write(inLen);
			bop.write(inByte);
			System.out.println(inStr);
		}
		bop.close();
		socket.close();
	}
	
	private final static byte[] int2ByteArray(int i){
         byte[] result=new byte[4];
         result[0]=(byte)((i >> 24)& 0xFF);
         result[1]=(byte)((i >> 16)& 0xFF);
         result[2]=(byte)((i >> 8)& 0xFF);
         result[3]=(byte)(i & 0xFF);
         return result;
     }

}

```
```language
public class SocketServer3 {

	public static void main(String[] args) throws Exception {
		SocketServer3 socketServer =  new SocketServer3();
		socketServer.init();
	}
	
	public void init() throws Exception{
		ServerSocket serverSocket = new ServerSocket(9006,2);
		serverSocket.setReuseAddress(true);
		while(true){
			Socket client = serverSocket.accept();//点1
			InputStream in= client.getInputStream();
			byte[] inLen = null ;
			byte[] inByte = null ;
			for(;;){//点2
				inLen = new byte[4];
				int readCount = in.read(inLen);
				if(readCount == -1){
					break;//点3：开始错写成continue
				}
				int length = bytes2Int(inLen);
				inByte = new byte[length];
				in.read(inByte);
				String inData = new String(inByte,"UTF-8");
				System.out.println("length:" + length + ",String:" + inData);
			}
		}
	}
	
	private final static int bytes2Int(byte[] bytes){
         int num=bytes[3] & 0xFF;
         num |=((bytes[2] <<8)& 0xFF00);
         num |=((bytes[1] <<16)& 0xFF0000);
         num |=((bytes[0] <<24)& 0xFF0000);
         return num;
	}

}
/**
 * 
length:21,String:hello...1558335696449
length:21,String:hello...1558335696449
length:21,String:hello...1558335696449
..........
 * 
 * */
```
点1：SocketServer3的while(true)循环简单是否有新的客户端请求；而客户端SocketClient3 每次请求完数据就会close即可认为客户端关闭的连接；此处是忙等。SocketClient3 每执行一次SocketServer3的Socket client = serverSocket.accept();就会检测到，然后等待输入。一次InputStream in= client.getInputStream();对应SocketClient3一次完整的所有数据。--可认为client.getInputStream()会一次性读取到客户端请求的所有数据。
点2：开始未添加for循环，验证时发现server端仅读取到一行就不再输出，是因为client.getInputStream()会一次性读取到客户端请求的所有数据，没有for循环则发现仅分别读取了一次int和data后就待待再一次客户端输入了。
点3：开始错写成continue，发现SocketClient3 第一次执行SocketServer3可完整获取到请求的输入，但SocketClient3 第二次执行时发现SocketServer3没有任何反应。原来是因为该处如果是continue会导致始终在for循环处无限等待；而实际如果需要获SocketClient3 第二次请求的数据则需要跳出for去执行while循环中的accept和getInputStream。

#### 三. 基于Selector&Channel实现
```language
/*
 * 解决粘包：自定义协议  数据包 =  数据包长度+数据包内容
 * **/
public class NIOClient {

	public static void main(String[] args) throws Exception, IOException {
		Socket socket = new Socket("127.0.0.1",9006);
		OutputStream out = socket.getOutputStream();
		BufferedOutputStream bop = new BufferedOutputStream(out);
		long times = System.currentTimeMillis();
		for(int i=0;i<5;i++){
			if(i>0 && i%1000==0) Thread.sleep(30000l);
			String inStr = "hello..." + times ;
			byte[] inByte = inStr.getBytes("UTF-8");
			byte[] inLen = int2ByteArray(inByte.length);
			bop.write(inLen);
			bop.write(inByte);
			System.out.println(inStr);
		}
		bop.close();
		socket.close();
	}
	
	private final static byte[] int2ByteArray(int i){
         byte[] result=new byte[4];
         result[0]=(byte)((i >> 24)& 0xFF);
         result[1]=(byte)((i >> 16)& 0xFF);
         result[2]=(byte)((i >> 8)& 0xFF);
         result[3]=(byte)(i & 0xFF);
         return result;
     }

}
```
```language
/***
 * NIO必须由Selector和Channel（ServerSocketChannel、SocketChannel）配合使用
 * jdk class api: https://docs.oracle.com/javase/7/docs/api/allclasses-noframe.html
 * 三个重要概念：
 * 	 Selector ： https://docs.oracle.com/javase/7/docs/api/java/nio/channels/Selector.html
 * 	 ServerSocket：
 * 	 SelectionKey : https://docs.oracle.com/javase/7/docs/api/java/nio/channels/SelectionKey.html
 * 
 * */
public class NIOServer {

	public static void main(String[] args) throws Exception {
		Selector selector= Selector.open();
		//Selector selector2= SelectorProvider.provider().openSelector();
		
		ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
		serverSocketChannel.configureBlocking(false);
		serverSocketChannel.register(selector,SelectionKey.OP_ACCEPT); 
		
		ServerSocket serverSocket = serverSocketChannel.socket();
		SocketAddress socketAddress = new InetSocketAddress(9006);
		serverSocket.bind(socketAddress);
		while(true){
			selector.select(); //没有客户端socket请求进来之前会阻塞
			//此行错误，具体见selector的keys、selectKeys方法返回结果集  https://www.cnblogs.com/xianyijun/p/5422727.html
			//Set<SelectionKey> selectionKeys = selector.keys(); ////fix-3 , debug返回示例为： Collections$UnmodifiableSet<E> 
			Set<SelectionKey> selectionKeys = selector.selectedKeys(); //fix-3 , debug返回示例为： HashSet
			Iterator<SelectionKey> iteratorKeys = selectionKeys.iterator();
			while(iteratorKeys.hasNext()){
				SelectionKey selectionKey = iteratorKeys.next();
				iteratorKeys.remove();// fix-2
				if(selectionKey.isAcceptable()){
					 System.out.println("client accept....");
					ServerSocketChannel clientSocket = (ServerSocketChannel) selectionKey.channel();
					 SocketChannel clientSocketChannel = clientSocket.accept();
					 clientSocketChannel.configureBlocking(false);
					 clientSocketChannel.register(selector, SelectionKey.OP_READ);
				}else if(selectionKey.isReadable()){
					 System.out.println("client read....");
					SocketChannel clientSocketChannel = (SocketChannel) selectionKey.channel();
					System.out.println(readData(clientSocketChannel));
					clientSocketChannel.close();
				}
				//iteratorKeys.remove();// fix-1
			}
		}
	}

	private static  String readData(SocketChannel socketChannel) throws IOException{
		ByteBuffer byteBuffer = ByteBuffer.allocateDirect(1024);
		StringBuilder builder = new StringBuilder();
		while(true){
			byteBuffer.clear();
			int n = socketChannel.read(byteBuffer);
			if(n==-1){
				break;
			}
			byteBuffer.flip();
			int limit = byteBuffer.limit();
			char[] dst = new char[limit];
			for(int i=0 ; i<limit; i++){
				dst[i] = (char) byteBuffer.get(i);
			}
			builder.append(dst);
			byteBuffer.clear();
		}
		return builder.toString();
	}
}

多次修改后的执行成功：
client accept....
client read....
hello...1558503625136??hello...1558503625136??hello...1558503625136??hello...1558503625136??

```
##### 具体验证及修改过程：
1. 测试一
```language
client accept....
Exception in thread "main" client accept....
java.lang.NullPointerException
	at com.zyl.base.io.NIOServer.main(NIOServer.java:47)

```
2. 修改 fix-1：iteratorKeys.remove();//fix-1：
```language
client accept....
Exception in thread "main" java.lang.UnsupportedOperationException
	at java.util.Collections$UnmodifiableCollection$1.remove(Unknown Source)
	at com.zyl.base.io.NIOServer.main(NIOServer.java:55)

```
3. 修改 fix-2(注释掉 fix-1 )：iteratorKeys.remove();// fix-2
```language
Exception in thread "main" java.lang.UnsupportedOperationException
	at java.util.Collections$UnmodifiableCollection$1.remove(Unknown Source)
	at com.zyl.base.io.NIOServer.main(NIOServer.java:43)

```
- 该问题从报错来看，问题出在remove方法，而根据之前的理解，对于集合如List、Set、Map需要remove元素时，有两种方式：1.集合自身的remove方法；2.集合迭代器Iterator的remove方法（remove前需调用Iterator.next()移动下标，否则会报IllegalStateException）。但不能并发线程调用List(set、map）的add、remove方法，会提示IllegalStateException，但可同时调用List的add方法和Iterator的remvoe方法。
- 而此处的原因呢？开始一直很纠结，之后各种百度发现：
> Selector包含了三个选择键的集合，在刚创建的时候都是空的。
1. keys方法返回与selector关联的选择键集合。该集合包括当前通道在selector上注册的选择键。
2. selectedKeys方法返回Selector的已选择键集合，它包含相关通道被选择器判断已经准备好的，并且包含于key的interest集合中的操作，它是已注册的键集合的子集合，每个键都关联一个已经准备好至少一个操作的通道，每个键都持有一个内嵌的ready集合，指示了所关联的通道已经准备好的操作。
3. 已取消的键的集合，包含了调用cancel方法的选择键的集合，但还没有被注销，是已注册的键的集合的子集，它是selector对象的私有成员，无法直接访问。
- 上面示例我最先使用的是 Selector的keys方法，根据debug发现keys返回实例Collections$UnmodifiableSet（不可修改的set视图）；而selectKeys返回则是普通的HashSet（可修改），具体可查询资料：https://blog.csdn.net/iteye_11587/article/details/82681489，结合Selector抽象类的实现类SelectorImpl 源码：
```language
public abstract class SelectorImpl extends AbstractSelector {
	protected Set<SelectionKey> selectedKeys = new HashSet();
	protected HashSet<SelectionKey> keys = new HashSet();
	private Set<SelectionKey> publicKeys;
	private Set<SelectionKey> publicSelectedKeys;

	protected SelectorImpl(SelectorProvider arg0) {
		super(arg0);
		if (Util.atBugLevel("1.4")) {
			this.publicKeys = this.keys;
			this.publicSelectedKeys = this.selectedKeys;
		} else {
			this.publicKeys = Collections.unmodifiableSet(this.keys);
			this.publicSelectedKeys = Util.ungrowableSet(this.selectedKeys);
		}

	}

	public Set<SelectionKey> keys() {
		if (!this.isOpen() && !Util.atBugLevel("1.4")) {
			throw new ClosedSelectorException();
		} else {
			return this.publicKeys;
		}
	}

	public Set<SelectionKey> selectedKeys() {
		if (!this.isOpen() && !Util.atBugLevel("1.4")) {
			throw new ClosedSelectorException();
		} else {
			return this.publicSelectedKeys;
		}
	}

```

- 修改4. 保留一行remove方法，并将Selector的keys方法修改为selectKeys方法即可正常运行。但有两个问题：
1. 注释掉iteratorKeys.remove()执行报错， 原因是循环时会重复获得同一client请求的事件，重复获取socket异常：可理解为，Selector的selectKeys返回的是一个事件集合，使用完需手动移除。
2. 结果Server打印出来的有乱码：经分析系请求时写入的数据长度 inLen 为 int的byte格式直接显示异常，可考虑注释掉这块代码，但也可针对修改完善（粘包）：
```language
/*
 * 解决粘包：自定义协议  数据包 =  数据包长度+数据包内容
 * **/
public class NIOClient {

	public static void main(String[] args) throws Exception, IOException {
		Socket socket = new Socket("127.0.0.1",9006);
		OutputStream out = socket.getOutputStream();
		BufferedOutputStream bop = new BufferedOutputStream(out);
		long times = System.currentTimeMillis();
		for(int i=0;i<5;i++){
			if(i>0 && i%1000==0) Thread.sleep(30000l);
			String inStr = "hello..." + times ;
			byte[] inByte = inStr.getBytes("UTF-8");
			byte[] inLen = int2ByteArray(inByte.length);
			bop.write(inLen);
			bop.write(inByte);
			System.out.println(inStr);
		}
		bop.close();
		socket.close();
	}
	
	private final static byte[] int2ByteArray(int i){
         byte[] result=new byte[4];
         result[0]=(byte)((i >> 24)& 0xFF);
         result[1]=(byte)((i >> 16)& 0xFF);
         result[2]=(byte)((i >> 8)& 0xFF);
         result[3]=(byte)(i & 0xFF);
         return result;
     }

}
```
```language

/***
 * NIO必须由Selector和Channel（ServerSocketChannel、SocketChannel）配合使用
 * jdk class api: https://docs.oracle.com/javase/7/docs/api/allclasses-noframe.html
 * 三个重要概念：
 * 	 Selector ： https://docs.oracle.com/javase/7/docs/api/java/nio/channels/Selector.html
 * 	 ServerSocket：
 * 	 SelectionKey : https://docs.oracle.com/javase/7/docs/api/java/nio/channels/SelectionKey.html
 * 
 * */
public class NIOServer {

	public static void main(String[] args) throws Exception {
		Selector selector= Selector.open();
		//Selector selector2= SelectorProvider.provider().openSelector();
		
		ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
		serverSocketChannel.configureBlocking(false);
		serverSocketChannel.register(selector,SelectionKey.OP_ACCEPT); 
		
		ServerSocket serverSocket = serverSocketChannel.socket();
		SocketAddress socketAddress = new InetSocketAddress(9006);
		serverSocket.bind(socketAddress);
		while(true){
			selector.select(); //没有客户端socket请求进来之前会阻塞
			//此行错误，具体见selector的keys、selectKeys方法返回结果集  https://www.cnblogs.com/xianyijun/p/5422727.html
			//Set<SelectionKey> selectionKeys = selector.keys(); ////fix-3 , debug返回示例为： Collections$UnmodifiableSet<E> 
			Set<SelectionKey> selectionKeys = selector.selectedKeys(); //fix-3 , debug返回示例为： HashSet
			Iterator<SelectionKey> iteratorKeys = selectionKeys.iterator();
			while(iteratorKeys.hasNext()){
				SelectionKey selectionKey = iteratorKeys.next();
				iteratorKeys.remove();// fix-2
				if(selectionKey.isAcceptable()){
					 System.out.println("client accept....");
					ServerSocketChannel clientSocket = (ServerSocketChannel) selectionKey.channel();
					 SocketChannel clientSocketChannel = clientSocket.accept();
					 clientSocketChannel.configureBlocking(false);
					 clientSocketChannel.register(selector, SelectionKey.OP_READ);
				}else if(selectionKey.isReadable()){
					 System.out.println("client read....");
					SocketChannel clientSocketChannel = (SocketChannel) selectionKey.channel();
					readData(clientSocketChannel);
					clientSocketChannel.close();
				}
				//iteratorKeys.remove();// fix-1
			}
		}
	}

	private static  void readData(SocketChannel socketChannel) throws IOException{
		ByteBuffer byteBuffer = ByteBuffer.allocateDirect(1024);
		StringBuilder builder = null;
		List<Entry> list = new ArrayList<Entry>();
		
		while(true){
			byteBuffer.clear();
			int n = socketChannel.read(byteBuffer);
			if(n==-1){
				break;
			}
			byteBuffer.flip();
			int limit = byteBuffer.limit();
			int len_index = 0 ;
			int data_index = 0 ;
			int entry_index = 0 ;
			boolean lenFlag = true;
			
			for(int i=0 ; i<limit; i++){
				if(len_index<4&&lenFlag){
					if(len_index==0){
						Entry entry = new Entry();
						list.add(entry);
					}
					list.get(entry_index).getLenByte()[len_index]= byteBuffer.get(i);
					if(len_index==3){
						int len = bytes2Int(list.get(entry_index).getLenByte());
						list.get(entry_index).setLenInt(len);
						list.get(entry_index).initDataByte(len);
						lenFlag = false ;
						len_index = 0 ;
					}else{
						len_index++;
					}
					
				}else{
					if(data_index<list.get(entry_index).getDataByte().length){
						list.get(entry_index).getDataByte()[data_index] = (char) byteBuffer.get(i);
						if(data_index== list.get(entry_index).getDataByte().length-1){
							list.get(entry_index).setDataStr(String.valueOf(list.get(entry_index).getDataByte()));
							data_index = 0 ;
							entry_index++;
							lenFlag = true ;
						}else{
							data_index++;
						}
					}
						
				}
			}
			byteBuffer.clear();
		}
		for(Entry entry: list){
			builder = new StringBuilder();
			System.out.println(builder.append("length:").append(entry.lenInt).append(",String:").append(entry.dataStr));
		}
	}
	
	
	private final static int bytes2Int(byte[] bytes){
         int num=bytes[3] & 0xFF;
         num |=((bytes[2] <<8)& 0xFF00);
         num |=((bytes[1] <<16)& 0xFF0000);
         num |=((bytes[0] <<24)& 0xFF0000);
         return num;
	}
	
	private static class Entry{
		 byte[] lens = new byte[4];
		 char[] data;
		 
		 int lenInt = 0 ;
		 String dataStr ;
		 
		 
		 private byte[] getLenByte(){
			 return this.lens;
		 }
		 
		private char[] initDataByte(int len){
			data = new char[len];
			return data;
		}
		
		private char[] getDataByte(){
			return this.data;
		}
		
		private void setLenInt(int lenInt){
			this.lenInt = lenInt ;
		}
		
		private void setDataStr(String dataStr){
			this.dataStr = dataStr ;
		}
		
	}
}

/**
 * 运行结果：
client accept....
client read....
length:21,String:hello...1558607281166
length:21,String:hello...1558607281166
length:21,String:hello...1558607281166
length:21,String:hello...1558607281166
length:21,String:hello...1558607281166
 * 
 * */
```
