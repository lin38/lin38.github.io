---
title: java网络编程-3-NIO多Selector处理高并发
date: 2018-08-01 14:16:23
categories: java网络编程
tags: 
  - ServerSocketChannel
  - SocketChannel
  - Selector
toc: true
list_number: false
---

# 1、单Selector问题

​	对于并发不是很大的情况下，一个Selector负责轮询注册事件的变化，并对事件进行处理，是没有问题的。但是当并发比较大的时候，客户端连接比较多，且IO也比较多的情况，一个Selector肯定是不能满足对于系统性能的要求。

​	对于高并发的情况，给去的解决方案是：分为boss（一个或多个）和worker（多个）两种线程来处理，每一个线程都拥有自己的Selector。boss线程负责接收客户端请求，将accept到的SocketChannel转交给其中的一个worker线程去处理IO请求。

​	结合上一篇文章的图解，这一篇实现的效果，类似于：

![](多Selector.png)

​	通俗解释：用餐高峰期，餐厅客人较多，一个服务员肯定是忙不过来的，这个时候就需要多个服务员来接待客人了。

<!--more-->



# 2、代码实现

```java
package com.example.part_02_nio.demo002;

import java.io.IOException;
import java.util.concurrent.Executor;
import java.util.concurrent.atomic.AtomicInteger;

/**
 * 线程池管理对象，用于管理boss线程和worker线程
 */
public class Pool {

	/**
	 * boss线程数组
	 */
	private final AtomicInteger bossIndex = new AtomicInteger();
	private Boss[] bosses;

	/**
	 * worker线程数组
	 */
	private final AtomicInteger workerIndex = new AtomicInteger();
	private Worker[] workeres;

	
	public Pool(Executor bossExecutor, Executor workerExecutor) throws IOException {
		initBoss(bossExecutor, 1);
		initWorker(workerExecutor, Runtime.getRuntime().availableProcessors() * 2);
	}

	/**
	 * 初始化boss线程
	 * @param boss
	 * @param count
	 * @throws IOException 
	 */
	private void initBoss(Executor bossExecutor, int count) throws IOException {
		this.bosses = new Boss[count];
		for (int i = 0; i < bosses.length; i++) {
			bosses[i] = new Boss("boss thread " + (i+1), this);
			bossExecutor.execute(bosses[i]);
		}

	}

	/**
	 * 初始化worker线程
	 * @param worker
	 * @param count
	 * @throws IOException 
	 */
	private void initWorker(Executor workerExecutor, int count) throws IOException {
		this.workeres = new Worker[count];
		for (int i = 0; i < workeres.length; i++) {
			workeres[i] = new Worker("worker thread " + (i+1));
			workerExecutor.execute(workeres[i]);
		}
	}

	/**
	 * 获取一个worker（取模方式，均匀分配）
	 * @return
	 */
	public Worker nextWorker() {
		 return workeres[Math.abs(workerIndex.getAndIncrement() % workeres.length)];

	}

	/**
	 * 获取一个boss（取模方式，均匀分配）
	 * @return
	 */
	public Boss nextBoss() {
		 return bosses[Math.abs(bossIndex.getAndIncrement() % bosses.length)];
	}

}
```

