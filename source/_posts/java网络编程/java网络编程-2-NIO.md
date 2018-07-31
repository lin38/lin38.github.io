---
title: java网络编程-2-NIO
date: 2018-07-31 18:47:09
categories: java网络编程
tags: 
  - ServerSocketChannel
  - SocketChannel
toc: true
list_number: false
---

# 1、NIO的方式实现服务端、客户端通信

```java
package com.example.part_02_nio.demo001;

import java.io.IOException;
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.SelectionKey;
import java.nio.channels.Selector;
import java.nio.channels.ServerSocketChannel;
import java.nio.channels.SocketChannel;
import java.util.Iterator;

/**
 * NIO服务端
 */
public class Server {
	// 通道管理器
	private Selector selector;

	/**
	 * 获得一个ServerSocketChannel，并对该通道做一些初始化的工作
	 * 
	 * @param port
	 *            绑定的端口号
	 * @throws IOException
	 */
	public void initServer(int port) throws IOException {
		// 获得一个ServerSocket通道
		ServerSocketChannel serverChannel = ServerSocketChannel.open();
		// 设置通道为非阻塞
		serverChannel.configureBlocking(false);
		// 将该通道对应的ServerSocket绑定到port端口
		serverChannel.socket().bind(new InetSocketAddress(port));
		// 获得一个通道管理器
		this.selector = Selector.open();
		// 将通道管理器和该通道绑定，并为该通道注册SelectionKey.OP_ACCEPT事件,注册该事件后，
		// 当该事件到达时，selector.select()会返回，如果该事件没到达selector.select()会一直阻塞。
		serverChannel.register(selector, SelectionKey.OP_ACCEPT);
	}

	/**
	 * 采用轮询的方式监听selector上是否有需要处理的事件，如果有，则进行处理
	 * 
	 * @throws IOException
	 */
	public void listen() throws IOException {
		System.out.println("服务端启动成功！");
		// 轮询访问selector
		while (true) {
			// 当注册的事件到达时，方法返回；否则,该方法会一直阻塞
			selector.select();
			// 获得selector中选中的项的迭代器，选中的项为注册的事件
			Iterator<?> ite = this.selector.selectedKeys().iterator();
			while (ite.hasNext()) {
				SelectionKey key = (SelectionKey) ite.next();
				// 删除已选的key,以防重复处理
				ite.remove();
				// 进行处理
				handler(key);
			}
		}
	}

	/**
	 * 处理请求
	 * 
	 * @param key
	 * @throws IOException
	 */
	public void handler(SelectionKey key) throws IOException {
		if (key.isAcceptable()) { // 客户端请求连接事件
			handlerAccept(key);
		} else if (key.isReadable()) { // 获得了可读的事件
			handelerRead(key);
		}
	}

	/**
	 * 处理连接请求
	 * 
	 * @param key
	 * @throws IOException
	 */
	public void handlerAccept(SelectionKey key) throws IOException {
		ServerSocketChannel server = (ServerSocketChannel) key.channel();
		// 获得和客户端连接的通道
		SocketChannel channel = server.accept();
		// 设置成非阻塞
		channel.configureBlocking(false);

		// 在这里可以给客户端发送信息哦
		// 回写数据
		ByteBuffer outBuffer = ByteBuffer.wrap("成功建立连接".getBytes());
		channel.write(outBuffer);// 将消息回送给客户端
		System.out.println("新的客户端连接");
		// 在和客户端连接成功之后，为了可以接收到客户端的信息，需要给通道设置读的权限。
		// 回写数据时候，可以监听写事件
		channel.register(this.selector, SelectionKey.OP_READ);
	}

	/**
	 * 处理读的事件
	 * 
	 * @param key
	 * @throws IOException
	 */
	public void handelerRead(SelectionKey key) throws IOException {
		// 服务器可读取消息:得到事件发生的Socket通道
		SocketChannel channel = (SocketChannel) key.channel();
		// 创建读取的缓冲区
		ByteBuffer buffer = ByteBuffer.allocate(1024);
		int read = channel.read(buffer);
		if(read > 0){
			byte[] data = buffer.array();
			String msg = new String(data).trim();
			System.out.println("服务端收到信息：" + msg);
			
			// 回写数据
			ByteBuffer outBuffer = ByteBuffer.wrap(("好的" + msg).getBytes());
			channel.write(outBuffer);// 将消息回送给客户端
		}else{
			System.out.println("客户端关闭");
			key.cancel();
		}
	}

	/**
	 * 启动服务端测试
	 * 
	 * @throws IOException
	 */
	public static void main(String[] args) throws IOException {
		Server server = new Server();
		server.initServer(8899);
		server.listen();
	}

}
```

