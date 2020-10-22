---
title: Java中的锁
tags:
  - Java锁
categories: [Java基础,锁]
date: 2020-10-19 14:59:36
---
# 公平锁 / 非公平锁

## 公平锁

**公平锁：(FIFO)多个线程在等待同一个锁时，必须按照申请锁的先后顺序来一次获得锁。**

> 公平锁就好像在餐厅的门口安装了一个排队的护栏，谁先来的谁就站的靠前，无法进行插队，当餐厅中的人用餐结束后会把钥匙交给排在最前边的那个人，以此类推。
>
> 公平锁的好处是，可以保证每个排队的人都有饭吃，先到先吃后到后吃。
>
> 但是弊端是，**要额外安装排队装置。**

<!--more-->

## 非公平锁

**非公平锁：多个线程获取锁的顺序并不是按照申请锁的顺序，有可能后申请的线程比先申请的线程优先获取锁。有可能，会造成优先级反转或者饥饿现象。**

> 非公平锁就是不用排队，当餐厅里的人出来后将钥匙往地上一扔，谁抢到算谁的。
>
> 但是这样就造成了一个问题，那些身强体壮的人可能总是会先抢到钥匙，而那些身体瘦小的人可能一直抢不到。
>
> 这就有可能将一直抢不到钥匙，最后导致需要很长时间才能拿到钥匙甚至一直拿不到直至饿死。
>
> **非公平锁的优点在于吞吐量比公平锁大。**

**对于`Synchronized`而言，是一种非公平锁。**由于其并不像`ReentrantLock`是通过`AQS`的来实现线程调度，所以并没有任何办法使其变成公平锁。

对于`Java ReentrantLock`而言，通过构造函数指定该锁是否是公平锁。默认是**非公平锁**。

```java
 /**
     * Creates an instance of {@code ReentrantLock}.
     * This is equivalent to using {@code ReentrantLock(false)}.
     */
    public ReentrantLock() {
        sync = new NonfairSync();
    }

    /**
     * Creates an instance of {@code ReentrantLock} with the
     * given fairness policy.
     *	传入true则变公平锁
     * @param fair {@code true} if this lock should use a fair ordering policy
     */
    public ReentrantLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
    }
```



# 互斥锁 / 读写锁

## 互斥锁



> 互斥锁 通过**悲观锁**的理念引出来的；
>
> 非互斥锁  通过**乐观锁**的概念引申的。



互斥锁也就是常说的**同步**，即一次最多只能有一个线程持有的锁，当一个线程持有该锁的时候其它线程无法进入上锁的区域。

在Java中`synchronized`就是互斥锁，从宏观概念来讲，互斥锁就是通过**悲观锁**的理念引出来的，而非互斥锁则是通过**乐观锁**的概念引申的。



## 读写锁

**读写锁有三种状态**：**读加锁**状态、写加锁状态和**不加锁**状态

**读写锁在Java中的具体实现就是**`ReadWriteLock`

一次只有一个线程可以占有写模式的读写锁，但是多个线程可以同时占有读模式的读写锁。
只有一个线程可以占有写状态的锁，但可以有多个线程同时占有读状态锁，这也是它可以实现高并发的原因。当其处于写状态锁下，任何想要尝试获得锁的线程都会被阻塞，直到写状态锁被释放；如果是处于读状态锁下，允许其它线程获得它的读状态锁，但是不允许获得它的写状态锁，直到所有线程的读状态锁被释放；为了避免想要尝试写操作的线程一直得不到写状态锁，当读写锁感知到有线程想要获得写状态锁时，便会阻塞其后所有想要获得读状态锁的线程。所以读写锁非常适合资源的读操作远多于写操作的情况。





# 可重入锁 / 不可重入锁

## 可重入锁

广义上的可重入锁指的是可重复可递归调用的锁，在外层使用锁之后，在内层仍然可以使用，并且不发生死锁（前提得是同一个对象或者class）`ReentrantLock`和`synchronized`都是可重入锁

```java
public class ReentrantDemo {
	private Lock mLock;
 	//若不是可重入锁，inner()将不会在outer()中调用。需要等待外层outer()先释放锁
	public ReentrantDemo(Lock mLock) {
		this.mLock = mLock;
	}
 
	public void outer() {
		mLock.lock();
		inner();
		mLock.unlock();
	}
 
	public void inner() {
		mLock.lock();
		// do something
		mLock.unlock();
	}
}

```

```java
synchronized void setA() throws Exception{
   Thread.sleep(1000);
   setB();
}
synchronized void setB() throws Exception{
   Thread.sleep(1000);
}//如果synchronized不是可重入锁的话，setB可能不会被当前线程执行，可能造成死锁。
```



## 不可重入锁

不可重入锁，与可重入锁相反，不可递归调用，递归调用就发生死锁。

### 例子1 不可重入的自旋锁

使用**自旋锁**来模拟一个不可重入锁，代码如下：

