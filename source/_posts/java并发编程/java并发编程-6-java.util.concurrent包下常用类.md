---
title: java并发编程-6-java.util.concurrent包下常用类
date: 2018-07-30 11:51:53
categories: java并发编程
tags: 
  - CyclicBarrier
  - CountDownLatch
  - Callable
  - Semaphore
toc: true
list_number: false
---

# 1、**CyclicBarrier**

​        假设有一个场景：每个线程代表一个跑步运动员，当运动员都准备好后，才能一起出发，只要有一个人没有准备好，其他人就要等待。

<center>示例：</center>

```java
package com.example.part_04_utils.demo001;

import java.io.IOException;
import java.util.Random;
import java.util.concurrent.BrokenBarrierException;
import java.util.concurrent.CyclicBarrier;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class UseCyclicBarrier {

	static class Runner implements Runnable {
		private CyclicBarrier barrier;
		private String name;

		public Runner(CyclicBarrier barrier, String name) {
			this.barrier = barrier;
			this.name = name;
		}

		@Override
		public void run() {
			try {
				Thread.sleep(1000 * (new Random()).nextInt(5));
				System.out.println(name + " 准备OK.");
				barrier.await();
				System.out.println(name + " Go!!");
			} catch (InterruptedException e) {
				e.printStackTrace();
			} catch (BrokenBarrierException e) {
				e.printStackTrace();
			}
		}
	}

	public static void main(String[] args) throws IOException, InterruptedException {
		CyclicBarrier barrier = new CyclicBarrier(3); // 3
		ExecutorService executor = Executors.newFixedThreadPool(3);

		executor.submit(new Thread(new Runner(barrier, "zhangsan")));
		executor.submit(new Thread(new Runner(barrier, "lisi")));
		executor.submit(new Thread(new Runner(barrier, "wangwu")));

		executor.shutdown();
	}

}
```

<!--more-->

执行结果：

```
zhangsan 准备OK.
lisi 准备OK.
wangwu 准备OK.
wangwu Go!!
zhangsan Go!!
lisi Go!!
```



# 2、**CountDownLatch**

​        经常用于监听初始化操作，等初始化完成后，通知主线程继续工作。

<center>示例：</center>

```java
package com.example.part_04_utils.demo002;

import java.util.concurrent.CountDownLatch;

public class UseCountDownLatch {

	public static void main(String[] args) {

		final CountDownLatch countDown = new CountDownLatch(2);

		Thread t1 = new Thread(new Runnable() {
			@Override
			public void run() {
				try {
					System.out.println("t1线程" + "等待其他线程处理完成...");
					countDown.await();
					System.out.println("t1线程继续执行...");
				} catch (InterruptedException e) {
					e.printStackTrace();
				}
			}
		}, "t1");

		Thread t2 = new Thread(new Runnable() {
			@Override
			public void run() {
				try {
					System.out.println("t2线程进行初始化操作...");
					Thread.sleep(3000);
					System.out.println("t2线程初始化完毕，通知t1线程继续...");
					countDown.countDown();
				} catch (InterruptedException e) {
					e.printStackTrace();
				}
			}
		});

		Thread t3 = new Thread(new Runnable() {
			@Override
			public void run() {
				try {
					System.out.println("t3线程进行初始化操作...");
					Thread.sleep(4000);
					System.out.println("t3线程初始化完毕，通知t1线程继续...");
					countDown.countDown();
				} catch (InterruptedException e) {
					e.printStackTrace();
				}
			}
		});

		t1.start();
		t2.start();
		t3.start();

	}
}
```

执行结果：

```
t3线程进行初始化操作...
t1线程等待其他线程处理完成...
t2线程进行初始化操作...
t2线程初始化完毕，通知t1线程继续...
t3线程初始化完毕，通知t1线程继续...
t1线程继续执行...
```



# 3、**Callable和Future**

​	Future模式非常适合在处理很耗时的业务逻辑时使用，可以有效的减小系统的响应时间，提高系统的吞吐量。