<!--more-->

```java
package com.example.part_02_nio.demo001;

import java.io.IOException;
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.SocketChannel;

public class Client {
	public static void main(String[] args) {

		// 创建连接的地址
		InetSocketAddress address = new InetSocketAddress("127.0.0.1", 8899);

		// 声明连接通道
		SocketChannel sc = null;

		// 建立缓冲区
		ByteBuffer buffer = ByteBuffer.allocate(1024);

		try {
			// 打开通道
			sc = SocketChannel.open();
			// 进行连接
			sc.connect(address);
			
			while (true) {
				int read = sc.read(buffer);
				if(read > 0) {
					byte[] data = buffer.array();
					String msg = new String(data).trim();
					System.out.println("客户端端收到信息：" + msg);
					buffer.clear();
				}
				
				// 定义一个字节数组，然后使用系统录入功能：
				byte[] bytes = new byte[1024];
				System.in.read(bytes);

				// 把数据放到缓冲区中
				buffer.put(bytes);
				// 对缓冲区进行复位
				buffer.flip();
				// 写出数据
				sc.write(buffer);
				// 清空缓冲区数据
				buffer.clear();
			}
		} catch (IOException e) {
			e.printStackTrace();
		} finally {
			if (sc != null) {
				try {
					sc.close();
				} catch (IOException e) {
					e.printStackTrace();
				}
			}
		}

	}

}
```



# 2、图解BIO与NIO

1. BIO

   ![](BIO.png)

   假设我们的应用系统是一家餐厅，那么可以把ServerSocket理解成餐厅的大门，以欢迎前来用餐的客人（即客户端），每来一个客人就需要一个服务员（线程）接待。

2. NIO

![](NIO.png)

同样假设为餐厅，可以把Selector理解为服务员，服务员并不需要时刻陪在每一个客人身边，只有当客人有需求的时候，如点餐、上菜、结账等，才需要服务员接待。这样就节省了很多服务员（线程）。



# 3、关于阻塞与非阻塞

​	阻塞与非阻塞说的是IO。

1. BIO：当发起IO的读或写操作时，均为阻塞方式，直到应用程序从流中读到数据或者将数据写入到流中。
2. NIO：基于事件驱动思想，当发起IO请求时，应用程序是非阻塞的。当有流可读或写的时候，由操作系统通知应用程序，应用程序再将流读取到缓冲区或者写入系统。



# 4、关于同步与异步

​	AIO：同样基于事件驱动的思想，在进行I/O操作时，直接调用API的read或write，这两种方法均为异步。对于读操作，操作系统将数据读到缓冲区，并通知应用程序，对于写操作，操作系统将write方法传递的流写入并主动通知应用程序。它节省了NIO中遍历事件通知队列的代价。

​	这里注意比较NIO和AIO的不同，AIO是操作系统完成IO并通知应用程序，NIO是操作系统通知应用程序，由应用程序完成。 



# 5、selector.select()方法

​        `select()`方法会一直阻塞，直到有注册的事件触发。

​	若不想一直阻塞，可通过其他方法实现：

1. `select(long timeout)`：超时返回
2. `selectNow()`：立马返回
3. `wakeup()`：唤醒阻塞的selector



# 6、SelectionKey.OP_WRITE

​	写就绪相对有一点特殊，一般来说，你不应该注册写事件。写操作的就绪条件为底层缓冲区有空闲空间，而写缓冲区绝大部分时间都是有空闲空间的，所以当你注册写事件后，写操作一直是就绪的，选择处理线程会占用整个CPU资源。所以，只有当你确实有数据要写时再注册写操作，并在写完以后马上取消注册。 

​	当有数据在写时，将数据写到缓冲区中，并注册写事件：

```java
public void write(byte[] data) throws IOException {
    writeBuffer.put(data);
    key.interestOps(SelectionKey.OP_WRITE);
}
```

​	注册写事件后，写操作就绪，这时将之前写入缓冲区的数据写入通道，并取消注册：

```java
channel.write(writeBuffer);
key.interestOps(key.interestOps() & ~SelectionKey.OP_WRITE);
```

