---
title: java并发编程-2-线程之间的通信
date: 2018-07-28 13:08:33
categories: java并发编程
tags: 
  - wait
  - notify
  - ThreadLocal
toc: true
list_number: false
---

# 1、线程通信概念

​	线程是操作系统中独立的个体，但这些个体如果不经过特殊的处理就不能成为一个整体。线程之间的通信就是成为整体的必用方式之一。当线程存在通信指挥，系统间的交互性会更强大，在提高CPU利用率的同时还会使开发人员对线程任务在处理的过程中进行有效的把控和监督。

​	使用wait/notify方法实现线程间的通信，这两个方法都是Object类的方法，换句话说java为所有的对象都提供了这两个方法：

1. wait和notify必须配合synchronized关键字使用

2. wait方法释放锁，notify方法不释放锁

<!--more-->

<center>示例1：</center>

```java
package com.example.part_02_wait_notify.demo001;

import java.util.ArrayList;
import java.util.List;

public class ListAdd1 {

	// 注意：volatile
	private volatile static List list = new ArrayList();

	public void add() {
		list.add("bjsxt");
	}

	public int size() {
		return list.size();
	}

	public static void main(String[] args) {

		final ListAdd1 list1 = new ListAdd1();

		Thread t1 = new Thread(new Runnable() {
			@Override
			public void run() {
				try {
					for (int i = 0; i < 10; i++) {
						list1.add();
						System.out.println("当前线程：" + Thread.currentThread().getName() + "添加了一个元素..");
						Thread.sleep(500);
					}
				} catch (InterruptedException e) {
					e.printStackTrace();
				}
			}
		}, "t1");

		Thread t2 = new Thread(new Runnable() {
			@Override
			public void run() {
				while (true) {
					if (list1.size() == 5) {
						System.out.println("当前线程收到通知：" + Thread.currentThread().getName() + " list size = 5 线程停止..");
						throw new RuntimeException();
					}
				}
			}
		}, "t2");

		t1.start();
		t2.start();
	}

}
```

执行结果：

```
当前线程：t1添加了一个元素..
当前线程：t1添加了一个元素..
当前线程：t1添加了一个元素..
当前线程：t1添加了一个元素..
当前线程：t1添加了一个元素..
Exception in thread "t2" 当前线程收到通知：t2 list size = 5 线程停止..
java.lang.RuntimeException
	at com.example.part_02_wait_notify.demo001.ListAdd1$2.run(ListAdd1.java:44)
	at java.lang.Thread.run(Thread.java:748)
当前线程：t1添加了一个元素..
当前线程：t1添加了一个元素..
当前线程：t1添加了一个元素..
当前线程：t1添加了一个元素..
当前线程：t1添加了一个元素..
```

<center>示例2：</center>

```java
package com.example.part_02_wait_notify.demo001;

import java.util.ArrayList;
import java.util.List;

/**
 * wait notfiy 方法，wait释放锁，notfiy不释放锁
 */
public class ListAdd2 {
	private volatile static List list = new ArrayList();

	public void add() {
		list.add("bjsxt");
	}

	public int size() {
		return list.size();
	}

	public static void main(String[] args) throws InterruptedException {

		final ListAdd2 list2 = new ListAdd2();

		// 1 实例化出来一个 lock
		// 当使用wait 和 notify 的时候 ， 一定要配合着synchronized关键字去使用
		final Object lock = new Object();

		Thread t1 = new Thread(new Runnable() {
			@Override
			public void run() {
				try {
					synchronized (lock) {
						for (int i = 0; i < 10; i++) {
							list2.add();
							System.out.println("当前线程：" + Thread.currentThread().getName() + "添加了一个元素..");
							Thread.sleep(500);
							if (list2.size() == 5) {
								System.out.println("已经发出通知..");
								lock.notify();
							}
						}
					}
				} catch (InterruptedException e) {
					e.printStackTrace();
				}

			}
		}, "t1");

		Thread t2 = new Thread(new Runnable() {
			@Override
			public void run() {
				synchronized (lock) {
					if (list2.size() != 5) {
						try {
							System.out.println("t2进入...");
							lock.wait();
						} catch (InterruptedException e) {
							e.printStackTrace();
						}
					}
					System.out.println("当前线程：" + Thread.currentThread().getName() + "收到通知线程停止..");
					throw new RuntimeException();
				}
			}
		}, "t2");

		t2.start();
		Thread.sleep(200);
		t1.start();

	}

}
```

执行结果：

```
t2进入...
当前线程：t1添加了一个元素..
当前线程：t1添加了一个元素..
当前线程：t1添加了一个元素..
当前线程：t1添加了一个元素..
当前线程：t1添加了一个元素..
已经发出通知..
当前线程：t1添加了一个元素..
当前线程：t1添加了一个元素..
当前线程：t1添加了一个元素..
当前线程：t1添加了一个元素..
当前线程：t1添加了一个元素..
Exception in thread "t2" 当前线程：t2收到通知线程停止..
java.lang.RuntimeException
	at com.example.part_02_wait_notify.demo001.ListAdd2$2.run(ListAdd2.java:63)
	at java.lang.Thread.run(Thread.java:748)
```



