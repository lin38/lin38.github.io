---
title: java并发编程-1-线程安全&synchronized&volatile
date: 2018-07-27 23:37:55
categories: java并发编程
tags: 
  - synchronized
  - volatile
toc: true
list_number: false
---

# 1、线程安全

## 1.1 线程安全的概念

​	当多个线程访问某一个类（对象或方法）时，这个类始终都能表现出正确的行为，那么这个类（对象或方法）就是线程安全的。 

## 1.2 synchronized

​	可以在任意对象及方法上加锁，而加锁的这段代码称为“互斥区”或“临界区”。

<!--more-->

<center>示例1：</center>

```java
package com.example.part_01_synchronized.demo001;

/**
 * 线程安全概念：当多个线程访问某一个类（对象或方法）时，这个对象始终都能表现出正确的行为，那么这个类（对象或方法）就是线程安全的。
 * synchronized：可以在任意对象及方法上加锁，而加锁的这段代码称为"互斥区"或"临界区"
 */
public class MyThread extends Thread {

	private int count = 5;

	// synchronized加锁
	public void run() {
		count--;
		System.out.println(currentThread().getName() + " count = " + count);
	}

	public static void main(String[] args) {
		/**
		 * 分析：当多个线程访问myThread的run方法时，以排队的方式进行处理（这里排队是按照CPU分配的先后顺序而定的），
		 * 一个线程想要执行synchronized修饰的方法里的代码： 1 尝试获得锁 2 如果拿到锁，执行synchronized代码体内容；
		 * 拿不到锁，这个线程就会不断的尝试获得这把锁，直到拿到为止，
		 * 而且是多个线程同时去竞争这把锁。（也就是会有锁竞争的问题）
		 */
		MyThread myThread = new MyThread();
		Thread t1 = new Thread(myThread, "t1");
		Thread t2 = new Thread(myThread, "t2");
		Thread t3 = new Thread(myThread, "t3");
		Thread t4 = new Thread(myThread, "t4");
		Thread t5 = new Thread(myThread, "t5");
		t1.start();
		t2.start();
		t3.start();
		t4.start();
		t5.start();
	}
}
```

执行结果：

- 无synchronized修饰（输出结果随机）： 

  ```
  t1 count = 3
  t5 count = 0
  t4 count = 1
  t3 count = 2
  t2 count = 3
  ```

- 有synchronized修饰： 

  ```
  t1 count = 4
  t5 count = 3
  t4 count = 2
  t2 count = 1
  t3 count = 0
  ```

<center>示例2：</center>

```java
package com.example.part_01_synchronized.demo002;

/**
 * 关键字synchronized取得的锁都是对象锁，而不是把一段代码（方法）当做锁，
 * 所以代码中哪个线程先执行synchronized关键字的方法，哪个线程就持有该方法所属对象的锁（Lock）。
 * 在静态方法上加synchronized关键字，表示锁定.class类，类一级别的锁（独占.class类）。
 */
public class MultiThread {

	private static int num = 0;

	/** static */
	public synchronized void printNum(String tag) {
		try {

			if (tag.equals("a")) {
				num = 100;
				System.out.println("tag a, set num over!");
				Thread.sleep(1000);
			} else {
				num = 200;
				System.out.println("tag b, set num over!");
			}

			System.out.println("tag " + tag + ", num = " + num);

		} catch (InterruptedException e) {
			e.printStackTrace();
		}
	}

	// 注意观察run方法输出顺序
	public static void main(String[] args) {

		// 两个不同的对象
		final MultiThread m1 = new MultiThread();
		final MultiThread m2 = new MultiThread();

		Thread t1 = new Thread(new Runnable() {
			@Override
			public void run() {
				m1.printNum("a");
			}
		});

		Thread t2 = new Thread(new Runnable() {
			@Override
			public void run() {
				m2.printNum("b");
			}
		});

		t1.start();
		t2.start();

	}

}
```

  执行结果：

- 无static：

  ```
  tag a, set num over!
  tag b, set num over!
  tag b, num = 200
  tag a, num = 200
  ```

- 有static：

  ```
  tag a, set num over!
  tag a, num = 100
  tag b, set num over!
  tag b, num = 200
  ```



# 2、对象锁的同步和异步

## 2.1 同步：synchronized

​    同步的概念就是共享，如果不是共享的资源，就没有必要进行同步。 

## 2.2 异步：asynchronized

​	异步的概念就是独立，相互之间不受到任何约束。 

