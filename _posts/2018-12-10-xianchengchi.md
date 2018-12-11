---
layout: post
#标题配置
title:  Java线程池
#时间配置
date:   2018-12-10 14:33:00 +0800
#大类配置
categories: 并发
#小类配置
tag: 线程池
---

* content
{:toc}




### 一、线程池的作用
Thread其实是一种特别重量级的资源，创建、启动、销毁其实都是比较耗费系统资源的，因此对于线程的重复利用是一种特别好的编程习惯，加之程序中可创建的线程数量是有限的，一般系统的性能越好，可以创建的线程数目就越多。但是当线程创建的越多时，系统的性能就会越差。自JDK1.5起，Util包就提供了ExecutorService线程池的实现，主要目的是为了重复利用线程，提高系统的效率。
###二、线程池的原理
线程池，就好比有一个大池子，里面装着已经创建好的线程，当有任务交给线程池执行的话，池中的某个线程会执行该任务。如果任务量太大，而池中的线程又不够的情况下，则需要重新扩充新的线程到线程池中，但是数量是有限的，因为池子是有容量的，不能无限去扩充。当任务比较少的时候，池中的线程会自动回收，不然占着不必要的系统资源。为了能够异步去提交任务和缓冲未被处理的任务，需要有一个任务队列。
####问题：完整的线程池需要包含哪些东西？？？
- 任务队列：用于缓存提交的任务。
- 线程数量管理功能：一个好的线程池必须有管理和控制线程的能力，可以通过如下三个参数去实现。
  1.init：创建线程池初始的线程数量
  2.max:自动扩充时最大的线程数量
  3.core:线程池虽然在空闲的时候需要释放一些线程，但是也需要保存一些，也就要有一定的核心线程量和活跃线程量core。
