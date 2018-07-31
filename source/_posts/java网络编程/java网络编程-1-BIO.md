---
title: java网络编程-1-BIO
date: 2018-07-31 14:55:57
categories: java网络编程
tags: 
  - Socket
  - ServerSocket
toc: true
list_number: false
---

# 1、传统的BIO编程

​	网络编程的基本模型是Server/Client模型，也就是两个进程直接进行相互通信，其中服务端提供配置信息（绑定的IP地址和监听的端口），客户端通过连接操作向服务端监听的地址发送连接请求，通过三次握手建立连接，如果连接成功，则双方即可进行通信（网络套接字Socket）。



# 2、传统的BIO实现通讯的方式及优缺点

## 2.1 服务端单线程形式

1. BIO的B即blocking，阻塞的意思，那么阻塞的点在哪呢？

   - `server.accept();`：服务端程序会一直阻塞在此处，直到有一个客户端连接过来（客户端程序存在此阻塞点）
   - `reader.readLine();`：服务端程序会一直阻塞在此处，直到客户端发送消息过来

2. 同一时刻，只能处理一个客户端请求

<!--more-->

```java
package com.example.part_01_bio.demo001;

import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStreamReader;
import java.io.PrintWriter;
import java.net.ServerSocket;
import java.net.Socket;

public class Server {

	public static final Integer PORT = 8899;

	public static void main(String[] args) throws Exception {

		/**
		 * 这种方式的问题： server.accept()会阻塞住，直到有一个客户端来访问才会向下执行，
		 * 但是一旦向下执行且未执行完成的这段时间，再有其他客户端访问，并不会立即被accept到。 也就是说，同一时刻，只能处理一个客户端请求。
		 * 所以在传统BIO中，不会采用这种形式，而是会在accept到客户端连接后，启动新的线程去异步处理请求，
		 * 服务端则继续accept新的客户端连接
		 */

		ServerSocket server = new ServerSocket(PORT);
		while (true) {
			Socket client = server.accept(); // 阻塞点！
			handle(client);
		}

	}

	private static void handle(Socket client) {
		try {
			BufferedReader reader = new BufferedReader(new InputStreamReader(client.getInputStream()));
			PrintWriter writer = new PrintWriter(client.getOutputStream(), true);

			while (true) {
				String request = reader.readLine();
				if (request == null) {
					break;
				}
				System.out.println("server accept:" + request);

				writer.println("response:" + request);
			}
		} catch (Exception e) {
			e.printStackTrace();
		} finally {
			if (client != null) {
				try {
					client.close();
				} catch (IOException e) {
					e.printStackTrace();
				}
			}
		}
	}
}
```

```java
package com.example.part_01_bio.demo001;

import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStreamReader;
import java.io.PrintWriter;
import java.net.Socket;
import java.net.UnknownHostException;

public class Client {

	public static void main(String[] args) {
		Socket client = null;

		try {
			client = new Socket("127.0.0.1", Server.PORT);
			PrintWriter writer = new PrintWriter(client.getOutputStream(), true);
			writer.println("request");

			BufferedReader reader = new BufferedReader(new InputStreamReader(client.getInputStream()));
			String request = reader.readLine(); // 阻塞点！
			System.out.println("client accept:" + request);
		} catch (UnknownHostException e) {
			e.printStackTrace();
		} catch (IOException e) {
			e.printStackTrace();
		} finally {
			if (client != null) {
				try {
					/**
					 * Closing this socket will also close the socket's
					 * {@link java.io.InputStream InputStream} and
					 * {@link java.io.OutputStream OutputStream}.
					 */
					client.close();
				} catch (IOException e) {
					e.printStackTrace();
				}
			}
		}

	}
}
```



## 2.2 服务端多线程形式

​	此方式是在服务端接收到一个客户端请求后，直接启动一个线程去处理这个请求，具体的处理过程，服务端不参与，服务端只是负责接收客户端的连接。因此，这种方式可以解决2.1中的同一时刻只能处理一个客户端请求的问题。

​	但是这种方式存在另一个问题，就是假如同一时刻有大量的客户端建立连接，则服务端需要创建大量的线程去处理这些请求，势必造成性能问题。

```java
package com.example.part_01_bio.demo001;

import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStreamReader;
import java.io.PrintWriter;
import java.net.ServerSocket;
import java.net.Socket;

public class Server2 {

	public static final Integer PORT = 8899;

	public static void main(String[] args) throws Exception {

		/**
		 * 这种方式解决了单线程情况下，服务端只能同时处理一个客户端请求的问题。 这种方式的问题：
		 * 每accept到一个客户端连接，就创建一个线程去处理客户端请求， 假如同一个时刻并发请求过多，则系统会创建很多的线程处理这些请求，
		 * 势必会影响系统性能。所以大多数情况下会使用线程池技术来管理线程。
		 */

		ServerSocket server = new ServerSocket(PORT);
		while (true) {
			Socket client = server.accept();
			new Thread(new Runnable() {

				@Override
				public void run() {
					handle(client);
				}
			}).start();
		}

	}

	private static void handle(Socket client) {
		try {
			BufferedReader reader = new BufferedReader(new InputStreamReader(client.getInputStream()));
			PrintWriter writer = new PrintWriter(client.getOutputStream(), true);

			while (true) {
				String request = reader.readLine();
				if (request == null) {
					break;
				}
				System.out.println("server accept:" + request);

				writer.println("response:" + request);
			}
		} catch (Exception e) {
			e.printStackTrace();
		} finally {
			if (client != null) {
				try {
					client.close();
				} catch (IOException e) {
					e.printStackTrace();
				}
			}
		}
	}
}
```



## 2.3 服务端线程池形式

​	这种方式采用线程池技术来管理线程，解决了2.2中高并发情况下大量创建线程的问题。

```java
package com.example.part_01_bio.demo001;

import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStreamReader;
import java.io.PrintWriter;
import java.net.ServerSocket;
import java.net.Socket;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class Server3 {

	public static final Integer PORT = 8899;

	public static void main(String[] args) throws Exception {

		/**
		 * 这种方式解决了线程创建过多的问题。但并不能解决阻塞的问题。
		 */

		ExecutorService threadPool = Executors.newFixedThreadPool(10); // 创建一个固定大小的线程池

		ServerSocket server = new ServerSocket(PORT);
		while (true) {
			Socket client = server.accept();
			threadPool.execute(new Runnable() {

				@Override
				public void run() {
					handler(client);
				}
			});
		}

	}

	private static void handler(Socket client) {
		try {
			BufferedReader reader = new BufferedReader(new InputStreamReader(client.getInputStream()));
			PrintWriter writer = new PrintWriter(client.getOutputStream(), true);

			while (true) {
				String request = reader.readLine();
				if (request == null) {
					break;
				}
				System.out.println("server accept:" + request);

				writer.println("response:" + request);
			}
		} catch (Exception e) {
			e.printStackTrace();
		} finally {
			if (client != null) {
				try {
					client.close();
				} catch (IOException e) {
					e.printStackTrace();
				}
			}
		}
	}
}
```

