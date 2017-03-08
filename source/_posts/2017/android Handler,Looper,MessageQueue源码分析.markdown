layout: "post"
title: "android Handler,Looper,MessageQueue源码分析"
date: "2017-01-03 14:09"
---
## android Handler,Looper,MessageQueue源码分析

  Handler是一个在android开发过程中经常用到的类,同时在面试的时候也会经常问道其中的实现原理,那今天就重点讲解一下Handler的用法和实现原理.
  >一个Handler可以使你发送并处理一个消息或者线程通过关联一个 *MessageQueue* .每一个handler的实例都唯一关联一个线程和 *MessageQueue* .当你创建一个Handler它就和当前线程和线程的 *MessageQueue* 绑定在一起了.它会发送消息或者线程到对应 *MessageQueue* ,并且在从 *MessageQueue* 取出的时候执行它们.
  >Handler主要有两个作用:一个是可以在未来的某个时间点执行消息或者线程 ;另一个是在你当前拥有的另一个线程中排队执行一些任务.
  >发送消息一般可以使用 post(Runnable),postAtTime(Runnable,long),postDelay(Runnable,long),sendEmptyMessage(int),sendMessage(Message),sendMessageAtTime(Message,long) 和 sendMessageDelay(Message,long).这几个方法.post开头的方法可以在消息队列入队的时候调用.而sendMessage开头的方法允许把一个带有bundle对象的消息对象入队,并在未来交给 *handleMessage(Message)* 去处理.

  上面这个是我照着官方文档翻译的,水平有限,说白了就是Handler可以把消息对象或者线程放入消息队列中,等到出列的时候调用.
  比如当子线程中发送网络请求之后回调可以通过给主线程的handler发送一个message,这样可以把网络请求的回调结果返回到主线程中处理.

  下面开始分析Handler的源码:

### Handler源码分析
  我们先从Handler的使用入口来看,也就是那一堆post和sendMessage开头的方法入手,可以看到所有的传递事件的方法最终都是调用了 *sendMessageAtTime* :


  ```java
  public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
        MessageQueue queue = mQueue;
        if (queue == null) {
            RuntimeException e = new RuntimeException(
                    this + " sendMessageAtTime() called with no mQueue");
            Log.w("Looper", e.getMessage(), e);
            return false;
        }
        return enqueueMessage(queue, msg, uptimeMillis);
    }
  ```

  post方法开头的只是多做了一部处理,把Runnable对象作为message的回调使用了,然后也是调用了 *sendMessageAtTime* :
  ```java

    private static Message getPostMessage(Runnable r) {
        Message m = Message.obtain();
        m.callback = r;
        return m;
    }

  ```
  可以看到在 *sendMessageAtTime* 的方法刚开始的时候显示判断消息队列 *MessageQueue* 是否为空,如果为空的直接返回false不予执行.那这个消息队列是从哪里来的的呢?这个时候回到Handler的构造方法:
  ```java
  public Handler() {
       this(null, false);
   }
   public Handler(Callback callback) {
       this(callback, false);
   }
   public Handler(Looper looper) {
       this(looper, null, false);
   }
   public Handler(Looper looper, Callback callback) {
      this(looper, callback, false);
  }
  /**
  *.......
  *@hide
  */
  public Handler(boolean async) {
        this(null, async);
  }
  public Handler(Callback callback, boolean async) {
       if (FIND_POTENTIAL_LEAKS) {
           final Class<? extends Handler> klass = getClass();
           if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
                   (klass.getModifiers() & Modifier.STATIC) == 0) {
               Log.w(TAG, "The following Handler class should be static or leaks might occur: " +
                   klass.getCanonicalName());
           }
       }

       mLooper = Looper.myLooper();
       if (mLooper == null) {
           throw new RuntimeException(
               "Can't create handler inside thread that has not called Looper.prepare()");
       }
       mQueue = mLooper.mQueue;
       mCallback = callback;
       mAsynchronous = async;
  }

  ```
  这里有个细节就是6个构造函数的其中一个是不对外开放的,就是那个加了 *@hide* 注解的构造函数.也就是这个构造函数只能在手机自己内部调用,和android的internal包戏下面的代码一样,都是不对外开发的.

  可以看到在构造方法顶部有一个tip提示,就是说如果这里handler不是静态的话可能导致内存泄漏,这种情况是很常见的,内部类持有外部类的引用,也就是当前handler持有外部的activity对象,当activity销毁的时候,如果handler中还是有任务没有完成的,activity是无法被内存回收的.这就会导致内存泄漏,解决方法就是把handler设置为static并且把activity的关联设置为弱引用(WeakReference).

  继续往下看,开始调用 Looper.myLooper() 如果Looper对象为空的话抛出异常,看来handler必须能够获取到looper对象,否者没法往下执行.到了后面我们就知道,looper的作用是用来不停的从消息队列中取出消息来交给Handler来执行的.所以没有looper的话handler获取不到消息就没有了意义.
  在往下就会发现handler里面的 MessageQueue用的就是looper里面的mQueue.后面我们在分析looper的实现.现在消息队列有了,我们回到上面的 *sendMessageAtTime* 方法中.可以看到该方法最终调用了 *enqueueMessage* 方法.
  ```java
  private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
       msg.target = this;
       if (mAsynchronous) {
           msg.setAsynchronous(true);
       }
       return queue.enqueueMessage(msg, uptimeMillis);
   }
  ```
  非常简单,调用消息队列自己的 *enqueueMessage* 方法,这里如果handler设置为异步(mAsynchronous)的话,那发送的消息类型也是异步的.至于异步有啥作用,我们后面会提到.
  可以看到Handler的代码非常的见到,就是把消息放入消息队列中就完成任务了.那怎么从消息队列中取出消息来回调了,这个功能就交给下面要讲的looper实现了.

