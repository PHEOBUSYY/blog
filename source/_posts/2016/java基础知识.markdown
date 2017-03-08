layout: "post"
title: "java基础知识"
date: "2016-12-09 14:50"
---

## map的put方法的返回值
  今天在看EventBus源码的时候发现一行代码 ,是放一个map中put数据,关键是map的put方法居然有返回值,今天才是刚刚发现这个神奇的常识,后来通过写个demo验证一下返回值情况
```java

Map<String, String> map = new TreeMap<>();
        String a = map.put("a", "123");
        System.out.println("a = " + a);

        String a1 = map.put("a", "123");
        System.out.println("a1 = " + a1);

        String a2 = map.put("a", "789");
        System.out.println("a2 = " + a2);

        String b = map.put("b", "456");
        System.out.println("b = "+ b);

```  

返回值为:

```java
a = null
a1 = 123
a2 = 123
b = null
```
可见,如果是map中没有的数据,返回空,如果已经存在的数据,返回对应的value,如果是覆盖数据的话会返回之前存储的value

```java
/**
     * Implements Map.put and related methods
     *
     * @param hash hash for key
     * @param key the key
     * @param value the value to put
     * @param onlyIfAbsent if true, don't change existing value
     * @param evict if false, the table is in creation mode.
     * @return previous value, or null if none
     */
    final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
        else {
            Node<K,V> e; K k;
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {
                for (int binCount = 0; ; ++binCount) {
                    if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);
                        break;
                    }
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
        }
        ++modCount;
        if (++size > threshold)
            resize();
        afterNodeInsertion(evict);
        return null;
    }
```

## java isAssignableFrom(Class<?> code)方法的使用
下面是该方法的个人翻译:
>决定当前的class对象是否是传入对象的父类,父接口,或者相同的类对象
>特别是这个用来判断传入的class对象是否可以转换为当前class对象或者更宽的关联转换

可见,该方法正好和 *instance of* 方法相反,用来判断当前class是不是param class的父类或本身.而 *instance of* 是判断一个对象是不是另一个类的实现
还有一个方法是class里面的 *isInstance(Object obj)* ,用来判断当前的obj是不是当前class的实现, *instance of* 和 *isInstance* 功能一样

## A.class和new A().getClass的区别
两个方法都是返回class实例变量,正常情况下是一样的,除了多态的时候
>a.getClass返回的是运行时类型,如果A a = new B(),那么a.getClass返回的是B class
> A.class返回的是A class静态对象 ,一般用作反射时候使用
> A.class是在编译的时候就确定了,而a.getClass()却是在运行的时候确定的

## java中的transient关键字
transient英文的意思是"瞬态的,短暂的",反义词是"permament",永久的.
在java中,如果一个变量的前面添加了 *transient* 关键字,表明在当前对象的序列号中中,该变量将不被序列化.
关于 *transient* 的使用有下面三点:
1. 一旦变量被 *transient* 修饰,变量将不再是对象持久化的一部分,该变量内容在序列化后无法获得访问
2. *transient* 关键字只能修饰变量,而不能修饰类和方法.注意,本地变量是不能被 *transient* 关键字修饰的.变量如果是用户自定义变量,则改类需要实现Serialiazble接口.
3. 被 *transient* 关键字修饰的变量不再能被序列化,一个静态变量不管是否被 *transient* 修饰,均不能被序列化.  

## java中如何快速的跳出多重循环
一种是通过外面设置一个flag,通过判断flag来结束每一层的循环,还有一种是通过循环标签来快速完成:
```java
public static void loop(){
    OK:
    for (int i = 1 ; i < 100 ; i++ ) {
        for (int j = 1 ;j <=i ;j++ ) {
            if (i == 10) {
              //在这里会退出所有的循环
              break OK;
            }else if(i == 2){
              //只会退出当前一次的内层循环
              continue OK;
            }
        }
    }
}
```

## java中Thread 的run和start方法的区别
run是在当前线程中直接调用run方法中的代码,这个时候线程对象就是一个普通的对象,线程没有运行.
start方法新开一个线程来执行run方法中的代码.
```java
public synchronized void start() {
      checkNotStarted();

      hasBeenStarted = true;

      nativeCreate(this, stackSize, daemon);
  }
```
```java
public void run() {
    if (target != null) {
        target.run();
    }
}
```