# 2、使用wait/notify模拟Queue

```
package com.example.part_02_wait_notify.demo002;

import java.util.LinkedList;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicInteger;

/**
 * 使用wait/notify模拟Queue
 */
public class MyQueue {

	// 1 需要一个承装元素的集合
	private LinkedList<Object> list = new LinkedList<Object>();

	// 2 需要一个计数器
	private AtomicInteger count = new AtomicInteger(0);

	// 3 需要制定上限和下限
	private final int minSize = 0;

	private final int maxSize;

	// 4 构造方法
	public MyQueue(int size) {
		this.maxSize = size;
	}

	// 5 初始化一个对象 用于加锁
	private final Object lock = new Object();

	// put(anObject):
	// 把anObject加到BlockingQueue里,如果BlockQueue没有空间,则调用此方法的线程被阻断，直到BlockingQueue里面有空间再继续.
	public void put(Object obj) {
		synchronized (lock) {
			while (count.get() == this.maxSize) {
				try {
					lock.wait();
				} catch (InterruptedException e) {
					e.printStackTrace();
				}
			}
			// 1 加入元素
			list.add(obj);
			// 2 计数器累加
			count.incrementAndGet();
			// 3 通知另外一个线程（唤醒）
			lock.notify();
			System.out.println("新加入的元素为:" + obj);
		}
	}

	// take:
	// 取走BlockingQueue里排在首位的对象,若BlockingQueue为空,阻断进入等待状态直到BlockingQueue有新的数据被加入.
	public Object take() {
		Object ret = null;
		synchronized (lock) {
			while (count.get() == this.minSize) {
				try {
					lock.wait();
				} catch (InterruptedException e) {
					e.printStackTrace();
				}
			}
			// 1 做移除元素操作
			ret = list.removeFirst();
			// 2 计数器递减
			count.decrementAndGet();
			// 3 唤醒另外一个线程
			lock.notify();
		}
		return ret;
	}

	public int getSize() {
		return this.count.get();
	}

	public static void main(String[] args) {

		final MyQueue mq = new MyQueue(5);
		mq.put("a");
		mq.put("b");
		mq.put("c");
		mq.put("d");
		mq.put("e");

		System.out.println("当前容器的长度:" + mq.getSize());

		Thread t1 = new Thread(new Runnable() {
			@Override
			public void run() {
				mq.put("f");
				mq.put("g");
			}
		}, "t1");

		t1.start();

		Thread t2 = new Thread(new Runnable() {
			@Override
			public void run() {
				Object o1 = mq.take();
				System.out.println("移除的元素为:" + o1);
				Object o2 = mq.take();
				System.out.println("移除的元素为:" + o2);
			}
		}, "t2");

		try {
			TimeUnit.SECONDS.sleep(2);
		} catch (InterruptedException e) {
			e.printStackTrace();
		}

		t2.start();

	}

}
```

执行结果：

```
新加入的元素为:a
新加入的元素为:b
新加入的元素为:c
新加入的元素为:d
新加入的元素为:e
当前容器的长度:5
移除的元素为:a
新加入的元素为:f
移除的元素为:b
新加入的元素为:g
```



# 3、ThreadLocal

​	ThreadLocal概念：线程局部变量，是一种多线程间并发访问变量的解决方案。与synchronized等加锁的方式不同，ThreadLocal完全不提供锁，而是使用以空间换时间的手段，为每一个线程提供变量的独立副本，以保障线程安全。

​	从性能上说，ThreadLocal不具有绝对的优势，在并发不是很高的时候，加锁的性能会更好，但作为一套与锁完全无关的线程安全解决方案，在高并发量或者竞争激烈的场景，使用ThreadLocal可以在一定程度上减小锁竞争。

```java
package com.example.part_02_wait_notify.demo003;

public class ConnThreadLocal {

	private static ThreadLocal<String> th = new ThreadLocal<String>();

	public void setTh(String value) {
		th.set(value);
	}

	public void getTh() {
		System.out.println(Thread.currentThread().getName() + ":" + th.get());
	}

	public static void main(String[] args) throws InterruptedException {

		final ConnThreadLocal ct = new ConnThreadLocal();
		Thread t1 = new Thread(new Runnable() {
			@Override
			public void run() {
				ct.setTh("张三");
				ct.getTh();
			}
		}, "t1");

		Thread t2 = new Thread(new Runnable() {
			@Override
			public void run() {
				try {
					Thread.sleep(1000);
					ct.getTh();
				} catch (InterruptedException e) {
					e.printStackTrace();
				}
			}
		}, "t2");

		t1.start();
		t2.start();
	}

}
```

执行结果：

```
t1:张三
t2:null
```