```java
package com.example.part_02_nio.demo002;

import java.io.IOException;
import java.nio.channels.ClosedChannelException;
import java.nio.channels.SelectionKey;
import java.nio.channels.Selector;
import java.nio.channels.ServerSocketChannel;
import java.nio.channels.SocketChannel;
import java.util.Iterator;
import java.util.Set;
import java.util.concurrent.ConcurrentLinkedQueue;
import java.util.concurrent.atomic.AtomicBoolean;

/**
 * Boss线程 ，用来接收ServerSocketChannel的注册，并监听OP_ACCEPT事件，
 * 并将accept到的SocketChannel请求转交给Worker处理
 */
public class Boss implements Runnable{

	// 线程名称
	private String threadName;
	// 管理boss和worker的池对象
	private Pool pool;
	// 选择器
	private Selector selector = Selector.open();
	// selector的状态标志位，判断selector当前是否处于阻塞状态
	private AtomicBoolean blocking = new AtomicBoolean(true);
	// 任务列表（无界队列）
	private ConcurrentLinkedQueue<Runnable> taskQueue = new ConcurrentLinkedQueue<>();
	
	public Boss(String threadName, Pool pool) throws IOException {
		this.threadName = threadName;
		this.pool = pool;
	}
	
	@Override
	public void run() {
		Thread.currentThread().setName(threadName);
		
		while(true) {
			
			try {
				// 每次循环，都将阻塞状态置为true
				blocking.set(true);
				
				// 监听注册时间，只有监听到事件时或者调用selector.wakeup()方法时才会返回
				selector.select();
				
				// 处理任务队列
				processQueue();
				
				// 处理监听到的请求
				processEvent();
				
			} catch (IOException e) {
				e.printStackTrace();
			}
		}
		
	}
	
	private void processQueue() {
		while(true) {
			Runnable task = taskQueue.poll();
			if(task == null) {
				break;
			}
			System.out.println(Thread.currentThread().getName() + "：从任务队列中执行任务");
			task.run();
		}
	}
	
	private void processEvent() {
		Set<SelectionKey> selectedKeys = selector.selectedKeys();
		if(selectedKeys == null || selectedKeys.isEmpty()) {
			return;
		}
		
		Iterator<SelectionKey> iterator = selectedKeys.iterator();
		while(iterator.hasNext()) {
			SelectionKey key = iterator.next();
			// 删除掉，以防重复处理
			iterator.remove();
			
			if(key.isAcceptable()) {
				try {
					System.out.println(Thread.currentThread().getName() + "：获取到客户端连接事件");
					
					ServerSocketChannel serverSocketChannel = (ServerSocketChannel) key.channel();
					
					SocketChannel socketChannel = serverSocketChannel.accept();
					// 设置为非阻塞
					socketChannel.configureBlocking(false);
					// 将接受到的客户端连接转给worker去处理
					System.out.println(Thread.currentThread().getName() + "：将客户端连接转交给worker去处理");
					this.pool.nextWorker().registSocketChannel(socketChannel);
				} catch (IOException e) {
					e.printStackTrace();
				}
			}
		}
		
	}
	
	/**
	 * 提供注册新管道的方法
	 */
	public void registServerSocketChannel(ServerSocketChannel channel) {
		if(this.selector != null) {
//			System.out.println(Thread.currentThread().getName() + "：接收到新管道的注册，已加入到任务队列");
			// 向任务队列里增加任务
			taskQueue.offer(new Runnable() {
				@Override
				public void run() {
					try {
						channel.register(selector, SelectionKey.OP_ACCEPT);
					} catch (ClosedChannelException e) {
						e.printStackTrace();
					}
				}
			});
			
			// 唤醒selector，不要继续阻塞，此时需要处理队列里的任务
			while(blocking.compareAndSet(true, false)) {
				this.selector.wakeup();
			}
		}
	}
	
}
```