```java
import java.util.concurrent.atomic.AtomicReference;

 //使用原子引用来存放线程，同一线程两次调用lock()方法，如果不执行unlock()释放锁的话，第二次调用自旋的时候就会产生死锁。
 //这个锁就不是可重入的，而实际上同一个线程不必每次都去释放锁再来获取锁，这样的调度切换是很耗资源的。
public class UnreentrantLock {
 
   private AtomicReference<Thread> owner = new AtomicReference<Thread>();
	
   public void lock() {
       Thread current = Thread.currentThread();
       //这句是很经典的“自旋”语法，AtomicInteger中也有
       for (;;) {
           if (!owner.compareAndSet(null, current)) {
               return;
           }
       }
   }
   public void unlock() {
       Thread current = Thread.currentThread();
       owner.compareAndSet(current, null);
   }
}
```



### 例子2 不可重入锁

下面也是一段**不可重入锁**的代码：

```java
//可以看到，当isLocked被设置为true后，在线程调用unlock()解锁之前不管线程是否已经获得锁，都只能wait()。
public class Lock{  
    private boolean isLocked = false;  
    public synchronized void lock()  
        throws InterruptedException{  
        while(isLocked){  
            wait();  
        }  
        isLocked = true;  
    }  
 
    public synchronized void unlock(){  
        isLocked = false;  
        notify();  
    }  
}  
```





# 乐观锁 / 悲观锁