```java
package com.example.part_04_utils.demo003;

import java.util.concurrent.Callable;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.Future;
import java.util.concurrent.FutureTask;

public class UseFuture implements Callable<String> {
	private String para;

	public UseFuture(String para) {
		this.para = para;
	}

	/**
	 * 这里是真实的业务逻辑，其执行可能很慢
	 */
	@Override
	public String call() throws Exception {
		// 模拟执行耗时
		System.out.println("call:正在处理" + this.para);
		Thread.sleep(5000);
		String result = this.para + "处理完成";
		System.out.println("call:完成处理" + this.para);
		return result;
	}

	// 主控制函数
	public static void main(String[] args) throws Exception {
		// 构造FutureTask，并且传入需要真正进行业务逻辑处理的类,该类一定是实现了Callable接口的类
		FutureTask<String> future1 = new FutureTask<String>(new UseFuture("任务1"));
		FutureTask<String> future2 = new FutureTask<String>(new UseFuture("任务2"));
		// 创建一个固定线程的线程池且线程数为2
		ExecutorService executor = Executors.newFixedThreadPool(2);
		// submit和execute的区别： 第一点是submit可以传入实现Callable接口的实例对象， 第二点是submit方法有返回值
		Future f1 = executor.submit(future1); // 单独启动一个线程去执行的
		Future f2 = executor.submit(future2);
		System.out.println("Main:请求完毕");
		try {
			// 这里可以做额外的数据操作，也就是主程序执行其他业务逻辑
			System.out.println("Main:处理其他业务逻辑...");
			Thread.sleep(1000);
		} catch (Exception e) {
			e.printStackTrace();
		}
		System.out.println("Main:开始获取future1和future2的数据");
		// 调用获取数据方法,如果call()方法没有执行完成,则依然会进行等待
		System.out.println("Main成功获得future数据：future1-" + future1.get());
		System.out.println("Main成功获得future数据：future2-" + future2.get());

		executor.shutdown();
	}

}
```

执行结果：

```
Main:请求完毕
call:正在处理任务2
call:正在处理任务1
Main:处理其他业务逻辑...
Main:开始获取future1和future2的数据
call:完成处理任务2
call:完成处理任务1
Main成功获得future数据：future1-任务1处理完成
Main成功获得future数据：future2-任务2处理完成
```



# 4、**Semaphore（信号量）**

​	可以控制系统的流量：拿到信号量的线程可以进入，否则就等待。通过acquire()和release()获取和释放访问许可。

```java
package com.example.part_04_utils.demo004;

import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.Semaphore;

public class UseSemaphore {

	public static void main(String[] args) {
		// 线程池
		ExecutorService exec = Executors.newCachedThreadPool();
		// 只能5个线程同时访问
		final Semaphore semp = new Semaphore(5);
		// 模拟20个客户端访问
		for (int index = 0; index < 20; index++) {
			final int no = index;
			Runnable run = new Runnable() {
				public void run() {
					try {
						// 获取许可
						semp.acquire();
						System.out.println("Accessing: " + no);
						// 模拟实际业务逻辑
						Thread.sleep((long) (Math.random() * 1000));
						// 访问完后，释放
						semp.release();
					} catch (InterruptedException e) {
					}
				}
			};
			exec.execute(run);
		}

		try {
			Thread.sleep(10);
		} catch (InterruptedException e) {
			e.printStackTrace();
		}

		System.out.println("加入到队列中等待的任务个数 " + semp.getQueueLength()); // 加入到队列中等待的任务个数

		try {
			Thread.sleep(1000);
		} catch (InterruptedException e) {
			e.printStackTrace();
		}

		System.out.println("加入到队列中等待的任务个数 " + semp.getQueueLength()); // 加入到队列中等待的任务个数

		try {
			Thread.sleep(1000);
		} catch (InterruptedException e) {
			e.printStackTrace();
		}

		System.out.println("加入到队列中等待的任务个数 " + semp.getQueueLength()); // 加入到队列中等待的任务个数

		// 退出线程池
		exec.shutdown();
	}

}
```

执行结果：

```
Accessing: 1
Accessing: 4
Accessing: 0
Accessing: 3
Accessing: 2
加入到队列中等待的任务个数 15
Accessing: 6
Accessing: 7
Accessing: 5
Accessing: 10
Accessing: 11
Accessing: 8
Accessing: 9
Accessing: 12
Accessing: 13
Accessing: 14
加入到队列中等待的任务个数 5
Accessing: 15
Accessing: 16
Accessing: 17
Accessing: 18
Accessing: 19
加入到队列中等待的任务个数 0
```