​	同步的目的就是为了线程安全，其实对于线程安全来说，需要满足两个特性：原子性（同步）、可见性。 

<center>示例：</center>

```java
package com.example.part_01_synchronized.demo003;

/**
 * 对象锁的同步和异步问题
 */
public class MyObject {

	public synchronized void method1() {
		try {
			System.out.println(Thread.currentThread().getName());
			Thread.sleep(4000);
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
	}

	/** synchronized */
	public void method2() {
		System.out.println(Thread.currentThread().getName());
	}

	public static void main(String[] args) {

		final MyObject mo = new MyObject();

		/**
		 * 分析： t1线程先持有object对象的Lock锁，t2线程可以以异步的方式调用对象中的非synchronized修饰的方法。
		 * t1线程先持有object对象的Lock锁，t2线程如果在这个时候调用对象中的同步（synchronized）方法则需等待，也就是同步。
		 */
		Thread t1 = new Thread(new Runnable() {
			@Override
			public void run() {
				mo.method1();
			}
		}, "t1");

		Thread t2 = new Thread(new Runnable() {
			@Override
			public void run() {
				mo.method2();
			}
		}, "t2");

		t1.start();
		t2.start();

	}

}
```

执行结果：

- 无synchronized： 

  ```
  t1
  t2
  等待4s
  ```

  

- 有synchronized： 

  ```
  t1
  等待4s
  t2
  ```



# 3、脏读

​	对于对象的同步和异步的方法，我们在设计自己的程序时，一定要考虑问题的整体，不然就会出现数据不一致的错误，很经典的错误就是脏读（dirtyread）。 

<center>示例：</center>

```java
package com.example.part_01_synchronized.demo004;

/**
 * 业务整体需要使用完整的synchronized，保持业务的原子性。
 */
public class DirtyRead {

	private String username = "bjsxt";
	private String password = "123";

	public synchronized void setValue(String username, String password) {
		this.username = username;

		try {
			Thread.sleep(2000);
		} catch (InterruptedException e) {
			e.printStackTrace();
		}

		this.password = password;

		System.out.println("setValue最终结果：username = " + username + " , password = " + password);
	}

	/* synchronized */
	public void getValue() {
		System.out.println("getValue方法得到：username = " + this.username + " , password = " + this.password);
	}

	public static void main(String[] args) throws Exception {

		/**
		 * 分析（可以理解成是两个事务）：
		 * 1、getValue()方法不加synchronized修饰时，t1线程更改dr对象的name值之后进入睡眠，
		 * 之后main线程通过非同步方法获取dr对象的值，此时出现脏读，因为t1线程还未执行完成，读取到了一部分t1对象写的数据。
		 * 2、getValue()方法加synchronized修饰时，t1线程更改dr对象的name值之后进入睡眠，
		 * 之后main线程通过同步方法获取dr对象的值，此时需要等待t1线程释放资源才能读取信息，因此不会出现脏读，
		 * 因为只有t1线程执行完成，main线程才能获得锁。
		 */
		
		final DirtyRead dr = new DirtyRead();
		Thread t1 = new Thread(new Runnable() {
			@Override
			public void run() {
				dr.setValue("z3", "456");
			}
		});
		t1.start();
		Thread.sleep(1000);

		dr.getValue();
	}

}
```

- 无synchronized： 

  ```
  getValue方法得到：username = z3 , password = 123
  setValue最终结果：username = z3 , password = 456
  ```

- 有synchronized：

  ```
  setValue最终结果：username = z3 , password = 456
  getValue方法得到：username = z3 , password = 456
  ```

  

# 4、**synchronized其他概念**

## 4.1 synchronized锁重入

​	关键字synchronized拥有锁重入的功能，也就是在使用synchronized时，当一个线程得到了一个对象的锁后，再次请求此对象时是可以再次得到该对象的锁的。 

<center>示例1：</center>

```java
package com.example.part_01_synchronized.demo005;

/**
 * synchronized的重入
 */
public class SyncDouble1 {

	public synchronized void method1() {
		System.out.println("method1..");
		method2();
	}

	public synchronized void method2() {
		System.out.println("method2..");
		method3();
	}

	public synchronized void method3() {
		System.out.println("method3..");
	}

	public static void main(String[] args) {
		final SyncDouble1 sd = new SyncDouble1();
		Thread t1 = new Thread(new Runnable() {
			@Override
			public void run() {
				sd.method1();
			}
		});
		t1.start();
	}
}
```

