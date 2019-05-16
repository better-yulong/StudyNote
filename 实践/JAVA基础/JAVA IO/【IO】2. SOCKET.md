Socket类代表一个客户端套接字;ServerSocket是等待客户端的请求，一旦获得一个连接请求，就创建一个Socket示例来与客户端进行通信。 
```language
	ServerSocket serverSocket = new ServerSocket(port,3); 	
```
把连接请求队列的长度设为 3。这意味着当队列中有了 3 个连接请求时，如果 Client 再请求连接，就会被 Server拒绝，因为服务器队列已满。