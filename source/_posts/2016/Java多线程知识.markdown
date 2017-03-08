layout: "post"
title: "Java多线程知识"
date: "2016-12-20 15:45"
---
## 目录
1. java的并行与并发
2. java线程安全的概念
3. java中CAS的概念
4. java中AtomicInteger相关类的实现原理
5. java乐观锁与悲观锁
6. java加锁 syncnized和reentrantlock
7. ReentrantLock的使用方法和原理
8. 如何让多个线程顺序执行,迅雷面试题ABC三个线程顺序输出的问题
9. java的join方法
10. ReadWriteLock使用和实现原理
11. 脏读,可重复读,幻读的概念
12. 死锁相关概念
13. java信号量
14. java 多线程包下的常用类使用
15. 生产者消费者

### 并发与并行
  并发性(concurrency) : 有称共行性,是指有能力处理多个同时性活动的能力,并发事件之间不一定要同一事件发生.
  并行(parallelism) : 是指同时发生的两个并发事件,具有并发的含义,而并发不一定并行.

  可见两者最大的区别是能否在同一时间进行.并行是并发的子集.

![并发与并行图例](/images/2016/12/并发与并行.jpg)

### java线程安全的概念
  java线程安全是相对多线程概念来提出的,线程安全是指一个方法或者类实例可以在不同的线程中同时使用而不会引起任何的问题.
  考虑一下下面的情况
  ```java
  private int myInt = 0;
  public int addOne(){
    int temp = myInt;
    temp = temp + 1;
    myInt = tmp;
    return temp;
  }
  ```
  现在有A和B两个线程将要同时执行addOne()方法,但是A先开始并且把myInt的值0赋给了temp,同时由于某些原因线程调度决定挂起A线程,同时执行B线程,B线程也是把myInt的值O赋值给了它自己的temp,并且方法完全执行结束,这个时候最终myInt=1 ,并且返回了1.现在又轮到了线程A,在A中的temp的值为0,执行加加一操作后,变为1,同时返回的值也为1.
  在上面的情况下,方法addOne执行了两边,但是由于该方法是非线程安全的导致没有返回预期的结果2,由于第二个线程先于第一个线程保存myInt的值,所以都是返回了1.

  创建线程安全的方法不算太繁琐,并且有一些技巧.在java中你可以声明一个方法为 *synchronized* ,这个关键字的含义是在同一时间只能有同一个线程来执行当前方法.其他线程需要等待.这会保证方法是线程安全的,但是假如在这个方法中有一堆事情要做的话会浪费很长的事件.另一种方式是 *对需要保证线程同步的方法中的部分* 来加锁或者信号量.只是锁住其中的一小部分(称为 *临界区* ).甚至在java中也提供了很多方法来保证非锁的线程安全,这些方法保证多线程可以在同一时间内竞争,并且导致该方法完成一套原子性的操作.原子性是指不能被中断并且在同一时间内只能在一个线程中执行. 后面我们会讲到的 *CAS* 就是这种方式的内部实现原理 .

### java中的CAS概念
  CAS全称Compare And Swap ,即比较并交换,设计并发算法中常用到的一种技术,java.util.concurrent包完全建立在CAS的基础上.
  当前的处理器基本都支持CAS,只不过每家的实现不一样.
  *CAS有三个操作数,内存值V,旧的预期值A,要修改的值B,当且仅当预期的值A和内存值V相同时,将内存值修改为B并返回True,否者什么都不做返回False*
  下面来看上面demo的CAS实现:
  首先myInt必须是 *valatile* 的了,这样才能保证多个线程之间的数据变化是可见的.(valatile的用法后面会提到)
  ```java
  private int valatile myInt = 0;
  public final int get(){
    return myInt;
  }
  public final int addOneAndGet(){
    for (; ; ) {
      int current = get();
      int next = current + 1;
      if(compareAndSet(current,next)){
        return next;
      }
    }
  }
  ```
  这里通过方法 *compareAndSet* 来完成CAS的底层实现.也就是比较current和内存中的值,如果两个一样就放回true,退出死循环返回next,否者不断的从内存中获取和当前的current比较,知道匹配成功.这里通过一个死循环来达到不断更新current值得目的.
  再来回头想想上面的非线程安全情景,如果B处理完成后myInt变成1,这个时候A发现自己的myInt与内存中的不一致,就重新获取一下myInt的值,直到两者一致,然后完成加一操作,就达到了一种不用加锁也可以线程之间同步资源的目的.

  ```java
  public final boolean compareAndSet(int expect, int update) {   
    return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
    }
  ```
  这里unsafe是底层native的jni实现,借由C来调用CPU底层指令来实现的.

  上面的CAS实现其实也是java的同步包中 AtomicInteger的内部实现原理.

  可以看到CAS还是比较容易理解的,就是字面意思,对比然后赋值交换.下面说说CAS的缺点;

  1. ABA问题.因为CAS需要在操作的时候检查值有没有发生变化,如果没变化就更新,如果一个值原来是A,后来改成B,又变回到A,这个时候CAS检查发现值没有变化,实际上已经变化过了.这个就是ABA问题,通过在变量前面加上版本号的思路可以解决ABA问题,在变量前面加上版本号,每次变量更新的时候把版本号加1,那么A-B-A就会变成1A-2B-3A,这时就可以比较出第一个A和最后一个A是不一样的了.在java的current包中提供了AtomicStampedRefrence来解决ABA问题.
  2. 循环时间长开销大.自旋CAS如果长时间不成功,会给CPU带来很大的开销.如果JVM支持pause指令那么效率会有一定的提升.
  3. 只能保证一个共享变量的原子操作.很明显,上面的demo只是对一个变量做了原子加一操作.可以通过AtomicRefrence类来保证引用对象之间的原子性.

### java中AtomicInteger的实现原理
  AtomicInteger是一个完成的CAS实现,通过上面的CAS说明,可以发现最重要的就是比较内存值和当前值是否相等的逻辑.首先看如何获取内存值
  ```java
  static {
      try {
          valueOffset = unsafe.objectFieldOffset
              (AtomicInteger.class.getDeclaredField("value"));
      } catch (Exception ex) { throw new Error(ex); }
  }

  ```
  这个是通过底层native直接获取 *value* 在内存中的偏移量.注意这个是静态的,只是在初次使用的时候就获取完成.
  由于 *value* 是 *valitale* 的,所以value的变化在各个线程中是可见.这样就可以比较当前value和内存中value是否相等了.
  其他的细节在上面已经提到了,这里就不在重复了.