执行结果：

```
method1..
method2..
method3..
```

<center>示例2：</center>

```java
package com.example.part_01_synchronized.demo005;

/**
 * synchronized的重入
 */
public class SyncDouble2 {

	static class Main {
		public int i = 10;

		public synchronized void operationSup() {
			try {
				i--;
				System.out.println("Main print i = " + i);
				Thread.sleep(100);
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
		}
	}

	static class Sub extends Main {
		public synchronized void operationSub() {
			try {
				while (i > 0) {
					i--;
					System.out.println("Sub print i = " + i);
					Thread.sleep(100);
					this.operationSup();
				}
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
		}
	}

	public static void main(String[] args) {

		Thread t1 = new Thread(new Runnable() {
			@Override
			public void run() {
				Sub sub = new Sub();
				sub.operationSub();
			}
		});

		t1.start();
	}
}
```

执行结果：

```
Sub print i = 9
Main print i = 8
Sub print i = 7
Main print i = 6
Sub print i = 5
Main print i = 4
Sub print i = 3
Main print i = 2
Sub print i = 1
Main print i = 0
```

## 4.2 出现异常，锁自动释放

<center>示例：</center>

```java
package com.example.part_01_synchronized.demo006;

/**
 * synchronized异常
 */
public class SyncException {

	private int i = 0;

	public synchronized void operation() {
		while (true) {
			try {
				i++;
				Thread.sleep(100);
				System.out.println(Thread.currentThread().getName() + " , i = " + i);
				if (i == 20 || i == 30) {
					// Integer.parseInt("a");
					throw new RuntimeException();
				}
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
		}
	}

	public static void main(String[] args) {

		final SyncException se = new SyncException();
		Thread t1 = new Thread(new Runnable() {
			@Override
			public void run() {
				se.operation();
			}
		}, "t1");
		t1.start();
		Thread t2 = new Thread(new Runnable() {
			@Override
			public void run() {
				se.operation();
			}
		}, "t2");
		t2.start();
	}

}
```

执行结果：

```
t1 , i = 1
t1 , i = 2
t1 , i = 3
t1 , i = 4
t1 , i = 5
t1 , i = 6
t1 , i = 7
t1 , i = 8
t1 , i = 9
t1 , i = 10
t1 , i = 11
t1 , i = 12
t1 , i = 13
t1 , i = 14
t1 , i = 15
t1 , i = 16
t1 , i = 17
t1 , i = 18
t1 , i = 19
t1 , i = 20
Exception in thread "t1" java.lang.RuntimeException
	at com.example.part_01_synchronized.demo006.SyncException.operation(SyncException.java:18)
	at com.example.part_01_synchronized.demo006.SyncException$1.run(SyncException.java:32)
	at java.lang.Thread.run(Unknown Source)
t2 , i = 21
t2 , i = 22
t2 , i = 23
t2 , i = 24
t2 , i = 25
t2 , i = 26
t2 , i = 27
t2 , i = 28
t2 , i = 29
t2 , i = 30
Exception in thread "t2" java.lang.RuntimeException
	at com.example.part_01_synchronized.demo006.SyncException.operation(SyncException.java:18)
	at com.example.part_01_synchronized.demo006.SyncException$2.run(SyncException.java:39)
	at java.lang.Thread.run(Unknown Source)
```



# 5、**synchronized代码块**

## 5.1 同步方法的弊端

​	使用synchronized声明的方法在某些情况下是有弊端的，比如A线程调用同步的方法执行一个很长时间的任务，那么B线程就必须等待比较长的时间才能执行，这样的情况下可以使用synchronized代码块去优化代码执行时间，也就是通常所说的减小锁的粒度。 

<center>示例：</center>

```java
package com.example.part_01_synchronized.demo007;

/**
 * 使用synchronized代码块减小锁的粒度，提高性能
 */
public class Optimize {

	public void doLongTimeTask() {
		try {

			System.out.println("当前线程开始：" + Thread.currentThread().getName() + ", 正在执行一个较长时间的业务操作，其内容不需要同步");
			Thread.sleep(2000);

			synchronized (this) {
				System.out.println("当前线程：" + Thread.currentThread().getName() + ", 执行同步代码块，对其同步变量进行操作");
				Thread.sleep(1000);
			}
			System.out.println("当前线程结束：" + Thread.currentThread().getName() + ", 执行完毕");

		} catch (InterruptedException e) {
			e.printStackTrace();
		}
	}

	public static void main(String[] args) {
		final Optimize otz = new Optimize();
		Thread t1 = new Thread(new Runnable() {
			@Override
			public void run() {
				otz.doLongTimeTask();
			}
		}, "t1");
		Thread t2 = new Thread(new Runnable() {
			@Override
			public void run() {
				otz.doLongTimeTask();
			}
		}, "t2");
		t1.start();
		t2.start();

	}
}
```

