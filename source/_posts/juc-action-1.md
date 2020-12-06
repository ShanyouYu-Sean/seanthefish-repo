---
layout: post
title: 并发案例之生产者消费者
date: 2020-11-25 13:00:00
tags: 
- 并发
categories:
- 并发
---

转自 https://my.oschina.net/hosee/blog/485121#OSC_h4_4 ， 用一个案例说清了大部分juc下工具的使用, 本文有删改。

## 生产者消费者问题

 生产者消费者问题是研究多线程程序时绕不开的经典问题之一，它描述是有一块缓冲区作为仓库，生产者可以将产品放入仓库，消费者则可以从仓库中取走产品。解决生产者/消费者问题的方法可分为两类：（1）采用某种机制保护生产者和消费者之间的同步；（2）在生产者和消费者之间建立一个管道。第一种方式有较高的效率，并且易于实现，代码的可控制性较好，属于常用的模式。第二种管道缓冲区不易控制，被传输数据对象不易于封装等，实用性不强。

同步问题核心在于：如何保证同一资源被多个线程并发访问时的完整性。常用的同步方法是采用信号或加锁机制，保证资源在任意时刻至多被一个线程访问。Java语言在多线程编程上实现了完全对象化，提供了对同步机制的良好支持。在Java中一共有五种方法支持同步，其中前四个是同步方法，一个是管道方法。

- wait() / notify()方法
- await() / signal()方法
- BlockingQueue阻塞队列方法
- Semaphore方法
- PipedInputStream / PipedOutputStream

## wait() / notify()方法

wait() / nofity()方法是基类Object的两个方法，也就意味着所有Java类都会拥有这两个方法，这样，我们就可以为任何对象实现同步机制。

wait()方法：当缓冲区已满/空时，生产者/消费者线程停止自己的执行，放弃锁，使自己处于等等状态，让其他线程执行。

notify()方法：当生产者/消费者向缓冲区放入/取出一个产品时，向其他等待的线程发出可执行的通知，同时放弃锁，使自己处于等待状态。

各起了4个生产者，4个消费者 ：

```java
package test;

public class Hosee
{
	private static Integer count = 0;
	private final Integer FULL = 10;
	private static String LOCK = "LOCK";

	class Producer implements Runnable
	{
		@Override
		public void run()
		{
			for (int i = 0; i < 10; i++)
			{
				try
				{
					Thread.sleep(3000);
				}
				catch (Exception e)
				{
					e.printStackTrace();
				}
				synchronized (LOCK)
				{
					while (count == FULL)
					{
						try
						{
							LOCK.wait();
						}
						catch (Exception e)
						{
							e.printStackTrace();
						}
					}
					count++;
					System.out.println(Thread.currentThread().getName()
							+ "生产者生产，目前总共有" + count);
					LOCK.notifyAll();
				}
			}
		}

	}

	class Consumer implements Runnable
	{

		@Override
		public void run()
		{
			for (int i = 0; i < 10; i++)
			{
				try
				{
					Thread.sleep(3000);
				}
				catch (InterruptedException e1)
				{
					e1.printStackTrace();
				}
				synchronized (LOCK)
				{
					while (count == 0)
					{
						try
						{
							LOCK.wait();
						}
						catch (Exception e)
						{
							// TODO: handle exception
							e.printStackTrace();
						}
					}
					count--;
					System.out.println(Thread.currentThread().getName()
							+ "消费者消费，目前总共有" + count);
					LOCK.notifyAll();
				}
			}

		}

	}

	public static void main(String[] args) throws Exception
	{
		Hosee hosee = new Hosee();
		new Thread(hosee.new Producer()).start();
		new Thread(hosee.new Consumer()).start();
		new Thread(hosee.new Producer()).start();
		new Thread(hosee.new Consumer()).start();

		new Thread(hosee.new Producer()).start();
		new Thread(hosee.new Consumer()).start();
		new Thread(hosee.new Producer()).start();
		new Thread(hosee.new Consumer()).start();
	}

}
```

(需要注意的是，用什么加锁就用什么notify和wait，实例中使用的是LOCK)

## await() / signal()方法

首先，我们先来看看await()/signal()与wait()/notify()的区别：

- wait()和notify()必须在synchronized的代码块中使用 因为只有在获取当前对象的锁时才能进行这两个操作 否则会报异常 而await()和signal()一般与Lock()配合使用。
- wait是Object的方法，而await只有部分类有，如Condition。
- await()/signal()和新引入的锁定机制Lock直接挂钩，具有更大的灵活性。

那么为什么有了synchronized还要提出Lock呢？