###  java的乐观锁和悲观锁
  我们都知道,CPU是时分复用的,也就是把cpu的时间片,分配给不同的thread/process轮流执行,时间片与时间片之间,需要cpu切换,也就是会发生线程切换.切换涉及到清空寄存器,缓存数据.然后重新加载新的thread所需数据.当一个线程被挂起的时候,加入阻塞队列,在一定的事件或者条件下,在通过notifiy(),notifiyAll()唤醒回来.

  悲观锁
  是指对数据被外界修改持保守态度,因此,在整个数据处理过程中,讲数据处于锁定状态.
  在某个资源不可用的时候,就将CPU让出,把当前等待线程切换为阻塞状态.等到资源(比如一个共享数据)可用了,那么就将线程唤醒,让他进入runaable状态等待CPU调度.这就是典型的悲观锁的实现.独占锁是一种悲观锁,Syncnized就是一种独占锁,它假设最坏的情况,并且只有在确保其他线程不会赵成干扰的情况下执行,会导致其它所有需要锁的线程挂起,等待持有锁的线程释放所;

  但是由于在进程挂起和恢复执行过程中存在着很大的开销.当一个线程正在等待锁时,它不能做任何事,所以悲观锁有很大的缺点.举个例子,如果一个线程需要某个资源,但是这个资源占用的时间很短,当线程第一次抢占这个资源时,可能这个资源被占用,如果此时挂起这个线程,可能立刻就发现资源可用,然后又需要花费很长的时间重新抢占锁,时间代价就会非常的高.

  所以就有了乐观锁的概念.它的核心概念就是,每次不加锁而是假设没有冲突而去完成某项操作,如果因为冲突失败就重试,知道成功为止.在上面的例子中,某个线程可以不让出cpu,而是一直while循环,如果失败就重试,知道成功为止.所以,当数据争用不严重时,乐观锁效果更好.比如上面说的CAS就是一种乐观锁思想的应用.

### java加锁 synchronized和reentrantlock

#### synchronized

  可以通过给方法或代码块增加 *synchronized* 关键字来加互斥锁.如果是加在当前的对象中的实例方法中那锁对象就是当前的对象也就是 *this* ,如果是加在静态方法中的,那当前锁对象就是 *class* 对象.如果是代码块中的话就是括号里面传入的对象.
  下面通过一个demo来说明 *synchronized* 的用法

  ```java
  public class Sync {
      public synchronized void test() {
              System.out.println("test 开始");
              try {
                  Thread.sleep(1000);
              } catch (InterruptedException e) {
                  e.printStackTrace();
              }
              System.out.println("test 结束");
      }
  }
  ```
  sync是一个普通的java类,内部的实例方法test增加了 *synchronized* 关键字来加锁,保证同一时间内只能有一个线程来调用test.
  然后是调用的线程MyThread
  ```java
  public class MyThread extends Thread {
    private Sync sync;

    public MyThread() {
    }

    public MyThread(Sync sync) {
        this.sync = sync;
    }

    @Override
    public void run() {
        if (this.sync == null) {
            sync = new Sync();
        }
        sync.test();
    }
  }
  ```
  最后是测试类
  ```java
  public class TestSynchronized {
    public static void main(String[] args) {
       test();
    }
    public static void test() {
        for (int i = 0; i < 3; i++) {
            new MyThread().start();
        }
    }
    public static void test1() {
        Sync sync = new Sync();
        for (int i = 0; i < 3; i++) {
            new MyThread(sync).start();
        }
    }
  }
  ```
  首先来看第一个test方法,通过创建3个MyThread来同时执行,那加的锁会成功吗?,看输出:
  >test 开始
  >test 开始
  >test 开始
  >test 结束
  >test 结束
  >test 结束

  很明显,锁没有起作用,3个线程同时执行的.

  在看调用第二个test方法,外部Sync对象创建后之后传入3个线程中,看输出:
  >test 开始
  >test 结束
  >test 开始
  >test 结束
  >test 开始
  >test 结束

  看来这种情况下是成功了.

  和上面的说明一致,方法锁锁定的是当前的实例对象,第一种情况下3个不同的实例对象,所以不行
  而第二个test是用的同一个实例对象,所以加锁成功.
  如果把Sync的test方法内部创建一个 Sync类锁的话,那两个test方法都会加锁成功
  ```java
  public class Sync {
    public void test() {
        synchronized (Sync.class) {
            System.out.println("test 开始");
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println("test 结束");
        }
    }
  }
  ```



#### reentrantLock详解
首先 *reentrantLock* 是一种乐观锁,它比 *synchronized* 更加灵活,内部增加了很多机制,比如支持请求所超时,公平锁等功能.
*reentrantLock* 的翻译是 "可重入的的锁",可以对同一个线程申请多次锁,内部有一个计数器,每次调用 *lock* 方法的时候加一, *unlock* 的时候减一,知道计数器为0,最终释放锁.
普通的用法如下:
```java
class X {
  private final ReentrantLock lock = new ReentrantLock();
  //...

  public void m(){
    lock.lock(); // block until condition holds
    try {
      //... method body
    }final{
      lock.unlock();
    }
  }
}
```

关于 *ReentrantLock* 的高级用法我们后面再讲,现在先看其内部实现,尝试讲解一下源码:
*ReeentrantLock* 内部主要是通过一个Sync的抽象类来完成大部分的操作,这个Sync对象继承至 *AbstractQueuedSynchronizer* ,其中 *AbstractQueuedSynchronizer* 中实现了大部分的锁机制,只是抽象出了 *tryAcquire* 和 *tryReleas* 方法,交由子类实现.
而在 *ReeentrantLock* 中的 Sync对象有有两个实现类,分别是 *NonfairSync* 和 *FairSync* ,从命名上可以看出分别是不公平锁和公平锁的概念.
为啥会有公平不公共这种说法呢,还得从多线程的角度考虑:由于有锁的出现,导致同一时间只能有个线程可以执行,那其他的线程处在阻塞状态,如果锁释放了,那那些阻塞的线程将被唤醒,说明肯定是有个队列用来维护阻塞线程的.公平锁和非公平锁的区别就在于唤醒的时候是否需要按照队列的顺序来唤醒.公平锁的话是按照阻塞队列的顺序来获取锁,非公平锁的唤醒是先直接尝试取得锁,如果失败才会进入阻塞队列中按顺序获取锁.