执行结果：

```
当前线程开始：t2, 正在执行一个较长时间的业务操作，其内容不需要同步
当前线程开始：t1, 正在执行一个较长时间的业务操作，其内容不需要同步
当前线程：t2, 执行同步代码块，对其同步变量进行操作
当前线程：t1, 执行同步代码块，对其同步变量进行操作
当前线程结束：t2, 执行完毕
当前线程结束：t1, 执行完毕
```

## 5.2 可以对任意Object加锁

​	synchronized可以使用任意的Object进行加锁，用法比较灵活。 

<center>示例：</center>

```java
package com.example.part_01_synchronized.demo008;

/**
 * 使用synchronized代码块加锁,比较灵活
 */
public class ObjectLock {

	public void method1() {
		synchronized (this) { // 对象锁
			try {
				System.out.println("do method1..");
				Thread.sleep(2000);
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
		}
	}

	public void method2() { // 类锁
		synchronized (ObjectLock.class) {
			try {
				System.out.println("do method2..");
				Thread.sleep(2000);
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
		}
	}

	private Object lock = new Object();

	public void method3() { // 任何对象锁
		synchronized (lock) {
			try {
				System.out.println("do method3..");
				Thread.sleep(2000);
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
		}
	}

	public static void main(String[] args) {

		final ObjectLock objLock = new ObjectLock();
		Thread t1 = new Thread(new Runnable() {
			@Override
			public void run() {
				objLock.method1();
			}
		});
		Thread t2 = new Thread(new Runnable() {
			@Override
			public void run() {
				objLock.method2();
			}
		});
		Thread t3 = new Thread(new Runnable() {
			@Override
			public void run() {
				objLock.method3();
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
do method3..
do method1..
do method2..
```

## 5.3 不要使用String的常量加锁

<center>示例：</center>

```java
package com.example.part_01_synchronized.demo009;

/**
 * synchronized代码块对字符串的锁，注意String常量池的缓存功能
 */
public class StringLock {

	public void method() {
        /**
         * ps:两次执行method方法，s对象不是同一个对象，因此同步失效。可以根据内存地址进行判断两次不是同一对象。注意：String的hashCode()方法被重写过，不可信。
         */
		String s = new String("字符串常量");
		/**
		 * ps:注意常量池的问题：在程序编译期，编译程序先去字符串常量池检查，是否存在“字符串常量”，
		 * 如果不存在，则在常量池中开辟一个内存空间存放“字符串常量”；如果存在的话，则不用重新开辟空间。
		 * 如果存在，就在栈中开辟一块空间，命名为变量名，存放的值为常量池中“字符串常量”的内存地址。
		 * 所以两次是同一个对象，同步生效。
		 */
//		String s = "字符串常量";
		synchronized (s) {
			try {
				while (true) {
//					System.out.println(System.identityHashCode(s)); // 内存地址
					System.out.println("当前线程 : " + Thread.currentThread().getName() + "开始");
					Thread.sleep(1000);
					System.out.println("当前线程 : " + Thread.currentThread().getName() + "结束");
//					break;
				}
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
		}
	}

	public static void main(String[] args) {
		final StringLock stringLock = new StringLock();
		Thread t1 = new Thread(new Runnable() {
			@Override
			public void run() {
				stringLock.method();
			}
		}, "t1");
		Thread t2 = new Thread(new Runnable() {
			@Override
			public void run() {
				stringLock.method();
			}
		}, "t2");

		t1.start();
		t2.start();
	}
}
```

执行结果：

- String s = new String("")的方式（加锁失败）： 

  ```
  当前线程 : t2开始
  当前线程 : t1开始
  当前线程 : t1结束
  当前线程 : t2结束
  当前线程 : t1开始
  当前线程 : t2开始
  ...
  ```

- String s = ""的方式（加锁成功）： 

  ```
  当前线程 : t1开始
  当前线程 : t1结束
  当前线程 : t1开始
  ```

## 5.4 锁对象的改变问题

