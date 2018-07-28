---
title: java并发编程-3-同步类容器&并发类容器
date: 2018-07-28 17:28:18
categories: java并发编程
tags: 
  - ConcurrentHashMap
  - ArrayBlockingQueue
  - ConcurrentLinkedQueue
  - LinkedBlockingDeque
  - LinkedBlockingQueue
  - SynchronousQueue
  - PriorityBlockingQueue
  - DelayQueue
toc: true
list_number: false
---

# 1、同步类容器

​	同步类容器都是线程安全的，但在某些场景下可能需要加锁来保护复合操作。

​	复合操作如：迭代（反复访问元素，遍历完容器中所有的元素）、跳转（根据指定的顺序找到当前元素的下一个元素）、以及条件运算。

​	这些复合操作在多线程并发的修改容器时，可能会表现出意外的行为，最经典的便是ConcurrentModificationException，原因是当容器迭代的过程中，被并发的修改了内容，这是由于早期迭代器设计的时候没有考虑并发修改的问题。

​	同步类容器：如古老的Vector、HashTable。这些容器的同步功能，其实都是由JDK的Collections.synchronized***等工厂方法去创建实现的。其底层的机制无非就是传统的synchronized关键字对每个公用的方法都进行同步，使得每次只有一个线程访问容器的状态。这很明显不满足我们今天互联网时代高并发的需求，在保证线程安全的同时，也必须要有足够好的性能。



# 2、**并发类容器**

​	jdk5.0以后提供了多种并发类容器来替代同步类容器，从而改善性能。同步类容器的状态都是串行化的。他们虽然实现了线程安全，但是严重降低了并发性，在多线程环境时，严重降低了应用程序的吞吐量。

​	并发类容器是专门针对并发设计的，使用ConcurrentHashMap来代替传统的HashTable，而且在ConcurrentHashMap中，还添加了一些常用复合操作的支持。使用了CopyOnWriteArrayList代替Vector。

​	并发的Queue：ConcurrentLinkedQueue和LinkedBlockingQueue。前者是提高性能的队列，后者是阻塞形式的队列。具体实现的Queue还有很多，例如ArrayBlockingQueue、PriorityBlockingQueue、SynchronousQueue等。



# 3、**ConcurrentMap**

​	ConcurrentMap接口下有两个重要的实现：

- ConcurrentHashMap
- ConcurrentSkipListMap（支持并发排序功能，弥补ConcurrentHashMap）

​	ConcurrentHashMap内部使用段（Segment）来表示这些不同的部分，每个段其实就是一个小的HashTable，它们有自己的锁。只要多个修改操作发生在不同的段上，它们就可以并发进行。把一个整体分成了16个段，也就是最高支持16个线程的并发修改操作。这也是在多线程场景时减小锁的粒度从而降低锁竞争的一种方案。并且代码中大多共享变量使用volatile关键字声明，目的是第一时间获取修改的内容，性能非常好。

<center>示例：</center>

```java
package com.example.part_03_container.demo001;

import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

public class UseConcurrentMap {

	public static void main(String[] args) {
		ConcurrentHashMap<String, Object> chm = new ConcurrentHashMap<String, Object>();
		chm.put("k1", "v1");
		chm.put("k2", "v2");
		chm.put("k3", "v3");
		chm.putIfAbsent("k4", "vvvv");
		chm.putIfAbsent("k4", "vvvv2");
		System.out.println(chm.size());

		for (Map.Entry<String, Object> me : chm.entrySet()) {
			System.out.println("key:" + me.getKey() + ",value:" + me.getValue());
		}
	}
}
```

执行结果：

```
4
key:k1,value:v1
key:k2,value:v2
key:k3,value:v3
key:k4,value:vvvv
```



# 4、**Copy-On-Write容器**

​	Copy-On-Write简称COW，是一种用于程序设计中的优化策略。

​	JDK里的COW容器有两种：CopyOnWriteArrayList和CopyOnWriteArraySet。

​	什么是CopyOnWrite容器？CopyOnWrite容器即写时复制的容器。通俗的理解是当我们往一个容器添加元素的时候，不直接往当前容器添加，而是先将当前容器进行Copy，复制出一个新的容器，然后新的容器里添加元素，添加完元素之后，再将原容器的引用指向新的容器。这样做的好处是我们可以对CopyOnWrite容器进行并发的读，而不需要加锁，因为读操作不会添加任何元素到容器中。所以CopyOnWrite容器也是一种读写分离的思想，读和写不同的容器。