```java
package com.example.part_02_nio.demo002;

import java.io.IOException;
import java.nio.ByteBuffer;
import java.nio.channels.ClosedChannelException;
import java.nio.channels.SelectionKey;
import java.nio.channels.Selector;
import java.nio.channels.SocketChannel;
import java.util.Iterator;
import java.util.Set;
import java.util.concurrent.ConcurrentLinkedQueue;

/**
 * Worker线程 ，用来接收Boss端accept到的SocketChannel的注册，并监听OP_READ事件，完成IO操作
 */
public class Worker implements Runnable{

	// 线程名称
	private String threadName;
	// 选择器
	private Selector selector = Selector.open();
	// 任务列表（无界队列）
	private ConcurrentLinkedQueue<Runnable> taskQueue = new ConcurrentLinkedQueue<>();
	
	public Worker(String threadName) throws IOException {
		this.threadName = threadName;
	}
	
	@Override
	public void run() {
		Thread.currentThread().setName(threadName);
		
		while(true) {
			
			try {
				// 监听注册时间0.5s，监听到则处理，超时未返回
				selector.select(500);
				
				// 处理任务队列
				processQueue();
				
				// 处理监听到的请求
				processEvent();
				
			} catch (IOException e) {
				e.printStackTrace();
			}
		}
		
	}
	
	private void processQueue() {
		while(true) {
			Runnable task = taskQueue.poll();
			if(task == null) {
				break;
			}
			System.out.println(Thread.currentThread().getName() + "：从任务队列中执行任务");
			task.run();
		}
	}
	
	private void processEvent() {
		Set<SelectionKey> selectedKeys = selector.selectedKeys();
		if(selectedKeys == null || selectedKeys.isEmpty()) {
			return;
		}
		
		Iterator<SelectionKey> iterator = selectedKeys.iterator();
		while(iterator.hasNext()) {
			SelectionKey key = iterator.next();
			// 删除掉，以防重复处理
			iterator.remove();
			
			if(key.isReadable()) {
				System.out.println(Thread.currentThread().getName() + "：获取到客户端写入事件");
				SocketChannel channel = (SocketChannel) key.channel();
				ByteBuffer byteBuffer = ByteBuffer.allocate(1024);
				try {
					int read = channel.read(byteBuffer);
					if(read > 0) {
						byte[] bs = byteBuffer.array();
						String msg = new String(bs).trim();
						System.out.println(Thread.currentThread().getName() + "：获取到客户端写入信息——" + msg);
						ByteBuffer responseBuffer = ByteBuffer.wrap(("response：" + msg).getBytes());
						channel.write(responseBuffer);
					} else {
						System.out.println(Thread.currentThread().getName() + "：客户端已经关闭连接");
						key.cancel(); // 不再接受该注册事件
					}
				} catch (IOException e) {
					e.printStackTrace();
				}
				
			}
		}
		
	}
	
	/**
	 * 提供注册新管道的方法
	 */
	public void registSocketChannel(SocketChannel channel) {
		if(this.selector != null) {
//			System.out.println(Thread.currentThread().getName() + "：接收到新管道的注册，已加入到任务队列");
			taskQueue.offer(new Runnable() {
				@Override
				public void run() {
					try {
						channel.register(selector, SelectionKey.OP_READ);
					} catch (ClosedChannelException e) {
						e.printStackTrace();
					}
				}
			});
		}
	}
	
}
```

```java
package com.example.part_02_nio.demo002;

import java.io.IOException;
import java.net.InetSocketAddress;
import java.nio.channels.ServerSocketChannel;
import java.util.concurrent.Executor;
import java.util.concurrent.Executors;

public class Server {

	/**
	 * 启动服务端测试
	 */
	public static void main(String[] args) throws IOException {
		Executor bossExecutor = Executors.newCachedThreadPool();
		Executor workerExecutor = Executors.newCachedThreadPool();

		Pool pool = new Pool(bossExecutor, workerExecutor);

		// 获得一个ServerSocket通道
		ServerSocketChannel serverChannel = ServerSocketChannel.open();
		// 设置通道为非阻塞
		serverChannel.configureBlocking(false);
		// 将该通道对应的ServerSocket绑定到port端口
		serverChannel.socket().bind(new InetSocketAddress(8899));

		pool.nextBoss().registServerSocketChannel(serverChannel);

		// 获得另一个ServerSocket通道
		ServerSocketChannel serverChannel2 = ServerSocketChannel.open();
		// 设置通道为非阻塞
		serverChannel2.configureBlocking(false);
		// 将该通道对应的ServerSocket绑定到port端口
		serverChannel2.socket().bind(new InetSocketAddress(8889));

		pool.nextBoss().registServerSocketChannel(serverChannel2);
	}

}
```