​	当使用一个对象进行加锁的时候，要注意对象本身发生改变的时候，那么持有的锁就不同步。如果对象本身不发生改变，那么依然是同步的，即使是对象的属性发生了改变。 

<center>示例1：</center>

```java
package com.example.part_01_synchronized.demo010;

/**
 * 锁对象的改变问题
 */
public class ChangeLock {

	private String lock = "lock";

	private void method() {
		synchronized (lock) {
			try {
				System.out.println("当前线程 : " + Thread.currentThread().getName() + "开始");
				lock = "change lock";
				Thread.sleep(2000);
				System.out.println("当前线程 : " + Thread.currentThread().getName() + "结束");
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
		}
	}

	public static void main(String[] args) {

		final ChangeLock changeLock = new ChangeLock();
		Thread t1 = new Thread(new Runnable() {
			@Override
			public void run() {
				changeLock.method();
			}
		}, "t1");
		Thread t2 = new Thread(new Runnable() {
			@Override
			public void run() {
				changeLock.method();
			}
		}, "t2");
		t1.start();
		try {
			Thread.sleep(100);
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
		t2.start();
	}

}
```

执行结果：

```
当前线程 : t1开始
当前线程 : t2开始
当前线程 : t1结束
当前线程 : t2结束
```

<center>示例2：</center>

```java
package com.example.part_01_synchronized.demo010;

/**
 * 同一对象属性的修改不会影响锁的情况
 */
public class ModifyLock {

	private String name;
	private int age;

	public String getName() {
		return name;
	}

	public void setName(String name) {
		this.name = name;
	}

	public int getAge() {
		return age;
	}

	public void setAge(int age) {
		this.age = age;
	}

	public synchronized void changeAttributte(String name, int age) {
		try {
			System.out.println("当前线程 : " + Thread.currentThread().getName() + " 开始");
			this.setName(name);
			this.setAge(age);

			System.out.println("当前线程 : " + Thread.currentThread().getName() + " 修改对象内容为： " + this.getName() + ", "
					+ this.getAge());

			Thread.sleep(2000);
			System.out.println("当前线程 : " + Thread.currentThread().getName() + " 结束");
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
	}

	public static void main(String[] args) {
		final ModifyLock modifyLock = new ModifyLock();
		Thread t1 = new Thread(new Runnable() {
			@Override
			public void run() {
				modifyLock.changeAttributte("张三", 20);
			}
		}, "t1");
		Thread t2 = new Thread(new Runnable() {
			@Override
			public void run() {
				modifyLock.changeAttributte("李四", 21);
			}
		}, "t2");

		t1.start();
		try {
			Thread.sleep(100);
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
		t2.start();
	}

}
```

执行结果：

```
当前线程 : t1 开始
当前线程 : t1 修改对象内容为： 张三, 20
当前线程 : t1 结束
当前线程 : t2 开始
当前线程 : t2 修改对象内容为： 李四, 21
当前线程 : t2 结束
```

## 5.5 死锁问题

<center>示例：</center>

```java
package com.example.part_01_synchronized.demo011;

/**
 * 死锁问题，在设计程序时就应该避免双方相互持有对方的锁的情况
 */
public class DeadLock implements Runnable {

	private String tag;
	private static Object lock1 = new Object();
	private static Object lock2 = new Object();

	public void setTag(String tag) {
		this.tag = tag;
	}

	@Override
	public void run() {
		if (tag.equals("a")) {
			synchronized (lock1) {
				try {
					System.out.println("当前线程 : " + Thread.currentThread().getName() + " 进入lock1执行");
					Thread.sleep(2000);
				} catch (InterruptedException e) {
					e.printStackTrace();
				}
				synchronized (lock2) {
					System.out.println("当前线程 : " + Thread.currentThread().getName() + " 进入lock2执行");
				}
			}
		}
		if (tag.equals("b")) {
			synchronized (lock2) {
				try {
					System.out.println("当前线程 : " + Thread.currentThread().getName() + " 进入lock2执行");
					Thread.sleep(2000);
				} catch (InterruptedException e) {
					e.printStackTrace();
				}
				synchronized (lock1) {
					System.out.println("当前线程 : " + Thread.currentThread().getName() + " 进入lock1执行");
				}
			}
		}
	}

	public static void main(String[] args) {

		DeadLock d1 = new DeadLock();
		d1.setTag("a");
		DeadLock d2 = new DeadLock();
		d2.setTag("b");

		Thread t1 = new Thread(d1, "t1");
		Thread t2 = new Thread(d2, "t2");

		t1.start();
		try {
			Thread.sleep(500);
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
		t2.start();
	}

}
```

