---
title: java并发编程-7-锁
date: 2018-07-30 14:29:43
categories: java并发编程
tags: 
  - 锁
  - ReentrantLock
  - Condition
  - ReentrantReadWriteLock
toc: true
list_number: false
---

# 1、锁

​	在java多线程中，我们可以使用synchronized关键字来实现线程间的同步互斥工作，其实还有一个更优秀的机制去完成这个“同步互斥”工作。它就是Lock对象。他们具有比synchronized更为强大的功能，并且有嗅探锁定、多路分支等功能。



# 2、**ReentrantLock（重入锁）**

​	在需要进行同步的代码部分加上锁定，但不要忘记最后一定要释放锁定，不然会造成锁永远无法释放，其他线程永远进不来的结果。

<center>示例：</center>

```java
package com.example.part_06_lock.demo001;

import java.util.concurrent.locks.ReentrantLock;

public class UseReentrantLock {

	private ReentrantLock lock = new ReentrantLock();

	public void method1() {
		try {
			lock.lock();
			System.out.println("当前线程:" + Thread.currentThread().getName() + "进入method1..");
			Thread.sleep(1000);
			System.out.println("当前线程:" + Thread.currentThread().getName() + "退出method1..");
			Thread.sleep(1000);
			method2();
		} catch (InterruptedException e) {
			e.printStackTrace();
		} finally {
			lock.unlock();
		}
	}

	public void method2() {
		try {
			lock.lock();
			System.out.println("当前线程:" + Thread.currentThread().getName() + "进入method2..");
			Thread.sleep(2000);
			System.out.println("当前线程:" + Thread.currentThread().getName() + "退出method2..");
			Thread.sleep(1000);
		} catch (InterruptedException e) {
			e.printStackTrace();
		} finally {
			lock.unlock();
		}
	}

	public static void main(String[] args) {

		final UseReentrantLock ur = new UseReentrantLock();
		Thread t1 = new Thread(new Runnable() {
			@Override
			public void run() {
				ur.method1();
			}
		}, "t1");
		Thread t2 = new Thread(new Runnable() {
			@Override
			public void run() {
				ur.method2();
			}
		}, "t2");

		t1.start();
		t2.start();
		try {
			Thread.sleep(10);
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
		System.out.println("正在队列中等待的线程个数:" + ur.lock.getQueueLength()); // 正在队列中等待的线程个数
	}
}
```

<!--more-->

执行结果：

```
当前线程:t1进入method1..
正在队列中等待的线程个数:1
当前线程:t1退出method1..
当前线程:t1进入method2..
当前线程:t1退出method2..
当前线程:t2进入method2..
当前线程:t2退出method2..
```



# 3、**锁与等待/通知**

​	我们在使用synchronized的时候，如果需要多线程间进行协作工作，则需要Object的wait()和notify()/notifyAll()方法进行配合工作。

​	那么同样，我们在使用Lock的时候，可以使用一个新的等待/通知的类，它就是Condition。这个Condition一定是针对具体某一把锁的，也就是只有锁的基础之上才会产生Condition。

<center>示例：</center>

```java
package com.example.part_06_lock.demo002;

import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

public class UseCondition {

	private Lock lock = new ReentrantLock();
	private Condition condition = lock.newCondition();

	public void method1() {
		try {
			lock.lock();
			System.out.println("当前线程：" + Thread.currentThread().getName() + "进入..");
			Thread.sleep(3000);
			System.out.println("当前线程：" + Thread.currentThread().getName() + "等待通知，释放锁..");
			condition.await(); // 相当于Object.wait
			System.out.println("当前线程：" + Thread.currentThread().getName() + "继续执行...");
		} catch (Exception e) {
			e.printStackTrace();
		} finally {
			lock.unlock();
		}
	}

	public void method2() {
		try {
			lock.lock();
			System.out.println("当前线程：" + Thread.currentThread().getName() + "进入..");
			Thread.sleep(3000);
			System.out.println("当前线程：" + Thread.currentThread().getName() + "发出唤醒..");
			condition.signal(); // 相当于Object.notify
			System.out.println("当前线程：" + Thread.currentThread().getName() + "释放锁..");
		} catch (Exception e) {
			e.printStackTrace();
		} finally {
			lock.unlock();
		}
	}

	public static void main(String[] args) throws InterruptedException {

		final UseCondition uc = new UseCondition();
		Thread t1 = new Thread(new Runnable() {
			@Override
			public void run() {
				uc.method1();
			}
		}, "t1");
		Thread t2 = new Thread(new Runnable() {
			@Override
			public void run() {
				uc.method2();
			}
		}, "t2");
		t1.start();
		Thread.sleep(200);
		t2.start();
	}

}
```