# 5、并发Queue

​	在并发队列上JDK提供了两套实现，一个是以ConcurrentLinkedQueue为代表的高性能队列，一个是以BlockingQueue接口为代表的阻塞队列，无论哪种都继承自Queue。

![](queue.png)



# 6、**ConcurrentLinkedQueue**

​	ConcurrentLinkedQueue是一个适用于高并发场景下的队列，通过无锁的方式，实现了高并发状态下的高性能，通常ConcurrentLinkedQueue性能好于BlockingQueue。它是一个基于链接节点的无界线程安全队列。该队列的元素遵循先进先出的原则。头是最先加入的，尾是最近加入的，该队列不允许null元素。

​	ConcurrentLinkedQueue重要方法：

- add()和offer()：都是加入元素的方法（在ConcurrentLinkedQueue中，这两个方法没有任何区别）

  从源码中可知，add()方法就是调用的offer()方法：

  ```java
      public boolean add(E e) {
          return offer(e);
      }
  ```

- poll()和peek()：都是取头元素节点，区别在于前者会删除元素，后者不会

<center>示例：</center>

```java
package com.example.part_03_container.demo002;

import java.util.concurrent.ConcurrentLinkedQueue;

public class UseConcurrentLinkedQueue {
	public static void main(String[] args) {
		// 高性能无阻塞无界队列：ConcurrentLinkedQueue
		ConcurrentLinkedQueue<String> q = new ConcurrentLinkedQueue<String>();
		// 在ConcurrentLinkedQueue中，可以认为offer()方法和add()这两个方法没有任何区别
		q.offer("a");
		q.offer("b");
		q.offer("c");
		q.offer("d");
		q.add("e");

		System.out.println(q.poll()); // a 从头部取出元素，并从队列里删除
		System.out.println(q.size()); // 4
		System.out.println(q.peek()); // b 从头部取出元素，不从队列里删除
		System.out.println(q.size()); // 4
	}
}
```

执行结果：

```
a
4
b
4
```



# 7、**BlockingQueue接口**

## 7.1 ArrayBlockingQueue

​	基于数组的阻塞队列实现，在ArrayBlockingQueue内部，维护了一个定长数组，以便缓存队列中的数据对象，其内部没实现读写分离，也就意味着生产和消费不能完全并行。长度是需要定义的。可以指定先进先出或者先进后出。也叫做有界队列。

<center>示例：</center>

```java
package com.example.part_03_container.demo002;

import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.TimeUnit;

public class UseArrayBlockingQueue {
	public static void main(String[] args) throws Exception {
		// 阻塞队列BlockingQueue
		ArrayBlockingQueue<String> array = new ArrayBlockingQueue<String>(5);
		// 在空间未满时，offer()、put()和add()都可以正常加入元素
		array.offer("a");
		array.put("b");
		array.add("c");
		array.add("d");
		array.add("e");
		System.out.println(array.size());
		// 当空间已满时，put()方法会阻塞住，直到有空间可以插入元素
//		array.put("f");
		// 当空间已满时，add()方法会直接抛出异常
//		array.add("f");
		// 当空间已满时，offer()方法不会抛出异常，而是返回插入的结果（true/false）
//		array.offer("f");
		// 可以为offer()方法设置超时时间，如果在超时时间之内未完成插入操作，则返回false，否则返回true，不抛出异常
//		System.out.println(array.offer("a", 3, TimeUnit.SECONDS));
	}
}
```

## 7.2 LinkedBlockingQueue

​	基于链表的阻塞队列，同ArrayBlockingQueue类似，其内部也维护着一个数据缓冲队列（该队列由一个链表构成），LinkedBlockingQueue之所以能够高效的处理并发数据，是因为其内部实现采用分离锁（读写分离两个锁），从而实现生产者和消费者操作的完全并行运行。它是一个无界队列（默认容量为Integer.MAX_VALUE，基本可认为无界）。

<center>示例：</center>