*ReentrantLock* 默认构造方法实现的是非公平锁.也就是 *NonfairSync* .

首先是从 *lock* 方法开始
```java
public void lock() {
       sync.lock();
   }
```
lock方法调用的Sync对象的lock方法,这里默认初始化的是 *NonfairSync*,进入其内部来看:
```java
static final class NonfairSync extends Sync {
      private static final long serialVersionUID = 7316153563782823691L;

      /**
       * Performs lock.  Try immediate barge, backing up to normal
       * acquire on failure.
       */
      final void lock() {
          if (compareAndSetState(0, 1))
              setExclusiveOwnerThread(Thread.currentThread());
          else
              acquire(1);
      }

      protected final boolean tryAcquire(int acquires) {
          return nonfairTryAcquire(acquires);
      }
  }
```
这里lock的时候直接通过CAS来获得锁,如果成功就把当前线程设为独占线程,失败的话进入 *acquire* 方法.这个方法是在 *AbstractQueuedSynchronizer* 实现的通用方法.
```java
public final void acquire(int arg) {
       if (!tryAcquire(arg) &&
           acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
           selfInterrupt();
   }
```

先看见if判断的第一个条件 *tryAcquire* ,就是上边的 *NonfairSync* 中的实现,最终进入了 *nonfairTryAcquire* 方法中.
```java
final boolean nonfairTryAcquire(int acquires) {
           final Thread current = Thread.currentThread();
           int c = getState();
           if (c == 0) {
               if (compareAndSetState(0, acquires)) {
                   setExclusiveOwnerThread(current);
                   return true;
               }
           }
           else if (current == getExclusiveOwnerThread()) {
               int nextc = c + acquires;
               if (nextc < 0) // overflow
                   throw new Error("Maximum lock count exceeded");
               setState(nextc);
               return true;
           }
           return false;
       }
```
这里的state就是前面说的计数器,通过state来说明当前锁的状态.
```java
/**
    * The synchronization state.
    */
   private volatile int state;

```
这里如果state为0,说明还没有线程获得了锁,那正好,我是第一个,立马通过CAS来把自己设为当前独占线程,同时state变为传入的 *acquires* ,第一次的时候为1,后续每次调用都会加一.
如果当前state不为0,但是独占线程是自己的话,把state加一,可重入体现在这里了.这里很明显最大可重入的值是最大的int值.
最后,如果都不符合返回false,进入上面 *acquire* 方法if判断的第二个条件 *acquireQueued(addWaiter(Node.EXCLUSIVE), arg)*.先看其中的 *addWaiter* 方法:
```java
/**
     * Creates and enqueues node for current thread and given mode.
     *
     * @param mode Node.EXCLUSIVE for exclusive, Node.SHARED for shared
     * @return the new node
     */
    private Node addWaiter(Node mode) {
        Node node = new Node(Thread.currentThread(), mode);
        // Try the fast path of enq; backup to full enq on failure
        Node pred = tail;
        if (pred != null) {
            node.prev = pred;
            if (compareAndSetTail(pred, node)) {
                pred.next = node;
                return node;
            }
        }
        enq(node);
        return node;
    }

```
如果锁已经被抢占了,后续线程将进入这里维护的 *双向队列* 中, *addWaiter* 方法就是把节点对象放入队列中的实现.
如果队列不为空的话,直接通过CAS把结尾节点更新为当前请求的节点对象,否者会进入一个队列初始化方法 *enq* 方法中.
```java
/**
    * Inserts node into queue, initializing if necessary. See picture above.
    * @param node the node to insert
    * @return node's predecessor
    */
   private Node enq(final Node node) {
       for (;;) {
           Node t = tail;
           if (t == null) { // Must initialize
               if (compareAndSetHead(new Node()))
                   tail = head;
           } else {
               node.prev = t;
               if (compareAndSetTail(t, node)) {
                   t.next = node;
                   return t;
               }
           }
       }
   }
```
如果尾部指针为空,说明队列没有初始化,把head和tail节点更新为一个创建的新的node对象
如果队列不为空,更新尾部节点,把当前请求的节点设为尾部节点.

回到 *acquireQueued(final Node node, int arg)* 方法中.
```java
final boolean acquireQueued(final Node node, int arg) {
       boolean failed = true;
       try {
           boolean interrupted = false;
           for (;;) {
               final Node p = node.predecessor();
               if (p == head && tryAcquire(arg)) {
                   setHead(node);
                   p.next = null; // help GC
                   failed = false;
                   return interrupted;
               }
               if (shouldParkAfterFailedAcquire(p, node) &&
                   parkAndCheckInterrupt())
                   interrupted = true;
           }
       } finally {
           if (failed)
               cancelAcquire(node);
       }
   }
```
首先是判断当前节点是不是头结点的后一个,同时再次申请锁,如果两个条件都满足就更新队列,把头结点置为当前的请求节点,然后返回false
如果不满足if条件就进入第二个判断,先看其第一个条件 *shouldParkAfterFailedAcquire(p, node)* :
```java
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        int ws = pred.waitStatus;
        if (ws == Node.SIGNAL)
            /*
             * This node has already set status asking a release
             * to signal it, so it can safely park.
             */
            return true;
        if (ws > 0) {
            /*
             * Predecessor was cancelled. Skip over predecessors and
             * indicate retry.
             */
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
            pred.next = node;
        } else {
            /*
             * waitStatus must be 0 or PROPAGATE.  Indicate that we
             * need a signal, but don't park yet.  Caller will need to
             * retry to make sure it cannot acquire before parking.
             */
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
        return false;
    }
```
默认node对象的waitStatus是0,进入最后面的else分支,返回false,由于外面的调用方法 *acquireQueued* 里面是个死循环,所以马上又会回到这里,然后更新waitStatus的值为 Node.SIGNAL, 返回true.
在看第二个判断条件 *parkAndCheckInterrupt()* :
```java
/**
   * Convenience method to park and then check if interrupted
   *
   * @return {@code true} if interrupted
   */
  private final boolean parkAndCheckInterrupt() {
      LockSupport.park(this);
      return Thread.interrupted();
  }
```
这里,就进入了阻塞线程的核心代码了,通过 *LockSupport.park(this);* 来阻塞当前的线程.至于 *LockSupport* 的用法,以后有时间再解释,只要明白这个就是阻塞现场的实现就ok了.
非公平锁的lock实现就到这里,再来简单看下公平锁的lock的实现,其实只是 *tryAcquire* 方法的实现不一样.