执行结果：

```
当前线程：t1进入..
当前线程：t1等待通知，释放锁..
当前线程：t2进入..
当前线程：t2发出唤醒..
当前线程：t2释放锁..
当前线程：t1继续执行...
```



# 4、**多Condition**

​	我们可以通过一个Lock对象产生多个Condition进行多线程间的交互，非常的灵活。可以使得部分需要唤醒的线程唤醒，其他线程则继续等待通知。

<center>示例：</center>

```java
package com.example.part_06_lock.demo003;

import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.ReentrantLock;

public class UseManyCondition {

	private ReentrantLock lock = new ReentrantLock();
	private Condition c1 = lock.newCondition();
	private Condition c2 = lock.newCondition();

	public void m1() {
		try {
			lock.lock();
			System.out.println("当前线程：" + Thread.currentThread().getName() + "进入方法m1等待c1..");
			c1.await();
			System.out.println("当前线程：" + Thread.currentThread().getName() + "方法m1继续..");
		} catch (Exception e) {
			e.printStackTrace();
		} finally {
			lock.unlock();
		}
	}

	public void m2() {
		try {
			lock.lock();
			System.out.println("当前线程：" + Thread.currentThread().getName() + "进入方法m2等待c1..");
			c1.await();
			System.out.println("当前线程：" + Thread.currentThread().getName() + "方法m2继续..");
		} catch (Exception e) {
			e.printStackTrace();
		} finally {
			lock.unlock();
		}
	}

	public void m3() {
		try {
			lock.lock();
			System.out.println("当前线程：" + Thread.currentThread().getName() + "进入方法m3等待c2..");
			c2.await();
			System.out.println("当前线程：" + Thread.currentThread().getName() + "方法m3继续..");
		} catch (Exception e) {
			e.printStackTrace();
		} finally {
			lock.unlock();
		}
	}

	public void m4() {
		try {
			lock.lock();
			System.out.println("当前线程：" + Thread.currentThread().getName() + "唤醒c1..");
			c1.signalAll();
		} catch (Exception e) {
			e.printStackTrace();
		} finally {
			lock.unlock();
		}
	}

	public void m5() {
		try {
			lock.lock();
			System.out.println("当前线程：" + Thread.currentThread().getName() + "唤醒c2..");
			c2.signal();
		} catch (Exception e) {
			e.printStackTrace();
		} finally {
			lock.unlock();
		}
	}

	public static void main(String[] args) {
		final UseManyCondition umc = new UseManyCondition();
		Thread t1 = new Thread(new Runnable() {
			@Override
			public void run() {
				umc.m1();
			}
		}, "t1");
		Thread t2 = new Thread(new Runnable() {
			@Override
			public void run() {
				umc.m2();
			}
		}, "t2");
		Thread t3 = new Thread(new Runnable() {
			@Override
			public void run() {
				umc.m3();
			}
		}, "t3");
		Thread t4 = new Thread(new Runnable() {
			@Override
			public void run() {
				umc.m4();
			}
		}, "t4");
		Thread t5 = new Thread(new Runnable() {
			@Override
			public void run() {
				umc.m5();
			}
		}, "t5");

		t1.start(); // c1
		t2.start(); // c1
		t3.start(); // c2

		try {
			Thread.sleep(2000);
		} catch (InterruptedException e) {
			e.printStackTrace();
		}

		t4.start(); // c1
		try {
			Thread.sleep(2000);
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
		t5.start(); // c2

	}

}
```

执行结果：

```
当前线程：t1进入方法m1等待c1..
当前线程：t3进入方法m3等待c2..
当前线程：t2进入方法m2等待c1..
当前线程：t4唤醒c1..
当前线程：t1方法m1继续..
当前线程：t2方法m2继续..
当前线程：t5唤醒c2..
当前线程：t3方法m3继续..
```



# 5、Lock/Condition其他方法和用法

1. 公平锁和非公平锁：`public ReentrantLock(boolean fair) {...}`

