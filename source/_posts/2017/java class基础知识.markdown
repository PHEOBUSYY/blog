layout: "post"
title: "java class基础知识"
date: "2017-01-13 15:16"
---

## java Class对象基础知识
这里要说的说Class对象指的是在java源文件完成编译之后生成的.class字节码文件,然后通过JVM的classLoader来把字节码文件载入到虚拟机中生成对应的class对象.也就是平时我们通过 A.class 或者 Class.forName("className")等方式生成的Class对象.

### 类加载器ClassLoader
java默认提供了三个ClassLoader,分别是:
1. BootStrap ClassLoader 是java类加载层次中最顶层的类加载器,负责加载jdk中的核心类库.
2. Extension ClassLoader 成为扩展类加载器,负责加载java的扩展类库,默认加载JAVA_HOME/jre/lib/ext/目录下的所有jar.
3. App ClassLoader 称为系统类加载器,负责加载应用程序classPath目录下的所有jar和class文件.

除了java提供的三个默认的ClassLoader之外,用户还可以创建自定义的ClassLoader,只要继承自java.lang.ClassLoader就可以了.也包括java提供的另外两个ClassLoader(Extension和App classLoader),但是Bootstrap ClassLoader不继承自ClassLoader,因为它不是一个普通的java类,底层由C++编写,已经嵌入到JVM内核中,当JVM启动后,Bootstrap Loader也随之启动,负责加载完核心类库后,并构造Extension 和App ClassLoader.

#### 类加载器加载类原理
这里使用了一种叫 *双亲委派* 的模式来搜索类,每个classLoader对象都有一个父类classLoader对象的引用,每当要加载类的时候,会去父类去找,如果没有找到就去父类的父类去找,一直到顶层的classLoader.如果没有找到,就尝试从顶层开始加载类,如果失败就依次往回返,如果一直都没成功加载,就叫交回到当前的classLoader,直接返回一个 *classNotFound* 异常.

这里来看下jdk8的ClassLoader源码:
```java
protected Class<?> loadClass(String name, boolean resolve)
      throws ClassNotFoundException
  {
      synchronized (getClassLoadingLock(name)) {
          // First, check if the class has already been loaded
          Class<?> c = findLoadedClass(name);
          if (c == null) {
              long t0 = System.nanoTime();
              try {
                  if (parent != null) {
                      c = parent.loadClass(name, false);
                  } else {
                      c = findBootstrapClassOrNull(name);
                  }
              } catch (ClassNotFoundException e) {
                  // ClassNotFoundException thrown if class not found
                  // from the non-null parent class loader
              }

              if (c == null) {
                  // If still not found, then invoke findClass in order
                  // to find the class.
                  long t1 = System.nanoTime();
                  c = findClass(name);

                  // this is the defining class loader; record the stats
                  sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                  sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                  sun.misc.PerfCounter.getFindClasses().increment();
              }
          }
          if (resolve) {
              resolveClass(c);
          }
          return c;
      }
  }
```
代码非常的清晰,就是从下到上的去找,然后再从上到下的加载.
![java classLoader加载顺序](images/2017/01/java classloader加载流程.gif)

#### 为什么要使用双亲委派这种模式?
1. 为了解决重复加载,当父类已经加载过了,子类就没有必要再去加载了.
2. 为了安全性,比如如果我们自己定义了和系统中一样的基础类,这两个类就会引起安全隐患.如果通过双亲委派,就会优先加载系统的原生类.放在核心代码被 "污染"

#### JVM搜索的时候,如何判断两个class对象是否相同
JVM在判定两个Class是否相同时,不仅要判断两个类名是否相同,而且要判断是否由同一个类加载器加载出来的.只有两者同时满足,JVM才认为这两个Class是相同的.

### class对象加载注意点

所有的类都是在对其第一次使用的时候,动态加载到JVM中的.当程序创建第一个对象的静态成员的引用时,就会加载这个类.
这个证明构造器也是类的静态方法,即使在构造器之前并没有static关键字.因此,使用new操作符创建类的新对象也会被当做对类的静态成员的引用.

### new A().getClass()方法
Class classA = new A().getClass();
class对象提供了一些非常实用的方法,比如getSimpleName用来返回不包含包名的类名称.而getName和getCanonicalName则会返回全限定的类名称
还可以通过 newInstance()方法来直接产生类的实例对象,前提是该类有默认的构造函数.
最后还有一些是 getInterfaces(),getSupperclass()等等,这里就不展开了.

#### A.class 字面量
这里的字面量指的是通过类名加class后缀之后获得了class对象.这个对象有很多特殊的情况需要特别说明一下.