### Looper的实现
  上面看Handler得源码发现,其内部的消息队列指向的是Looper的内部的消息队列,那可以断定这个消息队列是由looper来维护的,同时由于每个线程都有一个唯一的线程队列,这个是怎么实现呢,可以马上想到之前我们学习过的 *ThreadLocal* 每个线程都拥有自己唯一的变量对象.这里就是 *ThreadLocal* 的典型实现场景了.

  可以看到Looper的构造函数是私有的,外面只能通过调用 *myLooper* 方法来获取到当前线程中的Looper对象.
  ```java
  public static @Nullable Looper myLooper() {
      return sThreadLocal.get();
  }
  ```
  ```java
   static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();
  ```
  刚才说了Handler在发送之前必须得能通过 *myLooper* 方法获取到looper对象.但是这里的sThreadLocal并没有实现 *initialValue* 方法,所以必须得外部调用一个方法来给ThreadLocal赋值,这里使用的是 *prepare* 方法.
  ```java
  public static void prepare() {
        prepare(true);
    }

    private static void prepare(boolean quitAllowed) {
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper(quitAllowed));
    }
  ```
  通过 *prepare* 方法,就可以是当前线程中的looper对象不为空了,同时handler就可以来发送消息了.so,可以看到在创建Handler之前的必备步骤必须先得调用一下Looper.prepare()方法,这样才能正常运行.
  这个时候有人可能会说,为啥在activity的使用过程中,我们并没有显式的调用 *Looper.prepare()* 方法呢,其实是activity在创建的时候已经在底层帮我们调用过了,这样朱祥成中的looper对象就不为空,我们就可以随便创建新的Handler而不用担心looper为空了.

  looper为主线程的looper提供了直接调用方法:
  ```java
  public static void prepareMainLooper() {
       prepare(false);
       synchronized (Looper.class) {
           if (sMainLooper != null) {
               throw new IllegalStateException("The main Looper has already been prepared.");
           }
           sMainLooper = myLooper();
       }
   }
  ```

  这个方法就是用来给UI主线程来生成唯一的looper的.我们平时经常通 *Looper.myLooper() == Looper.getMainLooper()* 来判断当前是不是在UI主线程中就是这个道理.
  looper的初始化完成后,怎么才能让消息队列的消息出列呢.这里是通过调用 *loop* 方法来完成的
  ```java
  public static void loop() {
       final Looper me = myLooper();
       if (me == null) {
           throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
       }
       final MessageQueue queue = me.mQueue;

       // Make sure the identity of this thread is that of the local process,
       // and keep track of what that identity token actually is.
       Binder.clearCallingIdentity();
       final long ident = Binder.clearCallingIdentity();

       for (;;) {
           Message msg = queue.next(); // might block
           if (msg == null) {
               // No message indicates that the message queue is quitting.
               return;
           }

           // This must be in a local variable, in case a UI event sets the logger
           Printer logging = me.mLogging;
           if (logging != null) {
               logging.println(">>>>> Dispatching to " + msg.target + " " +
                       msg.callback + ": " + msg.what);
           }

           msg.target.dispatchMessage(msg);

           if (logging != null) {
               logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
           }

           // Make sure that during the course of dispatching the
           // identity of the thread wasn't corrupted.
           final long newIdent = Binder.clearCallingIdentity();
           if (ident != newIdent) {
               Log.wtf(TAG, "Thread identity changed from 0x"
                       + Long.toHexString(ident) + " to 0x"
                       + Long.toHexString(newIdent) + " while dispatching to "
                       + msg.target.getClass().getName() + " "
                       + msg.callback + " what=" + msg.what);
           }

           msg.recycleUnchecked();
       }
   }
  ```
  这里去掉无用信息之后,实际上就是调用了消息队列的 *next* 方法,如果取到新的消息对象,就交给消息对象的target也就是handler来处理.同时也看到在注释写到消息队列的next方法可能会阻塞.回到了handler的 *dispatchMessage* 方法中了

  ```java
  /**
   * Handle system messages here.
   */
  public void dispatchMessage(Message msg) {
      if (msg.callback != null) {
          handleCallback(msg);
      } else {
          if (mCallback != null) {
              if (mCallback.handleMessage(msg)) {
                  return;
              }
          }
          handleMessage(msg);
      }
  }
  private static void handleCallback(Message message) {
         message.callback.run();
     }
  /**
  * Subclasses must implement this to receive messages.
  */
   public void handleMessage(Message msg) {
   }

  ```
  这里可以看到,如果是post的时候,调用callBack,如果是sendMessage的话直接调用handleMessage,这个方法是个空方法,是交给我们自己根据需求去实现的.

  这里其实一直有一个疑问,就是明明在线程中调用Handler的post方法,传入一个线程对象,为啥就说这玩意是在主线程中运行的呢?明明代码都在线程里面的,要运行也应该是在子线程中运行,为啥大家都说是在主线程中运行?
  今天才明白了,一直忽略了一个细节,就是上面的 *handleCallback* 的内部实现,细细看它里面内部调用的居然是callback的 *run* 方法,而不是咱们一直使用的 *start* 方法,二者唯一的区别就是start是新开一个线程来执行run方法中的代码,而run方法是直接在当前线程中执行run方法中的代码,这是二者的不同之处,so,这里其实是在主线程中直接执行了run方法中的代码,那个子线程根本就没有运行,相当于一个普通的对象.

  到这里looper的功能就讲完了,可以看到looper就是不断的从消息队列中取消息的作用.
  下面我们进入消息队列的具体实现,来重点看一下两个方法 *enqueueMessage* , *next*:

### MessageQueue的实现
  先来看 *enqueueMessage* 的实现:
  ```java

    boolean enqueueMessage(Message msg, long when) {
        if (msg.target == null) {
            throw new IllegalArgumentException("Message must have a target.");
        }
        if (msg.isInUse()) {
            throw new IllegalStateException(msg + " This message is already in use.");
        }

        synchronized (this) {
            if (mQuitting) {
                IllegalStateException e = new IllegalStateException(
                        msg.target + " sending message to a Handler on a dead thread");
                Log.w(TAG, e.getMessage(), e);
                msg.recycle();
                return false;
            }

            msg.markInUse();
            msg.when = when;
            Message p = mMessages;
            boolean needWake;
            if (p == null || when == 0 || when < p.when) {
                // New head, wake up the event queue if blocked.
                msg.next = p;
                mMessages = msg;
                needWake = mBlocked;
            } else {
                // Inserted within the middle of the queue.  Usually we don't have to wake
                // up the event queue unless there is a barrier at the head of the queue
                // and the message is the earliest asynchronous message in the queue.
                needWake = mBlocked && p.target == null && msg.isAsynchronous();
                Message prev;
                for (;;) {
                    prev = p;
                    p = p.next;
                    if (p == null || when < p.when) {
                        break;
                    }
                    if (needWake && p.isAsynchronous()) {
                        needWake = false;
                    }
                }
                msg.next = p; // invariant: p == prev.next
                prev.next = msg;
            }

            // We can assume mPtr != 0 because mQuitting is false.
            if (needWake) {
                nativeWake(mPtr);
            }
        }
        return true;
    }
  ```
  可以很明显的看出消息队列实际上是一个单向链表的实现.这里的全局变量 *mMessages* 代表着初始的头结点,如果刚开始列表为空的话就把当前的入队的message作为头结点,如果不为空的话就遍历整个列表直到找到触发时间(when)小于当前的message的消息,把message插入他的后面.

  这里的needWake变量是用来处理如果是异步任务的时候来h唤醒队列用的,平时我们用不到,这里不深入讲了

  下面再来看下 *next* 方法是如何阻塞线程的:
  ```java
  Message next() {
        // Return here if the message loop has already quit and been disposed.
        // This can happen if the application tries to restart a looper after quit
        // which is not supported.
        final long ptr = mPtr;
        if (ptr == 0) {
            return null;
        }

        int pendingIdleHandlerCount = -1; // -1 only during first iteration
        int nextPollTimeoutMillis = 0;
        for (;;) {
            if (nextPollTimeoutMillis != 0) {
                Binder.flushPendingCommands();
            }

            nativePollOnce(ptr, nextPollTimeoutMillis);

            synchronized (this) {
                // Try to retrieve the next message.  Return if found.
                final long now = SystemClock.uptimeMillis();
                Message prevMsg = null;
                Message msg = mMessages;
                if (msg != null && msg.target == null) {
                    // Stalled by a barrier.  Find the next asynchronous message in the queue.
                    do {
                        prevMsg = msg;
                        msg = msg.next;
                    } while (msg != null && !msg.isAsynchronous());
                }
                if (msg != null) {
                    if (now < msg.when) {
                        // Next message is not ready.  Set a timeout to wake up when it is ready.
                        nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                    } else {
                        // Got a message.
                        mBlocked = false;
                        if (prevMsg != null) {
                            prevMsg.next = msg.next;
                        } else {
                            mMessages = msg.next;
                        }
                        msg.next = null;
                        if (DEBUG) Log.v(TAG, "Returning message: " + msg);
                        msg.markInUse();
                        return msg;
                    }
                } else {
                    // No more messages.
                    nextPollTimeoutMillis = -1;
                }

                // Process the quit message now that all pending messages have been handled.
                if (mQuitting) {
                    dispose();
                    return null;
                }

                // If first time idle, then get the number of idlers to run.
                // Idle handles only run if the queue is empty or if the first message
                // in the queue (possibly a barrier) is due to be handled in the future.
                if (pendingIdleHandlerCount < 0
                        && (mMessages == null || now < mMessages.when)) {
                    pendingIdleHandlerCount = mIdleHandlers.size();
                }
                if (pendingIdleHandlerCount <= 0) {
                    // No idle handlers to run.  Loop and wait some more.
                    mBlocked = true;
                    continue;
                }

                if (mPendingIdleHandlers == null) {
                    mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
                }
                mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
            }

            // Run the idle handlers.
            // We only ever reach this code block during the first iteration.
            for (int i = 0; i < pendingIdleHandlerCount; i++) {
                final IdleHandler idler = mPendingIdleHandlers[i];
                mPendingIdleHandlers[i] = null; // release the reference to the handler

                boolean keep = false;
                try {
                    keep = idler.queueIdle();
                } catch (Throwable t) {
                    Log.wtf(TAG, "IdleHandler threw exception", t);
                }

                if (!keep) {
                    synchronized (this) {
                        mIdleHandlers.remove(idler);
                    }
                }
            }

            // Reset the idle handler count to 0 so we do not run them again.
            pendingIdleHandlerCount = 0;

            // While calling an idle handler, a new message could have been delivered
            // so go back and look again for a pending message without waiting.
            nextPollTimeoutMillis = 0;
        }
    }
  ```
  可以看到这里已经来就弄出一个死循环来,就是这么的霸气,通过这个循环来不断从链表中获取头结点的message对象,for循环开始有一个 *nativePollOnce*
  ```java
    nativePollOnce(ptr, nextPollTimeoutMillis);
  ```
  这个方法是一个native方法,作用就是在指定的时间内唤醒ptr,也就是消息队列的内存地址.

  那这个参数 *nextPollTimeoutMillis* 就很是关键点了.如果头结点不为空的话,判断一下当前事件是不是小于message的when,如果小于说明还没有到消息出列的时候,把 *nextPollTimeoutMillis* 调整为二者的差值,等待下次的唤醒.如果时间正好,就把头结点返回,链表前移.
  有点像你定了6点闹钟上班,5点半醒了发现没到点就继续睡半小时的,等到了6点闹钟响了你就起床上班了.差不多就是这个意思.

  核心逻辑走完之后地下出现了一个 *IdleHandler* 数组,并调用了 *queueIdle* 这个东西的作用就是用来在消息队列空闲的时候执行一些额外的工作.比如GC说明的,你可以根据需要调用消息队列的 *addIdleHandler*
  ```java
  public void addIdleHandler(@NonNull IdleHandler handler) {
       if (handler == null) {
           throw new NullPointerException("Can't add a null IdleHandler");
       }
       synchronized (this) {
           mIdleHandlers.add(handler);
       }
   }

   public void removeIdleHandler(@NonNull IdleHandler handler) {
          synchronized (this) {
              mIdleHandlers.remove(handler);
          }
  }
  ```

  最后的最后我们可以看到在 *next* 方法中判断如果message的target为空的时候的处理.那什么情况下message的target为空的,是在消息队列中的 *postSyncBarrier*
  ```java
  public int postSyncBarrier() {
        return postSyncBarrier(SystemClock.uptimeMillis());
    }

    private int postSyncBarrier(long when) {
        // Enqueue a new sync barrier token.
        // We don't need to wake the queue because the purpose of a barrier is to stall it.
        synchronized (this) {
            final int token = mNextBarrierToken++;
            final Message msg = Message.obtain();
            msg.markInUse();
            msg.when = when;
            msg.arg1 = token;

            Message prev = null;
            Message p = mMessages;
            if (when != 0) {
                while (p != null && p.when <= when) {
                    prev = p;
                    p = p.next;
                }
            }
            if (prev != null) { // invariant: p == prev.next
                msg.next = p;
                prev.next = msg;
            } else {
                msg.next = p;
                mMessages = msg;
            }
            return token;
        }
    }
  ```
  这个barrier的作用翻译过来叫栅栏,也就是用来阻隔消息继续发送的.如果你调用了 *postSyncBarrier* 那这个时间点之后的同步消息都不会执行了,除非了你把它移除掉,调用 *removeSyncBarrier* .这里不包括异步消息,异步消息还是可以继续执行的.我们平时调用的都是同步消息,异步消息应该是系统内部使用的.
  ```java
  public void removeSyncBarrier(int token) {
       // Remove a sync barrier token from the queue.
       // If the queue is no longer stalled by a barrier then wake it.
       synchronized (this) {
           Message prev = null;
           Message p = mMessages;
           while (p != null && (p.target != null || p.arg1 != token)) {
               prev = p;
               p = p.next;
           }
           if (p == null) {
               throw new IllegalStateException("The specified message queue synchronization "
                       + " barrier token has not been posted or has already been removed.");
           }
           final boolean needWake;
           if (prev != null) {
               prev.next = p.next;
               needWake = false;
           } else {
               mMessages = p.next;
               needWake = mMessages == null || mMessages.target != null;
           }
           p.recycleUnchecked();

           // If the loop is quitting then it is already awake.
           // We can assume mPtr != 0 when mQuitting is false.
           if (needWake && !mQuitting) {
               nativeWake(mPtr);
           }
       }
   }
  ```

  消息队列的大概流程就讲完了,最后我们来看下Message对象的内部实现:

### Message源码
  可以看到在Message中提供了一系列的obtain方法来用来初始赋值,这里有个简单对象池的实现,值得我们注意下:

  在上面的looper的loop方法中,当取出message之后并交给message的target执行完成后,后面有一句 *msg.recycleUnchecked();* 用来回收message对象
  ```java
  void recycleUnchecked() {
      // Mark the message as in use while it remains in the recycled object pool.
      // Clear out all other details.
      flags = FLAG_IN_USE;
      what = 0;
      arg1 = 0;
      arg2 = 0;
      obj = null;
      replyTo = null;
      sendingUid = -1;
      when = 0;
      target = null;
      callback = null;
      data = null;

      synchronized (sPoolSync) {
          if (sPoolSize < MAX_POOL_SIZE) {
              next = sPool;
              sPool = this;
              sPoolSize++;
          }
      }
  }
  ```
  可以看到这里把用完的对象清空属性之后赋给了sPool,达到了回收的目的.
  ```java
  public static Message obtain() {
       synchronized (sPoolSync) {
           if (sPool != null) {
               Message m = sPool;
               sPool = m.next;
               m.next = null;
               m.flags = 0; // clear in-use flag
               sPoolSize--;
               return m;
           }
       }
       return new Message();
   }
  ```
  可以看到在obtain的时候优先从sPool中获取是否有么有使用的Message对象,如果有的话就直接使用了,没有的时候才创建新的Message对象.所以为了节约内存,最好还是使用obtain方法.

### 总结
看了下Handler,Looper,MessageQueue的源码感觉难点是在 MessageQueue的内部 *next* 方法中,里面有一些底层native的东西,自己不是很了解,耽搁了点时间来消化.至于Handler和Looper上层实现还是比较简单的.那么,handler的讲解就到这里.

### 参考资料
[MessageQueue源码分析][5e08da4b]

  [5e08da4b]: https://hjdzone.gitbooks.io/thinkandroid/content/ThreadMessage/Chapter_1_6.html "MessageQueue源码分析"