synchronized并不完美，它有一些功能性的限制 —— 它无法中断一个正在等候获得锁的线程，也无法通过投票得到锁，如果不想等下去，也就没法得到锁。同步还要求锁的释放只能在与获得锁所在的堆栈帧相同的堆栈帧中进行，多数情况下，这没问题（而且与异常处理交互得很好），但是，确实存在一些非块结构的锁定更合适的情况。

所以在确实需要一些 synchronized 所没有的特性的时候，比如时间锁等候、可中断锁等候、无块结构锁、多个条件变量或者锁投票使用ReentrantLock。

```java
package test;

import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

public class Hosee {
	private static Integer count = 0;
	private final Integer FULL = 10;
	final Lock lock = new ReentrantLock();
	final Condition NotFull = lock.newCondition();
	final Condition NotEmpty = lock.newCondition();

	class Producer implements Runnable {
		@Override
		public void run() {
			for (int i = 0; i < 10; i++) {
				try {
					Thread.sleep(3000);
				} catch (Exception e) {
					e.printStackTrace();
				}
				lock.lock();
				try {
					while (count == FULL) {
						try {
							NotFull.await();
						} catch (InterruptedException e) {
							// TODO Auto-generated catch block
							e.printStackTrace();
						}
					}
					count++;
					System.out.println(Thread.currentThread().getName()
							+ "生产者生产，目前总共有" + count);
					NotEmpty.signal();
				} finally {
					lock.unlock();
				}

			}
		}
	}

	class Consumer implements Runnable {

		@Override
		public void run() {
			for (int i = 0; i < 10; i++) {
				try {
					Thread.sleep(3000);
				} catch (InterruptedException e1) {
					e1.printStackTrace();
				}
				lock.lock();
				try {
					while (count == 0) {
						try {
							NotEmpty.await();
						} catch (Exception e) {
							// TODO: handle exception
							e.printStackTrace();
						}
					}
					count--;
					System.out.println(Thread.currentThread().getName()
							+ "消费者消费，目前总共有" + count);
					NotFull.signal();
				} finally {
					lock.unlock();
				}

			}

		}

	}

	public static void main(String[] args) throws Exception {
		Hosee hosee = new Hosee();
		new Thread(hosee.new Producer()).start();
		new Thread(hosee.new Consumer()).start();
		new Thread(hosee.new Producer()).start();
		new Thread(hosee.new Consumer()).start();

		new Thread(hosee.new Producer()).start();
		new Thread(hosee.new Consumer()).start();
		new Thread(hosee.new Producer()).start();
		new Thread(hosee.new Consumer()).start();
	}

}
```
运行结果与第一个类似。上述代码用了两个Condition，其实用一个也是可以的，只不过要signalall()。

## BlockingQueue阻塞队列方法

BlockingQueue是JDK5.0的新增内容，它是一个已经在内部实现了同步的队列，实现方式采用的是我们第2种await() / signal()方法。它可以在生成对象时指定容量大小。它用于阻塞操作的是put()和take()方法。

put()方法：类似于我们上面的生产者线程，容量达到最大时，自动阻塞。
take()方法：类似于我们上面的消费者线程，容量为0时，自动阻塞。

```java
package test;

import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.BlockingQueue;

public class Hosee {
	private static Integer count = 0;
	final BlockingQueue<Integer> bq = new ArrayBlockingQueue<Integer>(10);
	class Producer implements Runnable {
		@Override
		public void run() {
			for (int i = 0; i < 10; i++) {
				try {
					Thread.sleep(3000);
				} catch (Exception e) {
					e.printStackTrace();
				}
				try {
					bq.put(1);
					count++;
					System.out.println(Thread.currentThread().getName()
							+ "生产者生产，目前总共有" + count);
				} catch (InterruptedException e) {
					// TODO Auto-generated catch block
					e.printStackTrace();
				}
			}
		}
	}

	class Consumer implements Runnable {

		@Override
		public void run() {
			for (int i = 0; i < 10; i++) {
				try {
					Thread.sleep(3000);
				} catch (InterruptedException e1) {
					e1.printStackTrace();
				}
				try {
					bq.take();
					count--;
					System.out.println(Thread.currentThread().getName()
							+ "消费者消费，目前总共有" + count);
				} catch (Exception e) {
					// TODO: handle exception
					e.printStackTrace();
				}
			}
		}

	}

	public static void main(String[] args) throws Exception {
		Hosee hosee = new Hosee();
		new Thread(hosee.new Producer()).start();
		new Thread(hosee.new Consumer()).start();
		new Thread(hosee.new Producer()).start();
		new Thread(hosee.new Consumer()).start();

		new Thread(hosee.new Producer()).start();
		new Thread(hosee.new Consumer()).start();
		new Thread(hosee.new Producer()).start();
		new Thread(hosee.new Consumer()).start();
	}

}
```