```java
package com.example.part_03_container.demo002;

import java.util.ArrayList;
import java.util.Iterator;
import java.util.List;
import java.util.concurrent.LinkedBlockingQueue;

public class UseLinkedBlockingQueue {
	public static void main(String[] args) {
		LinkedBlockingQueue<String> q = new LinkedBlockingQueue<String>();
		// offer()方法和add()方法的区别，也是在空间满了的时候，offer()不抛出异常，只返回操作结果，add()会抛出异常
		q.offer("a");
		q.offer("b");
		q.offer("c");
		q.offer("d");
		q.offer("e");
		q.add("f");
		System.out.println("q size:" + q.size());

		for (Iterator<String> iterator = q.iterator(); iterator.hasNext();) {
			System.out.println(iterator.next());
		}

		List<String> list = new ArrayList<String>();
		System.out.println("transferred number:" + q.drainTo(list, 3));
		System.out.println("list size:" + list.size());
		System.out.println("q size:" + q.size());
		for (String string : list) {
			System.out.println(string);
		}
		System.out.println("------------------");
		for (Iterator<String> iterator = q.iterator(); iterator.hasNext();) {
			System.out.println(iterator.next());
		}
	}
}
```

执行结果：

```
q size:6
a
b
c
d
e
f
transferred number:3
list size:3
q size:3
a
b
c
------------------
d
e
f
```

## 7.3 PriorityBlockingQueue

​	基于优先级的阻塞队列（优先级的判断通过构造函数传入的Comparator对象来决定，也就是说传入队列的对象必须实现Comparable接口），在实现PriorityBlockingQueue时，内部控制线程同步的锁采用的是公平锁，他是一个无界队列。

<center>示例：</center>

```java
package com.example.part_03_container.demo003;

public class Task implements Comparable<Task> {
	private int id;
	private String name;

	@Override
	public int compareTo(Task task) {
		return this.id > task.id ? 1 : this.id < task.id ? -1 : 0;
	}

	// 省略getter、setter

	@Override
	public String toString() {
		return "Task [id=" + id + ", name=" + name + "]";
	}
}
```

```java
package com.example.part_03_container.demo003;

import java.util.concurrent.PriorityBlockingQueue;

public class UsePriorityBlockingQueue {
	public static void main(String[] args) throws Exception {
		PriorityBlockingQueue<Task> q = new PriorityBlockingQueue<Task>();
		Task t1 = new Task();
		t1.setId(3);
		t1.setName("id为3");
		Task t2 = new Task();
		t2.setId(4);
		t2.setName("id为4");
		Task t3 = new Task();
		t3.setId(1);
		t3.setName("id为1");
		Task t4 = new Task();
		t4.setId(2);
		t4.setName("id为2");
		q.add(t1); // 3
		q.add(t2); // 4
		q.add(t3); // 1
		q.add(t4); // 2

		// 1 2 3 4
		System.out.println("容器：" + q);
		System.out.println(q.take());
		System.out.println("容器：" + q);
		System.out.println(q.take());
		System.out.println("容器：" + q);
		System.out.println(q.take());
		System.out.println("容器：" + q);
		System.out.println(q.take());
		System.out.println("容器：" + q);
	}
}
```

执行结果：

```
容器：[Task [id=1, name=id为1], Task [id=2, name=id为2], Task [id=3, name=id为3], Task [id=4, name=id为4]]
Task [id=1, name=id为1]
容器：[Task [id=2, name=id为2], Task [id=4, name=id为4], Task [id=3, name=id为3]]
Task [id=2, name=id为2]
容器：[Task [id=3, name=id为3], Task [id=4, name=id为4]]
Task [id=3, name=id为3]
容器：[Task [id=4, name=id为4]]
Task [id=4, name=id为4]
容器：[]
```

## 7.4 DelayQueue

​	带有延迟时间的Queue，其中的元素只有当其指定的延迟时间到了，才能从列队中获取到该元素。DelayQueue中的元素必须实现Delayed接口，DelayQueue是一个没有大小限制的队列，应用场景很多，比如缓存超时的数据进行移除、任务超时处理、空闲连接的关闭等。

<center>示例：</center>

```java
package com.example.part_03_container.demo004;

import java.util.concurrent.Delayed;
import java.util.concurrent.TimeUnit;

public class Wangmin implements Delayed {
	private String name;
	// 身份证
	private String id;
	// 截止时间
	private long endTime;
	// 定义时间工具类
	private TimeUnit timeUnit = TimeUnit.SECONDS;

	public Wangmin(String name, String id, long endTime) {
		this.name = name;
		this.id = id;
		this.endTime = endTime;
	}

	public String getName() {
		return this.name;
	}

	public String getId() {
		return this.id;
	}

	/**
	 * 用来判断是否到了截止时间
	 */
	@Override
	public long getDelay(TimeUnit unit) {
		return endTime - System.currentTimeMillis();
	}

	/**
	 * 相互比较排序
	 */
	@Override
	public int compareTo(Delayed delayed) {
		Wangmin w = (Wangmin) delayed;
		return this.getDelay(this.timeUnit) - w.getDelay(this.timeUnit) > 0 ? 1
				: this.getDelay(this.timeUnit) - w.getDelay(this.timeUnit) < 0 ? -1 : 0;
	}
}
```