![image-20201019203700846](https://i.loli.net/2020/10/19/xt3d6XGgKWhaNr5.png)



## 乐观锁(非互斥锁)

> 总是假设最好的情况，每次去拿数据的时候都认为别人不会修改，所以不会上锁，但是在更新的时候会判断一下在此期间别人有没有去更新这个数据，可使用版本号机制和CAS算法实现。
>
> (比较与上一次的版本号，如果一样则更新），如果失败则要继续CAS(CompareAndSet)的操作。

乐观锁适用于**多读**的应用类型，这样可以提高吞吐量，像数据库提供的类似于`write_condition`机制，其实都是提供的乐观锁。

## CAS并不是一种实际的锁，它只是实现乐观锁的一种思想。

java中的乐观锁（如自旋锁）基本都是通过CAS操作实现的，CAS是一种更新的原子操作，比较当前值跟传入值是否一样，一样则更新，否则失败。在`Java`中`java.util.concurrent.atomic`包下面的**原子变量类就是使用了乐观锁的一种实现方式CAS实现的**。



## 悲观锁（互斥锁）

> 悲观锁是就是悲观思想，即认为**读少写多**，遇到**并发写**的可能性高。
>
> 每次去拿数据的时候都认为别人会修改，所以每次在读写数据的时候都会上锁，这样别人想读写这个数据就会block直到拿到锁。



传统的关系型数据库里边就用到了很多这种锁机制，比如**行锁**，**表锁**等，**读锁**，写锁等，都是在做操作之前先上锁。

`Java`中`synchronized`和`ReentrantLock`等**独占锁**就是悲观锁思想的实现。





# 自旋锁

**自旋锁（spinlock）当一个线程在获取锁的时候，如果锁已经被其它线程获取，那么该线程将循环等待，然后不断的判断锁是否能够被成功获取，直到获取到锁才会退出循环**。

**它是为实现保护共享资源而提出一种锁机制。**

> 自旋锁与互斥锁比较类似，它们都是为了解决对某项资源的互斥使用。
>
> **无论是互斥锁，还是自旋锁，在任何时刻，最多只能有一个保持者，也就说，在任何时刻最多只能有一个执行单元获得锁**。
>
> 但是两者在调度机制上略有不同：
>
> - 对于互斥锁，如果资源已经被占用，资源申请者只能进入睡眠状态。
> - 对于自旋锁，不会引起调用者睡眠，如果自旋锁已经被别的执行单元保持，调用者就一直循环在那里看是否该自旋锁的保持者已经释放了锁，”自旋”一词就是因此而得名。



## 不可重入的自旋锁

```java
import java.util.concurrent.atomic.AtomicReference;

public class UnreentrantLock {
  //使用原子引用来存放线程，同一线程两次调用lock()方法，如果不执行unlock()释放锁的话，第二次调用自旋的时候就会产生死锁。
 //这个锁就不是可重入的，而实际上同一个线程不必每次都去释放锁再来获取锁，这样的调度切换是很耗资源的。
   private AtomicReference<Thread> owner = new AtomicReference<Thread>();
	
   public void lock() {
       Thread current = Thread.currentThread();
       //这句是很经典的“自旋”语法，AtomicInteger中也有
       for (;;) {
           if (!owner.compareAndSet(null, current)) {
               return;
           }
       }
   }
   public void unlock() {
       Thread current = Thread.currentThread();
       owner.compareAndSet(current, null);
   }
}
```

`lock（)`方法利用的CAS，当第一个线程A获取锁的时候，能够成功获取到，不会进入while循环，如果此时线程A没有释放锁，另一个线程B又来获取锁，此时由于不满足CAS，所以就会进入while循环，不断判断是否满足CAS，直到A线程调用`unlock()`方法释放了该锁。



## 可重入的自旋锁

上面的那段代码，它是不支持重入的，即当一个线程第一次已经获取到了该锁，在锁释放之前又一次重新获取该锁，第二次就不能成功获取到。由于不满足CAS，所以第二次获取会进入while循环等待，而如果是可重入锁，第二次也是应该能够成功获取到的。

而且，即使第二次能够成功获取，那么当第一次释放锁的时候，第二次获取到的锁也会被释放，而这是不合理的。

为了实现可重入锁，我们需要引入一个计数器，用来记录获取锁的线程数。

```java
public class ReentrantSpinLock {
   // 利用AtomicReference来调用CAS，ar初始（内存）值是null
   private AtomicReference<Thread> cas = new AtomicReference<Thread>();
   private int count;
   public void lock() {
       Thread current = Thread.currentThread();
       // 获取内存值，若已取到锁（初始值为null，当内存值也为null说明取到锁了）
       if (current == cas.get()) { // 如果当前线程已经获取到了锁，线程数增加一，然后返回
           count++;
           return;
       }
       // 如果没获取到锁，则通过CAS自旋
       while (!cas.compareAndSet(null, current)) {
           // DO nothing
       }
   }
   public void unlock() {
       Thread cur = Thread.currentThread();
       if (cur == cas.get()) {
           if (count > 0) {// 如果大于0，表示当前线程多次获取了该锁，释放锁通过count减一来模拟
               count--;
           } else {// 如果count==0，可以将锁释放，这样就能保证获取锁的次数与释放锁的次数是一致的了。
               cas.compareAndSet(cur, null);
           }
       }
   }
}
```



## **自旋锁的优点**

1、自旋锁不会使线程状态发生切换，一直处于用户态，即线程一直都是active的；不会使线程进入阻塞状态，减少了不必要的上下文切换，执行速度快
2、非自旋锁在获取不到锁的时候会进入阻塞状态，从而进入内核态，当获取到锁的时候需要从内核态恢复，需要线程上下文切换。（线程被阻塞后便进入内核（Linux）调度状态，这个会导致系统在用户态与内核态之间来回切换，严重影响锁的性能）



## 自旋锁与互斥锁

1. 自旋锁与互斥锁都是为了实现保护资源共享的机制。

2. 无论是自旋锁还是互斥锁，在任意时刻，都最多只能有一个保持者。

3. 获取互斥锁的线程，如果锁已经被占用，则该线程将进入睡眠状态；获取自旋锁的线程则不会睡眠，而是一直循环等待锁释放。



### **自旋锁总结**

1. 自旋锁：线程获取锁的时候，如果锁被其他线程持有，则当前线程将循环等待，直到获取到锁。

2. 自旋锁等待期间，线程的状态不会改变，线程一直是用户态并且是活动的(active)。

3. 自旋锁如果持有锁的时间太长，则会导致其它等待获取锁的线程耗尽CPU。

4. 自旋锁本身无法保证公平性，同时也无法保证可重入性。

5. 基于自旋锁，可以实现具备公平性和可重入性质的锁。



# 死锁

## 什么是死锁

所谓死锁是指多个线程因竞争资源而造成的一种僵局（互相等待），若无外力作用，这些进程都将无法向前推进。

> 两人人一起吃饭但是只有一双筷子
>
> 某一个时候，A拿了左筷子，B拿了右筷子，两人都在**循环等待**等待另一跟筷子，二两人都不肯放弃手中的筷子（不可剥夺）。



## 死锁本质原因

   **本质原因：**

1. 系统资源有限。

2. 进程推进顺序不合理。



## 形成死锁的四个必要条件

1. **互斥**：进程要求对所分配的资源仅能给一个进程拥有

2. **不可剥夺**：进程所获得的资源在未使用完毕之前，不能被其他进程强行夺走（只能是主动释放)。

3. **请求和保持**：进程已经保持了至少一个资源，但又提出了新的资源请求，而该资源 已被其他进程占有，此时请求进程被阻塞，但对自己已获得的资源保持不放。

   > 比如：A拿了左筷子，但又想要右筷子，而右筷子在B手中。

4. **循环等待**：存在一种进程资源的循环等待链，链中每一个进程已获得的资源同时被 链中下一个进程所请求。即存在一个处于等待状态的进程集合{Pl, P2, ..., pn}，其中Pi等 待的资源被P(i+1)占有（i=0, 1, ..., n-1)，Pn等待的资源被P0占有。

   > 比如：A等待的筷子在B手中，B等待的筷子在A手中。



# 活锁

活锁可以认为是一种特殊的饥饿。

活锁和死锁在表现上是一样的两个线程都**没有任何进展**。

但是区别在于：死锁，两个线程都处于**阻塞状态**，说白了就是它不会再做任何动作，我们通过查看线程状态是可以分辨出来的。

而活锁并不会阻塞，而是一直**尝试去获取需要的锁**，不断的try，这种情况下线程并没有阻塞所以是**活的状态**，我们查看线程的状态也会发现线程是正常的，但重要的是整个程序却不能继续执行了，一直在做无用功。

例子：

> 两个人在窄路相遇，同时向一个方向避让，然后又向另一个方向避让，如此反复，还是不能让对方通过。