## Semaphore方法

Semaphore 信号量，就是一个允许实现设置好的令牌。也许有1个，也许有10个或更多。 
谁拿到令牌(acquire)就可以去执行了，如果没有令牌则需要等待。 
执行完毕，一定要归还(release)令牌，否则令牌会被很快用光，别的线程就无法获得令牌而执行下去了。

```java
package test;

import java.util.concurrent.Semaphore;

public class Hosee
{
	int count = 0;
	final Semaphore notFull = new Semaphore(10);
	final Semaphore notEmpty = new Semaphore(0);
	final Semaphore mutex = new Semaphore(1);

	class Producer implements Runnable
	{
		@Override
		public void run()
		{
			for (int i = 0; i < 10; i++)
			{
				try
				{
					Thread.sleep(3000);
				}
				catch (Exception e)
				{
					e.printStackTrace();
				}
				try
				{
					notFull.acquire();//顺序不能颠倒，否则会造成死锁。
					mutex.acquire();
					count++;
					System.out.println(Thread.currentThread().getName()
							+ "生产者生产，目前总共有" + count);
				}
				catch (Exception e)
				{
					e.printStackTrace();
				}
				finally
				{
					mutex.release();
					notEmpty.release();
				}

			}
		}
	}

	class Consumer implements Runnable
	{

		@Override
		public void run()
		{
			for (int i = 0; i < 10; i++)
			{
				try
				{
					Thread.sleep(3000);
				}
				catch (InterruptedException e1)
				{
					e1.printStackTrace();
				}
				try
				{
					notEmpty.acquire();//顺序不能颠倒，否则会造成死锁。
					mutex.acquire();
					count--;
					System.out.println(Thread.currentThread().getName()
							+ "消费者消费，目前总共有" + count);
				}
				catch (Exception e)
				{
					e.printStackTrace();
				}
				finally
				{
					mutex.release();
					notFull.release();
				}

			}

		}

	}

	public static void main(String[] args) throws Exception
	{
		Hosee hosee = new Hosee();
		new Thread(hosee.new Producer()).start();
		new Thread(hosee.new Consumer()).start();
		new Thread(hosee.new Producer()).start();
		new Thread(hosee.new Consumer()).start();

		new Thread(hosee.new Producer()).start();
		new Thread(hosee.new Consumer()).start();
		new Thread(hosee.new Producer()).start();
		new Thread(hosee.new Consumer()).start();
	}

}
```

## PipedInputStream / PipedOutputStream

这个类位于java.io包中，是解决同步问题的最简单的办法，一个线程将数据写入管道，另一个线程从管道读取数据，这样便构成了一种生产者/消费者的缓冲区编程模式。PipedInputStream/PipedOutputStream只能用于多线程模式，用于单线程下可能会引发死锁。

```java
package test;

import java.io.IOException;
import java.io.PipedInputStream;
import java.io.PipedOutputStream;

public class Hosee {
	final PipedInputStream pis = new PipedInputStream();
	final PipedOutputStream pos = new PipedOutputStream();
	{
		try {
			pis.connect(pos);
		} catch (IOException e) {
			e.printStackTrace();
		}
	}

	class Producer implements Runnable {
		@Override
		public void run() {
			try{
                while(true){
                    int b = (int) (Math.random() * 255);
                    System.out.println("Producer: a byte, the value is " + b);
                    pos.write(b);
                    pos.flush();
                }
            }catch(Exception e){
                e.printStackTrace();
            }finally{
                try{
                    pos.close();
                    pis.close();
                }catch(IOException e){
                    System.out.println(e);
                }
            }
		}
	}

	class Consumer implements Runnable {

		@Override
		public void run() {
			try{
                while(true){
                    int b = pis.read();
                    System.out.println("Consumer: a byte, the value is " + String.valueOf(b));
                }
            }catch(Exception e){
                e.printStackTrace();
            }finally{
                try{
                    pos.close();
                    pis.close();
                }catch(IOException e){
                    System.out.println(e);
                }
            }
		}

	}

	public static void main(String[] args) throws Exception {
		Hosee hosee = new Hosee();
		new Thread(hosee.new Producer()).start();
		new Thread(hosee.new Consumer()).start();
	}

}
```

与阻塞队列一样，由于read()/write()方法与输出方法不一定同步，输出结果方面会发生不匹配现象，为了使结果更加明显，这里只有1个消费者和1个生产者。