当通过使用 ".class"来创建Class对象的引用时,不会自动 *初始化* 该Class对象.
这里的 *初始化* 是什么意思呢?并不是说这个Class对象是空的,而生成的Class对象并没有调用Class内部的静态方法或者静态代码块.
为了使用类而做的准备工作实际上包含了三个步骤:
1. 加载. 这是由前面讲的类加载器执行的.该步骤讲查找字节码(通常在ClassPath所指定的路径下查找,但这并非必须),并从这些字节码中创建一个class对象.
2. 链接. 在链接阶段将验证类中的字节码,为静态域分配存储空间,并且如果必需的话,将解析这个类创建的对其他类的所有引用.
3. 初始化. 如果该类具有超类,则对其初始化,执行静态初始化器和静态代码块.

初始化被延迟到了对静态方法 (构造器隐式地是静态的) 或者非常数静态域进行首次引用才执行.
下面通过几个例子来说明:
```java
public class TestClass {
    public static void main(String[] args) throws ClassNotFoundException {
        System.out.println(TestInitable.num);
        System.out.println(TestInitable2.name);
        System.out.println(TestInitable2.num);
        System.out.println(Class.forName("com.justyan.TestInitable3"));
        System.out.println(TestInitable4.bean);
        System.out.println(TestInitable5.num);
    }
}
class TestInitable{
    final static int num = 100;
    static{
        System.out.println("TestInitable is init!!!!");
    }
}
class TestInitable2{
    static final String name = "YY";
    static int num = 20;
    static {
        System.out.println("TestInitable2 is init!!!!!");
    }
}
class TestInitable3{
    static{
        System.out.println("TestInitable3 is init!!!!");
    }
}
class TestInitable4{
    static final TestBean bean = new TestBean();
    static {
        System.out.println("TestInitable4 is init!!!!!");
    }
}
class TestInitable5 extends TestInitable{
    static{
        System.out.println("TestInitable5 is init!!!!");
    }
}
class TestBean {
    String name;
    static{
        System.out.println("TestBean is init!!!!");
    }
}
```
这个demo就是通过调用一些类的静态属性,来查看有没有打印类内部的静态代码块,表明这个类对象有没有初始化.
先看第一个:
```java
  System.out.println(TestInitable.num);
```
由于TestInitable的num属性是 static final的,属于 "编译期常量" ,那么这个值是不需要TestInitable来初始化获取的,所以这个时候 TestInitable对象并不会初始化.内部静态代码块也不会执行,也就不会打印出里面的那条初始化信息.
这里只会打印
```
100
```

```java
System.out.println(TestInitable2.name);
System.out.println(TestInitable2.num);
```
第一行和上面的情况一样, *name* 也是属于 "编译器常量",所以第一行的时候,TestInitable2是不会初始化,但是第二行的时候由于 *num* 不是final的,所以不属于 "编译期常量",那么这个时候就会先执行静态代码块,然后才打印num的值
这里会打印:
```
YY
TestInitable2 is init!!!!!
20
```

```java
    System.out.println(Class.forName("com.justyan.TestInitable3"));
```
如果通过 *Class.forName* 反射调用的话会直接初始化该类
这里会打印初始化信息:
```
TestInitable3 is init!!!!!
```

```java
  System.out.println(TestInitable4.bean);
```
这里由于用到了TestBean对象,故先去加载TestBean对象,执行TestBean的内部静态代码块,然后才是 TestInitable4的初始化
这里会答应:
```
TestBean is init!!!!
TestInitable4 is init!!!!!
com.justyan.TestBean@610455d6
```

```java
  System.out.println(TestInitable5.num);
```
这里TestInitable5继承自TestInitable,所以可以获取到里面的num属性.但是不会初始化 TestInitable5,因为那个属性不属于它.
这里只会打印
```
100
```
这里的这种情况比较特殊,需要注意一下.实际上只要调用的父类的静态属性,这里 TestInitable5都不会初始化.

一定要理解 *编译期常量* 指的是static final的基础类型,不能是对象,如果是对象就不属于 *编译期常量* ,这个时候对该变量的调用都会导致类的初始化.
如果一个static域不是final的,那么对它访问时,总会要求在它被读取之前,要先进行链接(为这个域分配空间) 和初始化 (初始化该空间),就像对前面的TestBean的调用一样.


### 参考资料

<java编程思想> 第14章 类型信息

[深入分析Java ClassLoader原理][3d0fd4e9]

  [3d0fd4e9]: http://blog.csdn.net/xyang81/article/details/7292380 "深入分析Java ClassLoader原理"