```java
package com.example.part_02_nio.demo002;

import java.io.IOException;
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.SocketChannel;
import java.util.concurrent.CountDownLatch;

/**
 * 客户端模拟高并发
 */
public class Client {
	public static void main(String[] args) {

		int count = 16;

		CountDownLatch latch = new CountDownLatch(count);

		for (int i = 1; i <= count; i++) {

			final int index = i;

			new Thread(new Runnable() {

				@Override
				public void run() {
					try {
						// 16个线程阻塞到这里，一起执行，实现并发效果
						latch.countDown();
						latch.await();

						// 创建连接的地址
						InetSocketAddress address = new InetSocketAddress("127.0.0.1", index / 2 == 0 ? 8899 : 8889);

						// 声明连接通道
						SocketChannel sc = null;

						// 建立缓冲区
						ByteBuffer buffer = ByteBuffer.allocate(1024);

						try {
							// 打开通道
							sc = SocketChannel.open();
							// 进行连接
							sc.connect(address);

							// 写出数据
							sc.write(ByteBuffer.wrap(("请求" + index).getBytes()));
							// 清空缓冲区数据
							buffer.clear();

							int read = sc.read(buffer);
							if (read > 0) {
								byte[] data = buffer.array();
								String msg = new String(data).trim();
								System.out.println("客户端端收到信息：" + msg);
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
					} catch (InterruptedException e) {
						e.printStackTrace();
					}
				}
			}).start();
		}
	}

}
```



# 3、测试结果