- 任务拒绝策略:如果线程都已经在使用，并且任务队列已经满了，肯定不能够接收新的任务了，这个时候需要拒绝任务并告知任务发送者。
- 线程工厂：个性化定制线程，比如把线程设置为守护线程或给线程设置线程名。
- QueueSize:任务队列主要存放提交的Runnable，但是为了防止内存溢出也要有一定的数量限制。
- Keepedalive:该时间主要决定线程各个主要参数自动维护的时间间隔。
###三、线程池的实现
我们通过写代码的形式来实现一个简单的ThreadPool(线程池)，虽然简单，但是功能基本具备。我们先来看一下线程的实现类图。
![线程池实现类图.png](https://upload-images.jianshu.io/upload_images/12475647-027d1d95998ba8bf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
#### 1.ThreadPool
主要定义了ThreadPool应具备的基本操作和方法
    package com.lntu.zlt;

    public interface ThreadPool {

	// 提交任务到线程池
	void execute(Runnable runnable);

	// 关闭线程池
	void shutdown();

	// 获取线程池的初始化大小
	int getInitSize();

	// 获取线程池最大的线程数
	int getMaxSize();

	// 获取线程池的核心线程数量
	int getCoreSize();

	// 获取线程池中用于缓存队列任务的大小
	int getQueueSize();

	// 获取线程池中活跃线程的数量
	int getActiveCount();

	// 查看线程池是否已经被shutdown
	boolean isShutdown();
    }

#### 2.RunnableQueue
RunnableQueue主要存放提交的Runnable，该Runnable是一个BlockedQueue，并有limit限制。

     package com.lntu.zlt;

    public interface RunnableQueue {
	// 将任务提交到队列中，有新的任务进来时首先会offer到队列中
	void offer(Runnable runnable);

	// 工作线程通过take()方法获取Runnable，从队列中获取相应的任务。
	Runnable take();

	// 获取任务队列中当前Runnable的数量
	int size();

      }
#### 3.ThreadFactory
ThreadFactory提供了创建线程的接口，个性化设置Thread，比如加入到哪个Group里面，优先级，Thread名字。

     package com.lntu.zlt;
    //创建线程的工厂
     @FunctionalInterface
     public interface ThreadFactory {
 	// 用于创建线程
	Thread createThread(Runnable runnable);
      }
#### 4.DenyPolicy
DenyPolicy主要用于当Queue中的Runnable达到了limit上限的话，决定采用哪种策略通知提交者。

    package com.lntu.zlt;

    @FunctionalInterface
    public interface DenyPolicy {
	void reject(Runnable runnable, ThreadPool threadPool);

	// 该策略会直接将任务丢弃
	class DiscardDenyPolicy implements DenyPolicy {

		@Override
		public void reject(Runnable runnable, ThreadPool threadPool) {
			// TODO Auto-generated method stub
			// do nothing

		}
	}

	// 该拒绝策略会向任务提交者抛出异常
	class AbortDenyPolicy implements DenyPolicy {

		@Override
		public void reject(Runnable runnable, ThreadPool threadPool) {
			// TODO Auto-generated method stub
			throw new RunnableDenyException("The runnable" + runnable + "will be abort");
		}

	}

	// 该拒绝策略会使任务在提交者所在的线程中执行任务
	class RunnerDenyPolicy implements DenyPolicy {

		@Override
		public void reject(Runnable runnable, ThreadPool threadPool) {
			// TODO Auto-generated method stub
			if (!threadPool.isShutdown()) {
				runnable.run();
			}
		}

	}
     }
#### 5.RunnableDenyException
RunnableDenyException是RuntimeException的子类，他主要通知任务的提交者，任务队列已经无法再接收新的任务。
    
    package com.lntu.zlt;
    public class RunnableDenyException extends RuntimeException {
	public RunnableDenyException(String message) {
		// TODO Auto-generated constructor stub
		super(message);
	}
    }
    
#### 6.InternalTask
InternalTask是Runnable接口的实现，那么大家应该能够知道他是干嘛的了。该类会读取RunnableQueue，然后不断从queue中读取出某个Runnable，然后执行Runnable的run方法
  
    package com.lntu.zlt;

    public class InternalTask implements Runnable {
	private final RunnableQueue runnableQueue;
	private volatile boolean running = true;

	public InternalTask(RunnableQueue runnableQueue) {
		this.runnableQueue = runnableQueue;
	}

	@Override
	public void run() {
		// 如果当前任务为running并且没有被中断，则其不断地从queue中获取runnable，然后执行润方法
		while (running && !Thread.currentThread().isInterrupted()) {
			try {
				Runnable task = runnableQueue.take();
				task.run();
			} catch (InterruptedException e) {
				// TODO: handle exception
				running = false;
				break;
			}

		}

	}

	// 停止当前任务，主要会在线程池的shutdown方法中使用
	public void stop() {
		this.running = false;
	}
    }

### 四、线程池面试题
这块转子一个简书作者，他这篇帖子写的比较细。
作者：闽越布衣
链接：https://www.jianshu.com/p/f84762a42512
來源：简书
   


- 什么是线程池
    创建一组可供管理的线程，它关注的是如何缩短或调整线程的创建与销毁所消费时间的技术，从而提高服务器程序性能的。
    它把线程的创建与销毁分别安排在服务器程序的启动和结束的时间段或者一些空闲的时间段以减少服务请求是去创建和销毁线程的时间。
    它还显著减少了创建线程的数目。

- 为什么要使用线程池
    当我们在使用线程时，如果每次需要一个线程时都去创建一个线程，这样实现起来很简单，但是会有一个问题：当并发线程数过多时，并且每个线程都是执行一个时间很短的任务就结束时，这样创建和销毁线程的时间要比花在实际处理任务的时间要长的多，在一个JVM里创建太多的线程可能会导致由于系统过度消耗内存或切换过度导致系统资源不足而导致OOM问题。    线程池为线程生命周期开销问题和资源不足问题提供了解决方案。通过对多个任务重用线程，线程创建的开销被分摊到了多个任务上。

- 如何正确的使用线程池
不要对那些同步等待其它任务结果的任务排队。这可能会导致上面所描述的那种形式的死锁，在那种死锁中，所有线程都被一些任务所占用，这些任务依次等待排队任务的结果，而这些任务又无法执行，因为没有空闲的线程可以使用。
在为任务时间可能很长的线程使用合用的线程时要小心。如果程序必须等待诸如 I/O 完成这样的某个资源，那么请指定最长的等待时间，以及随后是失效还是将任务重新排队以便稍后执行。
理解任务。要有效地调整线程池大小，需要理解正在排队的任务以及它们正在做什么。它们是 CPU 限制的吗？它们是 I/O 限制的吗？你的答案将影响如何调整应用程序。如果有不同的任务类，这些类有着截然不同的特征，那么为不同任务类设置多个工作队列可能会有意义，这样可以相应地调整每个池。
ThreadPoolExecutor构造函数中几个重要参数的解释
corePoolSize：核心池大小，即线程的数量。
maximumPoolSize：线程池最大线程数，表示在线程池中最多能创建多少个线程。
keepAliveTime：表示线程没有任务执行时最多保持多久时间会终止。
unit：参数keepAliveTime的时间单位，有7种取值，具体可查看前面章节。
workQueue：线程池采用的缓冲队列，用来存储等待执行的任务。
threadFactory：线程工厂，主要用来创建线程。
handler：线程的阻塞策略。当线程池中线程数量达到maximumPoolSize时，仍有任务需要创建线程来完成，则handler采取相应的策略。
- 线程池的状态
RUNNING：能接受新提交的任务，并且也能处理阻塞队列中的任务；
SHUTDOWN：关闭状态，不再接受新提交的任务，但却可以继续处理阻塞队列中已保存的任务。在线程池处于 RUNNING 状态时，调用 shutdown()方法会使线程池进入到该状态.(finalize() 方法在执行过程中也会调用shutdown()方法进入该状态);
STOP：不能接受新任务，也不处理队列中的任务，会中断正在处理任务的线程。在线程池处于 RUNNING 或 SHUTDOWN 状态时，调用 shutdownNow() 方法会使线程池进入到该状态；
TIDYING：如果所有的任务都已终止了，workerCount (有效线程数) 为0，线程池进入该状态后会调用 terminated() 方法进入TERMINATED 状态。
TERMINATED：在terminated() 方法执行完后进入该状态，默认terminated()方法中什么也没有做。
线程池的执行流程
当线程数量小于corePoolSize时，任务来时会创建新的线程来处理，并把该线程加入线程队列中(实际上是一个HashSet)(此步骤需要获取全局锁，ReentryLock)；
如果当前线程数量达到了corePoolSize，任务来时将任务加入BlockingQueue；
如果任务列队满了无法加入新的任务时，会创建新的线程(同样需要获取全局锁);
如果线程池数量达到maximumPoolSize，并且任务队列已满，新的任务将被拒绝;
- 常用的几种线程池
CachedThreadPool： 创建一个可缓存线程池，如果线程池长度超过处理需要，可灵活回收空闲线程，若无可回收，则新建线程。
FixedThreadPool：创建一个指定工作线程数量的线程池。每当提交一个任务就创建一个工作线程，如果工作线程数量达到线程池初始的最大数，则将提交的任务存入到阻塞队列中。
SingleThreadExecutor：创建一个单线程化的Executor，即只创建唯一的工作者线程来执行任务，它只会用唯一的工作线程来执行任务，保证所有任务按照指定顺序(FIFO, LIFO, 优先级)执行。
ScheduleThreadPool：创建一个定长的线程池，而且支持定时的以及周期性的任务执行，支持定时及周期性任务执行。


