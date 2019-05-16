Socket类代表一个客户端套接字;ServerSocket是等待客户端的请求，一旦获得一个连接请求，就创建一个Socket示例来与客户端进行通信。 
```language
	ServerSocket serverSocket = new ServerSocket(port,3); 	
```
把连接请求队列的长度设为 3。这意味着当队列中有了 3 个连接请求时，如果 Client 再请求连接，就会被 Server拒绝，因为服务器队列已满。
#### 示例1：
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
其原因呢？时OutputStream为abstract 抽象类，其flush()默认为空方法，即调用没有任何作用；而只有SocketClient的