```java
/**
        * Fair version of tryAcquire.  Don't grant access unless
        * recursive call or no waiters or is first.
        */
       protected final boolean tryAcquire(int acquires) {
           final Thread current = Thread.currentThread();
           int c = getState();
           if (c == 0) {
               if (!hasQueuedPredecessors() &&
                   compareAndSetState(0, acquires)) {
                   setExclusiveOwnerThread(current);
                   return true;
               }
           }
           else if (current == getExclusiveOwnerThread()) {
               int nextc = c + acquires;
               if (nextc < 0)
                   throw new Error("Maximum lock count exceeded");
               setState(nextc);
               return true;
           }
           return false;
       }
```
这里公平锁首先还是根据state来获取锁,只不过增加了一个条件 *hasQueuedPredecessors()*  ,也就是说判断队列前面还有没有正在等待的节点了
```java
public final boolean hasQueuedPredecessors() {
       // The correctness of this depends on head being initialized
       // before tail and on head.next being accurate if the current
       // thread is first in queue.
       Node t = tail; // Read fields in reverse initialization order
       Node h = head;
       Node s;
       return h != t &&
           ((s = h.next) == null || s.thread != Thread.currentThread());
   }

```
这里就是上面提到的公平锁和非公平锁的请求实现方式的区别所在了.

lock流程就到了这里结束了,其实不算复杂.下面是unlock方法的实现:
unlock在公平锁和非公平锁中的实现是一样的
```java
/**
    * Releases in exclusive mode.  Implemented by unblocking one or
    * more threads if {@link #tryRelease} returns true.
    * This method can be used to implement method {@link Lock#unlock}.
    *
    * @param arg the release argument.  This value is conveyed to
    *        {@link #tryRelease} but is otherwise uninterpreted and
    *        can represent anything you like.
    * @return the value returned from {@link #tryRelease}
    */
   public final boolean release(int arg) {
       if (tryRelease(arg)) {
           Node h = head;
           if (h != null && h.waitStatus != 0)
               unparkSuccessor(h);
           return true;
       }
       return false;
   }

```
先来看 *tryRelease* 方法:
```java
protected final boolean tryRelease(int releases) {
          int c = getState() - releases;
          if (Thread.currentThread() != getExclusiveOwnerThread())
              throw new IllegalMonitorStateException();
          boolean free = false;
          if (c == 0) {
              free = true;
              setExclusiveOwnerThread(null);
          }
          setState(c);
          return free;
      }
```

只有当前是独占线程,并且计数器state为0的时候,释放锁,把独占线程对象置为空,同时state清零.free返回true
上面进入if判断后,调用 *unparkSuccessor* 方法.
```java

    /**
     * Wakes up node's successor, if one exists.
     *
     * @param node the node
     */
    private void unparkSuccessor(Node node) {
        /*
         * If status is negative (i.e., possibly needing signal) try
         * to clear in anticipation of signalling.  It is OK if this
         * fails or if status is changed by waiting thread.
         */
        int ws = node.waitStatus;
        if (ws < 0)
            compareAndSetWaitStatus(node, ws, 0);

        /*
         * Thread to unpark is held in successor, which is normally
         * just the next node.  But if cancelled or apparently null,
         * traverse backwards from tail to find the actual
         * non-cancelled successor.
         */
        Node s = node.next;
        if (s == null || s.waitStatus > 0) {
            s = null;
            for (Node t = tail; t != null && t != node; t = t.prev)
                if (t.waitStatus <= 0)
                    s = t;
        }
        if (s != null)
            LockSupport.unpark(s.thread);
    }
```
这里设置当前节点的waitStatus为0,同时获取node的下一个节点,并且唤醒该节点的线程.那么问题来了,唤醒后面节点的线程之后,如何保证队列的顺序呢,同时把已经执行完的的node移除呢.还记得上面 *acquireQueued* 的方法吗,内部有个死循环
```java
final boolean acquireQueued(final Node node, int arg) {
       boolean failed = true;
       try {
           boolean interrupted = false;
           for (;;) {
               final Node p = node.predecessor();
               if (p == head && tryAcquire(arg)) {
                   setHead(node);
                   p.next = null; // help GC
                   failed = false;
                   return interrupted;
               }
               if (shouldParkAfterFailedAcquire(p, node) &&
                   parkAndCheckInterrupt())
                   interrupted = true;
           }
       } finally {
           if (failed)
               cancelAcquire(node);
       }
   }
```
这里p节点确实是head,进入里面第一个if判断,这个时候会把头结点更新,然后旧的头结点由于没有关联了会在后面被gc回收掉,这样就达到了动态维护阻塞队列的目的.

#### ReentrantLock的用法
除了普通的加锁方式之外, ReentrantLock还提供了 tryLock(),tryLock(int mill,TimeUtil util),lockInterruptibly()方法等,下面分别解释一下:
*tryLock()* 判断能否获取获取到锁,如果当前锁是空置状态,直接返回true,否者返回false
*tryLock(int mill,TimeUtil unit)* 类似上面的tryLock,只不过加入了时间控制,也就说说在指定的时间内能否获得到锁.
*lockInterruptibly* 支持中断的获取锁,如果在获取锁的过程中线程被中断,会直接返回已给中断异常.它与普通lock方法的区别在于,如果是 *lock* 方法,会忽略中断请求,继续获取锁直到的成功,而 *lockInterruptibly* 方法会直接响应中断操作,返回一个中断异常交由上层处理.如果要求中断操作线程不能参与锁的竞争可以使用 *lockInterruptibly* ,否则使用 *lock* 方法.