```
package com.example.part_03_container.demo004;

import java.util.concurrent.DelayQueue;

public class WangBa implements Runnable {
	private DelayQueue<Wangmin> queue = new DelayQueue<Wangmin>();
	public boolean yingye = true; // 是否营业

	public void shangji(String name, String id, int money) {
		Wangmin man = new Wangmin(name, id, 1000 * money + System.currentTimeMillis());
		System.out.println("网民" + man.getName() + " 身份证" + man.getId() + "交钱" + money + "块,开始上机...");
		this.queue.add(man);
	}

	public void xiaji(Wangmin man) {
		System.out.println("网民" + man.getName() + " 身份证" + man.getId() + "时间到下机...");
	}

	@Override
	public void run() {
		while (yingye) {
			try {
				Wangmin man = queue.take();
				xiaji(man);
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
		}
	}

	public static void main(String args[]) {
		try {
			System.out.println("网吧开始营业");
			WangBa wangba = new WangBa();
			Thread shangwang = new Thread(wangba);
			shangwang.start();
			wangba.shangji("路人甲", "123", 1);
			wangba.shangji("路人乙", "234", 10);
			wangba.shangji("路人丙", "345", 5);
		} catch (Exception e) {
			e.printStackTrace();
		}
	}
}
```

执行结果：

```
网吧开始营业
网民路人甲 身份证123交钱1块,开始上机...
网民路人乙 身份证234交钱10块,开始上机...
网民路人丙 身份证345交钱5块,开始上机...
网民路人甲 身份证123时间到下机...
网民路人丙 身份证345时间到下机...
网民路人乙 身份证234时间到下机...
```

## 7.5、SynchronousQueue

​	一个没有缓冲的队列，生产者产生的数据直接会被消费者获取并消费。

<center>示例：</center>

```java
package com.example.part_03_container.demo002;

import java.util.concurrent.SynchronousQueue;

public class UseSynchronousQueue {
	public static void main(String[] args) {
		// 必须先有一个take等待着，再有一个add，否则会出现Queue full错误（可将t1删除进行试验）
		final SynchronousQueue<String> q = new SynchronousQueue<String>();
		
		Thread t1 = new Thread(new Runnable() {
			@Override
			public void run() {
				try {
					System.out.println(q.take());
				} catch (InterruptedException e) {
					e.printStackTrace();
				}
			}
		});
		t1.start();
		
		Thread t2 = new Thread(new Runnable() {

			@Override
			public void run() {
				q.add("asdasd");
			}
		});
		t2.start();
	}
}
```

## 7.6 LinkedBlockingDeque

​	双向队列，可以从头/尾加入或取出元素。

<center>示例：</center>

```java
package com.example.part_03_container.demo002;

import java.util.concurrent.LinkedBlockingDeque;

/**
 * LinkedBlockingDeque
 */
public class UseDeque {

	public static void main(String[] args) {

		LinkedBlockingDeque<String> dq = new LinkedBlockingDeque<String>(10);
		dq.addFirst("a");
		dq.addFirst("b");
		dq.addFirst("c");
		dq.addFirst("d");
		dq.addFirst("e");
		dq.addLast("f");
		dq.addLast("g");
		dq.addLast("h");
		dq.addLast("i");
		dq.addLast("j");
		// dq.addLast("k"); // 已经超过10个范围，会抛出异常：Deque full
		boolean result = dq.offerFirst("k"); // 如果有空间可插入则插入成功，否则失败，不会抛出异常
		System.out.println(result);
		System.out.println("查看头元素：" + dq.peekFirst()); // peek不会删除元素
		System.out.println("获取尾元素：" + dq.pollLast()); // poll会删除元素
		Object[] objs = dq.toArray();
		for (int i = 0; i < objs.length; i++) {
			System.out.println(objs[i]);
		}

	}
}
```

执行结果：

```
false
查看头元素：e
获取尾元素：j
e
d
c
b
a
f
g
h
i
```