执行结果：

```
当前线程 : t1 进入lock1执行
当前线程 : t2 进入lock2执行
...
```



# 6、**volatile关键字的概念**

​	volatile概念：volatile关键字的主要作用是使变量在多个线程间可见。 

​	在java中，每一个线程都有一块工作内存区，其中存放着所有线程共享的主内存中的变量值的拷贝。当线程执行时，它在自己的工作内存中操作这些变量。 

​	volatile的作用就是强制线程到主内存里去读取变量，而不是线程工作内存区里去读取，从而实现了多个线程间的变量可见，也就是满足线程安全的可见性。 

<center>示例：</center>

```java
package com.example.part_01_synchronized.demo012;

/**
 * 主线程改变了isRunning的值，使rt获知后终止
 */
public class RunThread extends Thread {

	// volatile
	private boolean isRunning = true;

	private void setRunning(boolean isRunning) {
		this.isRunning = isRunning;
	}

	public void run() {
		System.out.println("进入run方法..");
		while (isRunning == true) {
			// ..
		}
		System.out.println("线程停止");
	}

	public static void main(String[] args) throws InterruptedException {
		RunThread rt = new RunThread();
		rt.start();
		Thread.sleep(1000);
		rt.setRunning(false);
		System.out.println("isRunning的值已经被设置了false");
	}
}
```

执行结果：

- 无volatile：

  ```
  进入run方法..
  isRunning的值已经被设置了false
  ...
  ```

- 有volatile：

  ```
  进入run方法..
  isRunning的值已经被设置了false
  线程停止
  ```



# 7、**volatile关键字的非原子性**

​	volatile关键字虽然拥有多个线程之间的可见性，但是却不具备同步性（也就是原子性），可以算上是一个轻量级的synchronized，性能要比synchronized强很多，不会造成阻塞。 

​	这里需要注意：一般volatile用于只针对多个线程可见的变量操作，并不能代替synchronized的同步功能。 

​	要实现原子性，建议使用atomic类的系列对象，支持原子性操作。（注意：atomic类只保证本身方法的原子性，不能保证多次操作的原子性）。 

<center>示例1：</center>

```java
package com.example.part_01_synchronized.demo013;

import java.util.concurrent.atomic.AtomicInteger;

/**
 * volatile关键字不具备synchronized关键字的原子性（同步）
 */
public class VolatileNoAtomic extends Thread {
	private static volatile int count;

//      private static AtomicInteger count = new AtomicInteger(0);
	private static void addCount() {
		for (int i = 0; i < 1000; i++) {
			count++;
//                         count.incrementAndGet();
		}
		System.out.println(count);
	}

	public void run() {
		addCount();
	}

	public static void main(String[] args) {

		VolatileNoAtomic[] arr = new VolatileNoAtomic[10];
		for (int i = 0; i < 10; i++) {
			arr[i] = new VolatileNoAtomic();
		}

		for (int i = 0; i < 10; i++) {
			arr[i].start();
		}
	}

}
```

执行结果：

```
1000
3143
3486
2000
5486
4486
6486
7486
8486
9486
```

<center>示例2：</center>

```java
package com.example.part_01_synchronized.demo013;

import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.atomic.AtomicInteger;

public class AtomicUse {

	private static AtomicInteger count = new AtomicInteger(0);

	// 多个addAndGet在一个方法内是非原子性的，需要加synchronized进行修饰，保证4个addAndGet整体原子性
	/** synchronized */
	public int multiAdd() {
		try {
			Thread.sleep(100);
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
		count.addAndGet(1);
		count.addAndGet(2);
		count.addAndGet(3);
		count.addAndGet(4); // +10
		return count.get();
	}

	public static void main(String[] args) {

		final AtomicUse au = new AtomicUse();

		List<Thread> ts = new ArrayList<Thread>();
		for (int i = 0; i < 10; i++) {
			ts.add(new Thread(new Runnable() {
				@Override
				public void run() {
					System.out.println(au.multiAdd());
				}
			}));
		}

		for (Thread t : ts) {
			t.start();
		}

	}
}
```

执行结果：

- 无synchronized：

  ```
  10
  20
  30
  60
  50
  80
  40
  90
  100
  70
  ```

- 有synchronized：

  ```
  10
  20
  30
  40
  50
  60
  70
  80
  90
  100
  ```

  