除了上面说的这几个方法,平时常用的还有Condition方式,也就是更加灵活的唤醒阻塞方式,对应Object的wait,notify,notifiyAll等方法.
Condition 提供了 await,signal,signalAll等三个方法.可以用来灵活调整唤醒的次序,下面通过一个面试题来说明Condition的用法.

> 如何让两个线程交替答应A和B十次,输出结果为ABABABABABABABABABAB

思路:我们可以让保持一个index,每次答应之后加一,如果是偶数,唤醒A线程先答应A字符,然后在A中唤醒B线程,如果是奇数唤醒B线程答应A,然后在B中唤醒A线程.
```java
public class TestReentrantLockCondition3 {
    static Lock lock = new ReentrantLock();
    static Condition condition = lock.newCondition();

    int index;

    public void printFirst() {
        lock.lock();
        try {
            while (index % 2 != 0) {
                condition.await();
            }
            System.out.print("A");
            index++;
            condition.signal();
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
    }

    public void printLast() {
        lock.lock();
        try {
            while (index % 2 != 1) {
                condition.await();
            }
            System.out.print("B");
            index++;
            condition.signal();
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
    }

    static class MyThread extends Thread {
        private TestReentrantLockCondition3 condition3;

        public MyThread(TestReentrantLockCondition3 condition3) {
            this.condition3 = condition3;
        }

        @Override
        public void run() {
            condition3.printFirst();
        }
    }

    static class MyThread2 extends Thread {
        private TestReentrantLockCondition3 condition3;

        public MyThread2(TestReentrantLockCondition3 condition3) {
            this.condition3 = condition3;
        }

        @Override
        public void run() {
            condition3.printLast();
        }
    }

    public static void main(String[] args) {
        TestReentrantLockCondition3 condition3 = new TestReentrantLockCondition3();
        for (int i = 0; i < 10; i++) {
            new MyThread(condition3).start();
            new MyThread2(condition3).start();
        }
    }
  }
```

核心就在上面的 *printFirst* 和 *printLast* 两个方法,在index为偶数的情况下A唤醒B阻塞,反之,然后通过不断的更新index来达到交替打印的逻辑.

#### java 线程join方法
join方法的含义是等待对应的时长直到调用的线程执行完成.
也就是说,在调用的线程没有完成的时候,当前线程处于阻塞状态,不会继续往下执行,直到调用的join方法的线程完成或者超时.

来看源码:
```
public final void join() throws InterruptedException {
       join(0);
   }
```
```java
public final synchronized void join(long millis)
   throws InterruptedException {
       long base = System.currentTimeMillis();
       long now = 0;

       if (millis < 0) {
           throw new IllegalArgumentException("timeout value is negative");
       }

       if (millis == 0) {
           while (isAlive()) {
               wait(0);
           }
       } else {
           while (isAlive()) {
               long delay = millis - now;
               if (delay <= 0) {
                   break;
               }
               wait(delay);
               now = System.currentTimeMillis() - base;
           }
       }
   }
```
如果当前线程不是Alive的话,join方法无效,如果是alive的话等待.
下面通过一个demo来说明用法:
```java
public class TestJoin {
    static class TestThread extends Thread{
        @Override
        public void run() {
            System.out.println("I am TestThread !!!!");
            try {
                Thread.sleep(2000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println("TestThread done");
        }
    }
    static class TestThread2 extends Thread{
        @Override
        public void run() {
            System.out.println("I am TestThread2 !!!!");
            try {
                Thread.sleep(2000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println("TestThread2 done");
        }
    }

    public static void main(String[] args) {
        Thread thread1 = new TestThread();
        Thread thread2 = new TestThread2();
        thread1.start();
        try {
            thread1.join();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        thread2.start();


    }
}
```
输出:
```java
I am TestThread !!!!
TestThread done
I am TestThread2 !!!!
TestThread2 done
```

#### ReentrantReadWriteLock用法和原理
首先 *ReentrantReadWriteLock* 和 *ReentrantLock* 的唯一关联就是都是 *AbstractQueuedSynchronizer* 的实现,其他就没有任何联系了,不要认为二者是继承关系.
*ReentrantReadWriteLock* 主要是内部分别包含该一个读锁和写锁,主要是用的大量读少量写的时候保证性能时使用,毕竟如果直接 *ReentrantLock* 的话就是读写都加锁,性能上会有写影响.在 *ReentrantReadWriteLock* 中写锁基本类似于 *ReentrantLock* 是一种独占锁,也就是说同一时间只能有一个线程来写.而 *ReentrantReadWriteLock* 中的读锁是一种共享锁,可以有多个线程来同时读取数据.
读写锁的目的说白了就是为了保证数据一致性,在大量读取少量修改的时候,为了节约性能,就通过加读写锁的方式来解决独占锁的长时间的占用问题.从而达到数据一致的目的.

读与写的机制:
"读-读" 不互斥
"读-写" 互斥
"写-写" 互斥

即在任何时候必须保证:
只有一个线程在写入
线程正在读取的时候,写入操作等待
线程正在写入的时候,其他线程的写入操作和读取操作都要等待;

锁降级:写线程可以在写入锁后可以获取读取锁,然后释放写入锁,这样就从写入锁变成了读取锁,从而实现了锁降级的特性