1. server端

   ```
   boss thread 1：从任务队列中执行任务
   boss thread 1：从任务队列中执行任务
   boss thread 1：获取到客户端连接事件
   boss thread 1：将客户端连接转交给worker去处理
   boss thread 1：获取到客户端连接事件
   boss thread 1：将客户端连接转交给worker去处理
   boss thread 1：获取到客户端连接事件
   boss thread 1：将客户端连接转交给worker去处理
   boss thread 1：获取到客户端连接事件
   boss thread 1：将客户端连接转交给worker去处理
   boss thread 1：获取到客户端连接事件
   boss thread 1：将客户端连接转交给worker去处理
   boss thread 1：获取到客户端连接事件
   boss thread 1：将客户端连接转交给worker去处理
   boss thread 1：获取到客户端连接事件
   boss thread 1：将客户端连接转交给worker去处理
   boss thread 1：获取到客户端连接事件
   boss thread 1：将客户端连接转交给worker去处理
   boss thread 1：获取到客户端连接事件
   boss thread 1：将客户端连接转交给worker去处理
   boss thread 1：获取到客户端连接事件
   boss thread 1：将客户端连接转交给worker去处理
   boss thread 1：获取到客户端连接事件
   boss thread 1：将客户端连接转交给worker去处理
   boss thread 1：获取到客户端连接事件
   boss thread 1：将客户端连接转交给worker去处理
   boss thread 1：获取到客户端连接事件
   boss thread 1：将客户端连接转交给worker去处理
   boss thread 1：获取到客户端连接事件
   boss thread 1：将客户端连接转交给worker去处理
   boss thread 1：获取到客户端连接事件
   boss thread 1：将客户端连接转交给worker去处理
   boss thread 1：获取到客户端连接事件
   boss thread 1：将客户端连接转交给worker去处理
   worker thread 1：从任务队列中执行任务
   worker thread 2：从任务队列中执行任务
   worker thread 4：从任务队列中执行任务
   worker thread 3：从任务队列中执行任务
   worker thread 4：从任务队列中执行任务
   worker thread 5：从任务队列中执行任务
   worker thread 6：从任务队列中执行任务
   worker thread 1：从任务队列中执行任务
   worker thread 2：从任务队列中执行任务
   worker thread 4：获取到客户端写入事件
   worker thread 5：从任务队列中执行任务
   worker thread 6：从任务队列中执行任务
   worker thread 3：从任务队列中执行任务
   worker thread 8：从任务队列中执行任务
   worker thread 7：从任务队列中执行任务
   worker thread 6：获取到客户端写入事件
   worker thread 5：获取到客户端写入事件
   worker thread 6：获取到客户端写入信息——请求9
   worker thread 3：获取到客户端写入事件
   worker thread 3：获取到客户端写入信息——请求6
   worker thread 6：获取到客户端写入事件
   worker thread 1：获取到客户端写入事件
   worker thread 4：获取到客户端写入信息——请求3
   worker thread 1：获取到客户端写入信息——请求12
   worker thread 7：从任务队列中执行任务
   worker thread 3：获取到客户端写入事件
   worker thread 1：获取到客户端写入事件
   worker thread 3：获取到客户端写入信息——请求2
   worker thread 1：获取到客户端写入信息——请求10
   worker thread 1：获取到客户端写入事件
   worker thread 1：客户端已经关闭连接
   worker thread 2：获取到客户端写入事件
   worker thread 2：获取到客户端写入信息——请求14
   worker thread 2：获取到客户端写入事件
   worker thread 2：获取到客户端写入信息——请求15
   worker thread 2：获取到客户端写入事件
   worker thread 8：从任务队列中执行任务
   worker thread 5：获取到客户端写入信息——请求5
   worker thread 8：获取到客户端写入事件
   worker thread 5：获取到客户端写入事件
   worker thread 5：获取到客户端写入信息——请求8
   worker thread 5：获取到客户端写入事件
   worker thread 5：客户端已经关闭连接
   worker thread 2：客户端已经关闭连接
   worker thread 1：获取到客户端写入事件
   worker thread 1：客户端已经关闭连接
   worker thread 5：获取到客户端写入事件
   worker thread 5：客户端已经关闭连接
   worker thread 8：获取到客户端写入信息——请求4
   worker thread 2：获取到客户端写入事件
   worker thread 2：客户端已经关闭连接
   worker thread 8：获取到客户端写入事件
   worker thread 8：获取到客户端写入信息——请求16
   worker thread 7：获取到客户端写入事件
   worker thread 3：获取到客户端写入事件
   worker thread 3：客户端已经关闭连接
   worker thread 3：获取到客户端写入事件
   worker thread 3：客户端已经关闭连接
   worker thread 6：获取到客户端写入信息——请求1
   worker thread 4：获取到客户端写入事件
   worker thread 7：获取到客户端写入信息——请求13
   worker thread 6：获取到客户端写入事件
   worker thread 7：获取到客户端写入事件
   worker thread 7：获取到客户端写入信息——请求11
   worker thread 7：获取到客户端写入事件
   worker thread 7：客户端已经关闭连接
   worker thread 8：获取到客户端写入事件
   worker thread 7：获取到客户端写入事件
   worker thread 7：客户端已经关闭连接
   worker thread 6：客户端已经关闭连接
   worker thread 4：获取到客户端写入信息——请求7
   worker thread 6：获取到客户端写入事件
   worker thread 6：客户端已经关闭连接
   worker thread 8：客户端已经关闭连接
   worker thread 4：获取到客户端写入事件
   worker thread 4：客户端已经关闭连接
   worker thread 4：获取到客户端写入事件
   worker thread 4：客户端已经关闭连接
   worker thread 8：获取到客户端写入事件
   worker thread 8：客户端已经关闭连接
   ```

2. client端

   ```
   客户端端收到信息：response：请求6
   客户端端收到信息：response：请求9
   客户端端收到信息：response：请求3
   客户端端收到信息：response：请求10
   客户端端收到信息：response：请求12
   客户端端收到信息：response：请求2
   客户端端收到信息：response：请求14
   客户端端收到信息：response：请求15
   客户端端收到信息：response：请求5
   客户端端收到信息：response：请求8
   客户端端收到信息：response：请求4
   客户端端收到信息：response：请求16
   客户端端收到信息：response：请求1
   客户端端收到信息：response：请求11
   客户端端收到信息：response：请求13
   客户端端收到信息：response：请求7
   ```

3. 结果分析：

   - 一个Selector可以注册多个channel，如代码中的boss就注册了两个ServerSocketChannel，分别为8899和8889
   - 一个系统中可以存在多个Selector，代码中每一个boss/worker线程，都拥有一个Selector
   - 通过结果输出可以看出，客户端的IO请求被均匀的分配到各个worker线程去处理，boss只是负责接收连接