2. `public boolean tryLock() {...}`：尝试获得锁，获得结果用true/false返回

3. `public boolean tryLock(long timeout, TimeUnit unit) throws InterruptedException {...}`：在给定时间内尝试获得锁，获得结果用true/false返回

4. `public final boolean isFair() {...}`：是否是公平锁

5. `public boolean isLocked() {...}`：是否锁定

6. `public int getHoldCount() {...}`：查询当前线程持有此锁的个数，也就是调用lock()的次数

   <center>示例：</center>

   ```java
   package com.example.part_06_lock.demo005;
   
   import java.util.concurrent.locks.ReentrantLock;
   
   /**
    * lock.getHoldCount()方法：只能在当前调用线程内部使用，不能再其他线程中使用
    */
   public class TestHoldCount {
   
   	// 重入锁
   	private ReentrantLock lock = new ReentrantLock();
   
   	public void m1() {
   		try {
   			lock.lock();
   			System.out.println("进入m1方法，holdCount数为：" + lock.getHoldCount());
   
   			// 调用m2方法
   			m2();
   
   		} catch (Exception e) {
   			e.printStackTrace();
   		} finally {
   			lock.unlock();
   		}
   	}
   
   	public void m2() {
   		try {
   			lock.lock();
   			System.out.println("进入m2方法，holdCount数为：" + lock.getHoldCount());
   		} catch (Exception e) {
   			e.printStackTrace();
   		} finally {
   			lock.unlock();
   		}
   	}
   
   	public static void main(String[] args) {
   		TestHoldCount thc = new TestHoldCount();
   		thc.m1();
   	}
   }
   ```

   执行结果：

   ```
   进入m1方法，holdCount数为：1
   进入m2方法，holdCount数为：2
   ```

7. `public void lockInterruptibly() throws InterruptedException {...}`：优先响应中断的锁

   - lock()与lockInterruptibly()的区别：

     lock()优先考虑获取锁，待获取锁成功后，才响应中断。

     lockInterruptibly()优先考虑响应中断，而不是响应锁的普通获取或重入获取。

     ReentrantLock.lockInterruptibly()允许在等待时由其它线程调用等待线程的Thread.interrupt()方法来中断等待线程的等待而直接返回，这时不用获取锁，而会抛出一个InterruptedException。 ReentrantLock.lock()方法不允许Thread.interrupt()中断,即使检测到Thread.isInterrupted，一样会继续尝试获取锁，失败则继续休眠。只是在最后获取锁成功后再把当前线程置为interrupted状态，然后再中断线程。 

     <center>示例：</center>

     ```java
     package com.example.part_06_lock.demo004;
     
     import java.util.concurrent.locks.Lock;
     import java.util.concurrent.locks.ReentrantLock;
     
     /**
      * lock()方法和lockInterruptibly()方法的区别：
      * 1、ReentrantLock.lockInterruptibly()允许在等待时由其它线程调用等待线程的Thread.interrupt()方法来中断等待线程的等待而直接返回，
      * 这时不用获取锁，而会抛出一个InterruptedException。 
      * 2、ReentrantLock.lock()方法不允许Thread.interrupt()中断，即使检测到Thread.isInterrupted，一样会继续尝试获取锁，失败则继续休眠。
      * 只是在最后获取锁成功后再把当前线程置为interrupted状态,然后再中断线程。
      */
     public class TestLockInterruptibly {
     
     	public static void main(String[] args) throws InterruptedException {
     
     		final Lock lock = new ReentrantLock();
     		lock.lock();
     		System.out.println("Main:获取锁成功");
     		Thread t1 = new Thread(new Runnable() {
     			@Override
     			public void run() {
     				System.out.println("t1:尝试获取锁");
     				
     				/*lock.lock();
     				System.out.println("t1:获取到锁，继续执行");
     				System.out.println("t1此时状态是否是Interrupted的？"+ Thread.currentThread().isInterrupted());*/
     				
     				/*try {
     					lock.lockInterruptibly();
     					System.out.println("t1:获取到锁，继续执行");
     				} catch (InterruptedException e) {
     					e.printStackTrace();
     					System.out.println("t1:被中断！");
     				}*/
     			}
     		});
     		t1.start();
     		Thread.sleep(1000);
     		t1.interrupt();
     		System.out.println("Main:发起中断");
     		Thread.sleep(1000);
     		System.out.println("Main:释放锁");
     		lock.unlock();
     	}
     }
     ```

     执行结果：

     - lock()：

       ```
       Main:获取锁成功
       t1:尝试获取锁
       Main:发起中断
       Main:释放锁
       t1:获取到锁，继续执行
       t1此时状态是否是Interrupted的？true
       ```

     - lockInterruptibly()：

       ```
       Main:获取锁成功
       t1:尝试获取锁
       Main:发起中断
       java.lang.InterruptedException
       t1:被中断！
       	at java.util.concurrent.locks.AbstractQueuedSynchronizer.doAcquireInterruptibly(AbstractQueuedSynchronizer.java:898)
       	at java.util.concurrent.locks.AbstractQueuedSynchronizer.acquireInterruptibly(AbstractQueuedSynchronizer.java:1222)
       	at java.util.concurrent.locks.ReentrantLock.lockInterruptibly(ReentrantLock.java:335)
       	at com.example.part_06_lock.demo004.TestLockInterruptibly$1.run(TestLockInterruptibly.java:30)
       	at java.lang.Thread.run(Thread.java:748)
       Main:释放锁
       ```