下面是读写锁的一个使用场景实现:
```java
private final ReadWriteLock lock = new ReentrantReadWriteLock();//读写锁  
// 读取缓存：  
    public Object read(String key) {  
        lock.readLock().lock();  
        Object obj = null;  
        try {  
            obj = cache.get(key);  
            if (obj == null) {  
                lock.readLock().unlock();  
                // 在这里的时候，其他的线程有可能获取到锁  
                lock.writeLock().lock();  
                try {  
                    if (obj == null) {  
                        obj = "查找数据库"; // 实际动作是查找数据库  
                        // 把数据更新到缓存中：  
                        cache.put(key, obj);  
                    }  
                } finally {  
                    // 当前线程在获取到写锁的过程中，可以获取到读锁，这叫锁的重入，然后导致了写锁的降级，称为降级锁。  
                    // 利用重入可以将写锁降级，但只能在当前线程保持的所有写入锁都已经释放后，才允许重入 reader使用  
                    // 它们。所以在重入的过程中，其他的线程不会有获取到锁的机会（这样做的好处）。试想，先释放写锁，在  
                    // 上读锁，这样做有什么弊端？--如果这样做，那么在释放写锁后，在得到读锁前，有可能被其他线程打断。  
                    // 重入————>降级锁的步骤：先获取写入锁，然后获取读取锁，最后释放写入锁（重点）  
                    lock.readLock().lock();   
                    lock.writeLock().unlock();  
                }  
            }  
        } finally {  
            lock.readLock().unlock();  
        }  
        return obj;  
    }  
```
读写锁的写锁和上面的降到的ReentrantLock的实现类似这里就不展开了,重点讲下读锁也就是共享锁的实现,共享锁其实严格来说不能算作锁,其内部只是维护了一个计算器.
```java
public void lock() {
           sync.acquireShared(1);
       }
```
这里的 *acquireShared* 属于AQS中的实现:
```java
public final void acquireShared(int arg) {
       if (tryAcquireShared(arg) < 0)
           doAcquireShared(arg);
   }
```
```java
protected final int tryAcquireShared(int unused) {
          /*
           * Walkthrough:
           * 1. If write lock held by another thread, fail.
           * 2. Otherwise, this thread is eligible for
           *    lock wrt state, so ask if it should block
           *    because of queue policy. If not, try
           *    to grant by CASing state and updating count.
           *    Note that step does not check for reentrant
           *    acquires, which is postponed to full version
           *    to avoid having to check hold count in
           *    the more typical non-reentrant case.
           * 3. If step 2 fails either because thread
           *    apparently not eligible or CAS fails or count
           *    saturated, chain to version with full retry loop.
           */
          Thread current = Thread.currentThread();
          int c = getState();
          if (exclusiveCount(c) != 0 &&
              getExclusiveOwnerThread() != current)
              return -1;
          int r = sharedCount(c);
          if (!readerShouldBlock() &&
              r < MAX_COUNT &&
              compareAndSetState(c, c + SHARED_UNIT)) {
              if (r == 0) {
                  firstReader = current;
                  firstReaderHoldCount = 1;
              } else if (firstReader == current) {
                  firstReaderHoldCount++;
              } else {
                  HoldCounter rh = cachedHoldCounter;
                  if (rh == null || rh.tid != getThreadId(current))
                      cachedHoldCounter = rh = readHolds.get();
                  else if (rh.count == 0)
                      readHolds.set(rh);
                  rh.count++;
              }
              return 1;
          }
          return fullTryAcquireShared(current);
      }
```
可以看到首先是判断当前有没有写锁占用着并且占用的线程不是当前线程,直接返回失败
*readerShouldBlock* 是一个抽象方法,在公平锁和非公平锁中实现方式不一样
这是公平锁的
```java
final boolean readerShouldBlock() {
           return hasQueuedPredecessors();
}
public final boolean hasQueuedPredecessors() {
       // The correctness of this depends on head being initialized
       // before tail and on head.next being accurate if the current
       // thread is first in queue.
       Node t = tail; // Read fields in reverse initialization order
       Node h = head;
       Node s;
       return h != t &&
           ((s = h.next) == null || s.thread != Thread.currentThread());
   }
```
这是非公平锁的
```java
final boolean readerShouldBlock() {
           /* As a heuristic to avoid indefinite writer starvation,
            * block if the thread that momentarily appears to be head
            * of queue, if one exists, is a waiting writer.  This is
            * only a probabilistic effect since a new reader will not
            * block if there is a waiting writer behind other enabled
            * readers that have not yet drained from the queue.
            */
return apparentlyFirstQueuedIsExclusive();
}
final boolean apparentlyFirstQueuedIsExclusive() {
     Node h, s;
     return (h = head) != null &&
         (s = h.next)  != null &&
         !s.isShared()         &&
         s.thread != null;
 }
```
 ,第二个条件是r得小于最大的读锁限制,最大的读锁的限制是65535.因为读写锁是把state的高16位存放读锁数量,后16位存放写锁数量.然后通过CAS来更新计数器.
HoldCounter是这样一个对象,内部包含该了线程ID和线程中的读锁计数,然后通过ThreadLocal *readHolds* 来对应不同的线程.

```java
static final class HoldCounter {
          int count = 0;
          // Use id, not reference, to avoid garbage retention
          final long tid = getThreadId(Thread.currentThread());
      }
```
```java
static final class ThreadLocalHoldCounter
         extends ThreadLocal<HoldCounter> {
         public HoldCounter initialValue() {
             return new HoldCounter();
         }
     }
       private transient ThreadLocalHoldCounter readHolds;
```

