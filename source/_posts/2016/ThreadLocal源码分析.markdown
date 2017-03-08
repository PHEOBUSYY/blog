layout: "post"
title: "ThreadLocal源码分析"
date: "2016-12-16 09:44"
---
## ThreadLocal源码分析

  根据命名可以理解为"线程独有的"变量,也就是说这些变量是与所属线程相对应的,每个线程都有该变量独有的一份拷贝,相互之间是透明的.ThreadLocal实例的典型运用应该是private static的成员变量,比如去获取userId或者Transaction ID.

  ThreadLocal是一种线程安全的区别于锁机制的实现方式,通过线程隔离来完成线程数据之间不互相干扰.

  ThreadLocal使用非常简单,只有4个常用方法,分别是:

  *get()* :用来获取当前线程中的变量值
  *set(T value)* :用来给当前线程中的变量赋值
  *remove* :用来移除当前线程中的变量值
  *initialValue()* :用来在第一次get没有值得时候做初始化操作,一般是通过复写该方法来完成初始化的

  下面是一个demo代码,用来给每个调用它的线程产生一个唯一ID:
  ```java
  public class ThreadId {
     // Atomic integer containing the next thread ID to be assigned
     private static final AtomicInteger nextId = new AtomicInteger(0);

     // Thread local variable containing each thread's ID
     private static final ThreadLocal<Integer> threadId =
         new ThreadLocal<Integer>() {
             @Override protected Integer initialValue() {
                 return nextId.getAndIncrement();
         }
     };

     // Returns the current thread's unique ID, assigning it if necessary
     public static int get() {
         return threadId.get();
     }
 }
  ```

### 源码分析
  既然不同的线程所包含的变量值不一样,那么内部肯定是有一种类似map的数据结构在做一一对应关系.
  构造函数就是一个普通构造函数,我们从 *set()* 方法开始:

  ```java
  public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        return setInitialValue();
    }

  ```

  首先是获取当前的线程对象,然后调用 *getMap* 方法返回一个 *ThreadLocalMap* 对象,再看 *getMap* 方法:

  ```java
  ThreadLocalMap getMap(Thread t) {
       return t.threadLocals;
   }
  ```

  说明 *ThreadLocalMap* 来自于每个线程对象中.不过定义是在ThreadLocal中定义的,从明明商可以看到这个对象就是我们前面猜测的一种具有map性质的数据结构.
  其中key就是this也就是当前的ThreadLocal对象本身,value就是我们给ThreadLocal的赋值对象了.如果没有找到 *ThreadLocalMap* 对象,进入 *setInitialValue* 方法
  ```java
  /**
    * Variant of set() to establish initialValue. Used instead
    * of set() in case user has overridden the set() method.
    *
    * @return the initial value
    */
   private T setInitialValue() {
       T value = initialValue();
       Thread t = Thread.currentThread();
       ThreadLocalMap map = getMap(t);
       if (map != null)
           map.set(this, value);
       else
           createMap(t, value);
       return value;
   }
  ```
  在这里就会调用前面说过的4个常用方法之一的 *initialValue* 方法,用来做初始化赋值.同样,如果map存在就直接赋值,否者进入 *createMap* 方法

  ```java

    /**
     * Create the map associated with a ThreadLocal. Overridden in
     * InheritableThreadLocal.
     *
     * @param t the current thread
     * @param firstValue value for the initial entry of the map
     */
    void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }
  ```
  非常简单,给Thread的对象的ThreadLocalMap对象赋值,同时把变量值也赋给ThreadLocalMap.
  再看 *set* 方法,也很简单清晰:
  ```java
  /**
    * Sets the current thread's copy of this thread-local variable
    * to the specified value.  Most subclasses will have no need to
    * override this method, relying solely on the {@link #initialValue}
    * method to set the values of thread-locals.
    *
    * @param value the value to be stored in the current thread's copy of
    *        this thread-local.
    */
   public void set(T value) {
       Thread t = Thread.currentThread();
       ThreadLocalMap map = getMap(t);
       if (map != null)
           map.set(this, value);
       else
           createMap(t, value);
   }
  ```
  4个方法分分钟就讲完了,是不是非常简单,剩余核心其实就是 *ThreadLocalMap* 的实现方式了,在其内部实现了map类型的数据结构,先来看map中的Entry对象:

  ```java
  /**
        * The entries in this hash map extend WeakReference, using
        * its main ref field as the key (which is always a
        * ThreadLocal object).  Note that null keys (i.e. entry.get()
        * == null) mean that the key is no longer referenced, so the
        * entry can be expunged from table.  Such entries are referred to
        * as "stale entries" in the code that follows.
        */
       static class Entry extends WeakReference<ThreadLocal<?>> {
           /** The value associated with this ThreadLocal. */
           Object value;

           Entry(ThreadLocal<?> k, Object v) {
               super(k);
               value = v;
           }
       }
  ```
  可以看到key是 通过弱引用包含的ThreadLocal对象,value就是我们给ThreadLocal的赋值.
  下面来看map中的set实现:
  ```java
  /**
         * Set the value associated with key.
         *
         * @param key the thread local object
         * @param value the value to be set
         */
        private void set(ThreadLocal<?> key, Object value) {

            // We don't use a fast path as with get() because it is at
            // least as common to use set() to create new entries as
            // it is to replace existing ones, in which case, a fast
            // path would fail more often than not.

            Entry[] tab = table;
            int len = tab.length;
            int i = key.threadLocalHashCode & (len-1);

            for (Entry e = tab[i];
                 e != null;
                 e = tab[i = nextIndex(i, len)]) {
                ThreadLocal<?> k = e.get();

                if (k == key) {
                    e.value = value;
                    return;
                }

                if (k == null) {
                    replaceStaleEntry(key, value, i);
                    return;
                }
            }

            tab[i] = new Entry(key, value);
            int sz = ++size;
            if (!cleanSomeSlots(i, sz) && sz >= threshold)
                rehash();
        }
  ```
  遍历所有的Entry数组,如果找到key值一样的直接覆盖,如果发现一个key为空的情况,把那个位置的对象更新一下,否者如果刚开始Entry数组对象为空的时候,直接赋值数组的指定位置.这个位置是通过上面的对应hash算法生成的.
  map内部的清空无用的值的算法太复杂了,就不介绍了,重点是明白ThreadLocal的原理就可以了.