8. `public final int getQueueLength() {...}`：获取正在等待获取此锁的线程数

9. `public int getWaitQueueLength(Condition condition) {...}`：获取正在等待某个condition的线程数

10. `public final boolean hasQueuedThread(Thread thread) {...}`：查询指定的线程是否正在等待此锁

11. `public final boolean hasQueuedThreads() {...}`：查询是否有线程正在等待此锁

12. `public boolean hasWaiters(Condition condition) {...}`：查询是否有线程正在等待指定condition



# 6、**ReentrantReadWriteLock（读写锁）**

​	其核心就是实现读写分离的锁。在高并发访问下，尤其是读多写少的情况下，性能要远高于重入锁。

​	synchronized、ReentrantLock，在同一时间内只能有一个线程进行访问。读写锁则不同，其本质是分成两个锁，即读锁、写锁。在读锁下，多个线程可以并发进行访问，但是在写锁的时候，只能一个一个的顺序访问。

​	口诀：读读共享，写写互斥，读写互斥。

<center>示例：</center>

```java
package com.example.part_06_lock.demo006;

import java.util.concurrent.locks.ReentrantReadWriteLock;
import java.util.concurrent.locks.ReentrantReadWriteLock.ReadLock;
import java.util.concurrent.locks.ReentrantReadWriteLock.WriteLock;

public class UseReentrantReadWriteLock {

	private ReentrantReadWriteLock rwLock = new ReentrantReadWriteLock();
	private ReadLock readLock = rwLock.readLock();
	private WriteLock writeLock = rwLock.writeLock();

	public void read() {
		try {
			readLock.lock();
			System.out.println("读——当前线程:" + Thread.currentThread().getName() + "进入...");
			Thread.sleep(3000);
			System.out.println("读——当前线程:" + Thread.currentThread().getName() + "退出...");
		} catch (Exception e) {
			e.printStackTrace();
		} finally {
			readLock.unlock();
		}
	}

	public void write() {
		try {
			writeLock.lock();
			System.out.println("写——当前线程:" + Thread.currentThread().getName() + "进入...");
			Thread.sleep(3000);
			System.out.println("写——当前线程:" + Thread.currentThread().getName() + "退出...");
		} catch (Exception e) {
			e.printStackTrace();
		} finally {
			writeLock.unlock();
		}
	}

	public static void main(String[] args) {

		final UseReentrantReadWriteLock urrw = new UseReentrantReadWriteLock();

		Thread t1 = new Thread(new Runnable() {
			@Override
			public void run() {
				urrw.read();
			}
		}, "t1");
		Thread t2 = new Thread(new Runnable() {
			@Override
			public void run() {
				urrw.read();
			}
		}, "t2");
		Thread t3 = new Thread(new Runnable() {
			@Override
			public void run() {
				urrw.write();
			}
		}, "t3");
		Thread t4 = new Thread(new Runnable() {
			@Override
			public void run() {
				urrw.write();
			}
		}, "t4");

		t1.start(); // R
		t2.start(); // R

		// t1.start(); // R
		// t3.start(); // W

		// t3.start(); // W
		// t4.start(); // W

	}
}
```