如果上面的判断条件成功放回1,否者进入下面的 *fullTryAcquireShared* ;
```java
final int fullTryAcquireShared(Thread current) {
          /*
           * This code is in part redundant with that in
           * tryAcquireShared but is simpler overall by not
           * complicating tryAcquireShared with interactions between
           * retries and lazily reading hold counts.
           */
          HoldCounter rh = null;
          for (;;) {
              int c = getState();
              if (exclusiveCount(c) != 0) {
                  if (getExclusiveOwnerThread() != current)
                      return -1;
                  // else we hold the exclusive lock; blocking here
                  // would cause deadlock.
              } else if (readerShouldBlock()) {
                  // Make sure we're not acquiring read lock reentrantly
                  if (firstReader == current) {
                      // assert firstReaderHoldCount > 0;
                  } else {
                      if (rh == null) {
                          rh = cachedHoldCounter;
                          if (rh == null || rh.tid != getThreadId(current)) {
                              rh = readHolds.get();
                              if (rh.count == 0)
                                  readHolds.remove();
                          }
                      }
                      if (rh.count == 0)
                          return -1;
                  }
              }
              if (sharedCount(c) == MAX_COUNT)
                  throw new Error("Maximum lock count exceeded");
              if (compareAndSetState(c, c + SHARED_UNIT)) {
                  if (sharedCount(c) == 0) {
                      firstReader = current;
                      firstReaderHoldCount = 1;
                  } else if (firstReader == current) {
                      firstReaderHoldCount++;
                  } else {
                      if (rh == null)
                          rh = cachedHoldCounter;
                      if (rh == null || rh.tid != getThreadId(current))
                          rh = readHolds.get();
                      else if (rh.count == 0)
                          readHolds.set(rh);
                      rh.count++;
                      cachedHoldCounter = rh; // cache for release
                  }
                  return 1;
              }
          }
      }
```
这里通过自旋的方式来不断的获取读锁的状态,并通过CAS来更新.
最后是释放锁:
```java
protected final boolean tryReleaseShared(int unused) {
           Thread current = Thread.currentThread();
           if (firstReader == current) {
               // assert firstReaderHoldCount > 0;
               if (firstReaderHoldCount == 1)
                   firstReader = null;
               else
                   firstReaderHoldCount--;
           } else {
               HoldCounter rh = cachedHoldCounter;
               if (rh == null || rh.tid != getThreadId(current))
                   rh = readHolds.get();
               int count = rh.count;
               if (count <= 1) {
                   readHolds.remove();
                   if (count <= 0)
                       throw unmatchedUnlockException();
               }
               --rh.count;
           }
           for (;;) {
               int c = getState();
               int nextc = c - SHARED_UNIT;
               if (compareAndSetState(c, nextc))
                   // Releasing the read lock has no effect on readers,
                   // but it may allow waiting writers to proceed if
                   // both read and write locks are now free.
                   return nextc == 0;
           }
       }
```
#### 脏读,幻读,不可重复读
脏读:是指当一个事物正在访问数据,对数据进行了修改,而这种事物还没有提交到数据库,这时另一个事物访问了该数据,然后使用了就得数据
不可重复读:是指在一个事物,多次访问同一事物,两次结果不一致.比如在一个事物中先读取了一条数据,这时另一个数据对这条数据进行了修改并提交,这时前面的事物如果再次读取同一数据,发现两次的事物不一致,这就是不可重复读
幻读:是指事物对数据进行两次查询,第二次的查询结果包含第一次查询中没有的数据后者少了第一次查询中出现的数据.这是因为在两次查询过程中另外一个事物进行了插入操作

#### 死锁
死锁问题是多线程特有的问题,在死锁时,线程间相互等待资源,而又不释放自身资源,导致无穷尽的等待,其结果时系统任务永远无法执行完成.
一般来说,要出现死锁必须满足下面几个条件:
1. 互斥条件:一个资源一次只能被一个线程使用
2. 请求与保持条件:一个线程因请求资源而阻塞时,对已获得资源保持不妨
3. 不剥夺条件:线程已获得的资源,在未使用完之前,不强行剥夺
4. 循环等待条件:若干线程之间形成了一种头尾相连的等待循环关系

如果要打破死锁,只要打破4个必要条件中一个或多个就可以了.
如何避免死锁:
1. 让程序每次至多获取一个锁.当然,在多线程环境下,这种情况通常不现实
2. 设计时考虑清楚锁的顺序,尽量减少嵌套的加锁数量
3. 既然死锁的是两个线程无线等待对方持有的所,那只要设置一个等待时间上限就可以了,可以使用Lock类中tryLock方法来尝试获得所,在超时之后返回获取失败

#### java信号量Semaphore
java信号量有点像"许可证集合",比如银行排队有3个窗口(permit),但是有10个人排队,那这里窗口就相当于许可证,每当一个窗口处理完成后,释放(realse)出一个许可证,排队的人进入一个窗口,知道所有排队的人完成业务处理.
下面通过一个demo来说明用法:
```java
public class TestSemaphore {
    public static void main(String[] args) {
        Semaphore semaphore = new Semaphore(5);
        for (int i = 0; i < 20; i++) {
            final int NO = i;
            new Thread(new Runnable() {
                @Override
                public void run() {
                    try {
                        semaphore.acquire();
                        System.out.println("Access No =" + NO);
                        Thread.sleep((long)(Math.random()*1000));
                        semaphore.release();
                        System.out.println("Semaphore availiable =" + semaphore.availablePermits());
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }

                }
            }).start();
        }
    }
}
```
输出
```
Access No =1
Access No =2
Access No =0
Access No =3
Access No =4
Semaphore availiable =1
Access No =5
Semaphore availiable =1
Access No =6
Semaphore availiable =1
Access No =7
Semaphore availiable =1
Access No =8
Semaphore availiable =1
Access No =9
Semaphore availiable =1
Access No =10
Semaphore availiable =1
Access No =11
Semaphore availiable =1
Access No =12
Semaphore availiable =1
Access No =13
Semaphore availiable =1
Access No =14
Semaphore availiable =1
Access No =15
Semaphore availiable =1
Access No =16
Semaphore availiable =1
Access No =17
Semaphore availiable =1
Access No =18
Semaphore availiable =1
Access No =19
Semaphore availiable =1
Semaphore availiable =2
Semaphore availiable =3
Semaphore availiable =4
Semaphore availiable =5
```
用法非常简单,有点类似线程池的实现

#### java 多线程包下的常用类使用
##### CountDownLatch
CountDownLatch是指在其内部计数器不为0的情况下,运行把制定的线程阻塞并等待,直到内部计数器为0后,才唤醒阻塞的线程
```java
public class TestCountdownLatch {

    static class Thread1 extends Thread{
        private CountDownLatch latch;

        public Thread1(CountDownLatch latch) {
            this.latch = latch;
        }

        @Override
        public void run() {
            try {
                Thread.sleep(2000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            latch.countDown();
            System.out.println("latch down  "+latch.getCount());
        }
    }

    public static void main(String[] args) throws InterruptedException {
        CountDownLatch latch = new CountDownLatch(3);
        for (int i = 0; i < 3; i++) {
            Thread1 thread1 = new Thread1(latch);
            thread1.start();
            thread1.join();
        }
        latch.await();
        System.out.println("done!!!!");
    }
}
```
输出:
```
latch down  2
latch down  1
latch down  0
done!!!!
```
##### BlockingQueue阻塞队列
阻塞队列是一个支持两种附加超值的队列.这两个附加操作是: 在队列为空的时候,获取元素的线程会一直等待队列变为非空.当队列满时,存储元素的线程会等待队列可用.一个典型的生产者消费者的应用场景.
阻塞队列提供了下面四种处理方式:

方法\处理方法  |抛出异常   |返回特殊值   | 一直阻塞  | 超时退出
--|---|---|---|--
  插入方法|add(e)   | offer(e)  | put(e)  | offer(e,time,unit)
  移除方法|remove   | poll()  | take()  | poll(time,unit)
  检查方法| element()  | peek()  | 不可用  | 不可用

#### 生产者消费者队列
生产者消费者队列有两种常用的实现,一种是通过对象本身的wait和notify来实现,另一种是通过阻塞队列来实现
先看第一种方式:
```java
public class Channel {
    private Queue<Good> goodList = new LinkedList<>();

    public synchronized Good get() {
        if (goodList.size() == 0) {
            return null;
        }
        return goodList.remove();
    }

    public synchronized void put(Good good) {
        goodList.add(good);
        notify();
    }
}
public class Good {
    private String name;

    public Good(String name) {
        this.name = name;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }
}
```
Channel表示仓库的含义,good表示里面放入的物品,在put的时候唤醒
```java
public class Consumer implements Runnable{
    private String name;
    private Channel channel;

    public Consumer(String name, Channel channel) {
        this.name = name;
        this.channel = channel;
    }

    @Override
    public void run() {
        while (true) {
            Good good = channel.get();
            if (good != null) {
                System.out.println(name + "获得商品: " + good.getName());
            } else {
                synchronized (channel) {
                    try {
                        System.out.println(name + " 进入等待 ");
                        channel.wait();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }
        }
    }
}
```
消费者不断的从仓库中获取,如果发现没有了,阻塞进入等待  
```java
public class Producer implements Runnable {

    private static volatile int goodNumber = 0;
    private String name;
    private Channel channel;

    public Producer(String name, Channel channel) {
        this.name = name;
        this.channel = channel;
    }

    @Override
    public void run() {
        while (true) {
            int sleep = new Random().nextInt(2000);
            try {
                Thread.sleep(sleep);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            Good good = new Good("商品-编号" + (++goodNumber));
            System.out.println(name + "生成商品:" + good.getName());
            channel.put(good);
        }
    }
}
```
生产者不断的放入物品到仓库中
```java
public class Tester {
    public static void main(String[] args) {
        Channel channel = new Channel();
        new Thread(new Producer("生成者1", channel)).start();
        new Thread(new Producer("生成者2", channel)).start();
        new Thread(new Producer("生成者3", channel)).start();

        new Thread(new Consumer("消费者1",channel)).start();
        new Thread(new Consumer("消费者2",channel)).start();
        new Thread(new Consumer("消费者3",channel)).start();
    }

}
```
输出:

```
消费者1 进入等待
消费者3 进入等待
消费者2 进入等待
生成者2生成商品:商品-编号1
消费者1获得商品: 商品-编号1
消费者1 进入等待
生成者3生成商品:商品-编号2
消费者3获得商品: 商品-编号2
消费者3 进入等待
生成者1生成商品:商品-编号3
消费者2获得商品: 商品-编号3
消费者2 进入等待
生成者3生成商品:商品-编号4
消费者1获得商品: 商品-编号4
消费者1 进入等待
生成者3生成商品:商品-编号5
消费者3获得商品: 商品-编号5
消费者3 进入等待
生成者2生成商品:商品-编号6
消费者2获得商品: 商品-编号6
消费者2 进入等待
生成者3生成商品:商品-编号7
消费者1获得商品: 商品-编号7
消费者1 进入等待
生成者2生成商品:商品-编号8
消费者3获得商品: 商品-编号8
消费者3 进入等待
生成者1生成商品:商品-编号9
消费者2获得商品: 商品-编号9
消费者2 进入等待
生成者3生成商品:商品-编号10
消费者1获得商品: 商品-编号10
消费者1 进入等待
生成者1生成商品:商品-编号11
消费者3获得商品: 商品-编号11
消费者3 进入等待
生成者1生成商品:商品-编号12
消费者2获得商品: 商品-编号12
消费者2 进入等待
生成者2生成商品:商品-编号13
消费者1获得商品: 商品-编号13
消费者1 进入等待
生成者3生成商品:商品-编号14
消费者3获得商品: 商品-编号14
消费者3 进入等待
生成者2生成商品:商品-编号15
消费者2获得商品: 商品-编号15
消费者2 进入等待
生成者1生成商品:商品-编号16
消费者1获得商品: 商品-编号16
消费者1 进入等待
生成者3生成商品:商品-编号17
消费者3获得商品: 商品-编号17
消费者3 进入等待
生成者1生成商品:商品-编号18
消费者2获得商品: 商品-编号18
消费者2 进入等待
生成者2生成商品:商品-编号19
消费者1获得商品: 商品-编号19
消费者1 进入等待
生成者3生成商品:商品-编号20
```

第二种方式,通过阻塞队列来实现:
```java
public class TestBlockingQueue {
    static class Consumer extends Thread{
        private BlockingQueue<Integer> blockingQueue;

        public Consumer(BlockingQueue<Integer> blockingQueue) {
            this.blockingQueue = blockingQueue;
        }

        @Override
        public void run() {
            while (true) {
                try {
                   int i = blockingQueue.take();
                    System.out.println("get "+i);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }
    }
    static class Producer extends Thread{
        private BlockingQueue<Integer> blockingQueue;

        public Producer(BlockingQueue<Integer> blockingQueue) {
            this.blockingQueue = blockingQueue;
        }

        @Override
        public void run() {
            try {
                for (int i = 0; i < 10; i++) {
                    System.out.println("put " + i);
                    blockingQueue.put(i);
                }
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }

    public static void main(String[] args) {
        BlockingQueue<Integer> queue = new LinkedBlockingQueue<>();
        new Consumer(queue).start();
        new Producer(queue).start();
    }
}
```
