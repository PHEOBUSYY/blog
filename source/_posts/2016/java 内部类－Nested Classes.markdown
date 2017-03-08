layout: "post"
title: "java 内部类－Nested Classes"
date: "2016-03-29 20:20"
---

## 定义

在java中允许在一个内内部定义另一个类，这个被定义的类就叫做内部类（很形象嘛）.比如下面示例：

```java
 class OuterClass {
   ...
   static class StaticNestedClass{
     ...
   }
   ...
   class NestedClass{
     ...
   }
 }
```
其中，StaticNestedClass叫做静态内部类，也叫嵌套类，NestedClass叫做成员内部类。其中成员内部类又包括匿名内部类(Anonymous innerClass)和局部内部类(local innerCalss)内部类是外部类中一个成员;

成员内部类可以访问其外部类的成员，包括其中的作用于为private的成员,其内部有一个对外部类的引用，所依可以获取到外部类所有的成员。
静态内部类和其外部类是平级关系，与一个普通的外部类作用类似，没有对其外部类的引用。

## 为什么使用内部类

* 一种可以把逻辑按组合整合在一起的方法
* 增强封装
* 增加代码的可读性和可维护性
* 一种变相支持多重继承的方式（后面会讲到）
* 闭包（回调）类似lambda
<!--more-->
## 内部类初始化
  一个成员内部类对象必须基于外部类对象而存活，不能独立存在，通过.new Instance()的方式来初始化
  比如上面成员内部类NetstedClass初始化  new OuterClass().new NestedClass()
## Shadowing(这里不知道如何翻译？阴影？)

```java
public class ShadowTest {

    public int x = 0;

    class FirstLevel {

        public int x = 1;

        void methodInFirstLevel(int x) {
            System.out.println("x = " + x);
            System.out.println("this.x = " + this.x);
            System.out.println("ShadowTest.this.x = " + ShadowTest.this.x);
        }
    }

    public static void main(String... args) {
        ShadowTest st = new ShadowTest();
        ShadowTest.FirstLevel fl = st.new FirstLevel();
        fl.methodInFirstLevel(23);
    }
}
```
output:
  > x =23 普通的初始化
  > this.x =1   对自己本身的引用方式，和普通类一样
  > ShadowTest.this.x = 0  对外部类的引用方式

不管嵌套多少层，都可以使用这种方式来获取对应的变量。

## 一个内部类实现迭代器的demo

外部类实现Iterable接口，返回一个Iterator对象，通过一个内部类实现Iterator接口的next和hasNext方法，来遍历容器中的对象

```java
public class TestIteratorClass implements Iterable<String>{
    public static final String[] arr = new String[]{"apple","orange","banana","lemon"};
    @Override
    public Iterator<String> iterator() {
        return new TestIterator();
    }

    public static void main(String[] args) {
        TestIteratorClass test = new TestIteratorClass();
        Iterator<String> iterator = test.iterator();
        while (iterator.hasNext()) {
            System.out.println("name="+iterator.next());
        }
    }
    class TestIterator implements Iterator<String> {
       private int index = 0;
        @Override
        public boolean hasNext() {
            return index < arr.length;
        }

        @Override
        public String next() {
            return arr[index ++];
        }

        @Override
        public void remove() {
            throw new UnsupportedOperationException();
        }
    }
}

```

## local Classes
local Classes是可以定义在一组或者多组配对括号组成的代码块中的内部类，一般出现在方法的内部。

```java
public class TestLocalClass2 {
    {
        //这个也属于local classes
        class Test2{
          private int x = 0;
        }
    }
    public void test(int x) {
      //local class不能定义接口 编译无法通过
      interface Itest{
           void test();
       }
        if (x > 10) {
            //LcalClass的生命周期仅限于if块内
            class LocalClass {
                private int gender;

                //可以声明一些静态常量 可以编译通过
                public static final int id = 10;
                //必须加上final的static才可以
                public static int id2 = 10;//error 无法编译通过

                public void innerTest() {

                }
                //可以继续添加内部类，层次不限
                class InnerClass {
                    private int k;
                }
                //不能有静态方法 编译无法通过
                public static void innerTest2(){

                }
            }
            LocalClass localClass = new LocalClass();
        } else {
            //这里已经获取不到LocalClass 了
            LocalClass localClass = new LocalClass();//编译无法通过
        }
    }
}
```
通过上面demo的注释可以发现：不能声明接口，可以定义静态常量，不能定义静态方法

## 匿名内部类

匿名内部类可以是代码更加的精简易读，通常在各种回调函数中使用，这个就不多做介绍了。

## lambda 表达式

通过匿名内部类的使用过程中，会发现实际上它就是一个可以执行的代码块函数，就是一段可执行代码
```java
button.setOnClickListener(new View.OnClickListener{
  void onClick(View view){
      //do something;
  }
});
```
在jdk1.8中正式引入了lambda表达式，来简化代码并且是代码的可读性进一步加强。实际上在python中lambda的概念很早就有了，函数式编程的思想后续再讨论。下面通过一个demo的演变来展示lambda的简化过程
```java
public class Person {

    public enum Sex {
        MALE, FEMALE
    }

    String name;
    LocalDate birthday;
    Sex gender;
    String emailAddress;

    public int getAge() {
        // ...
    }

    public void printPerson() {
        // ...
    }
}
```
person是一个标准的java bean，下面实现一个过滤器，通过属性过滤来得到适合的person对象,刚开始的代码如下：

```java
 public static void printPersonsOlderThan(List<Person> list, int age) {
        for (Person p : list) {
            if (p.getAge() >= age) {
                p.printPerson();
            }
        }
 }
```
不过这样的代码很脆弱，没有很好的达到隔离和封装，如果要修改筛选属性条件的话就很难受，下面是更普遍得做法：

```java
 public static void printPersonsOlderThan(List<Person> list, CheckPerson checkPerson) {
        for (Person p : list) {
            if (checkPerson.test(p)) {
                p.printPerson();
            }
        }
 }
 public interface CheckPerson{
    boolean test(Person person);
 }
```
通过定义CheckPerson接口来定义筛选行为，在代码中实现该接口来完成筛选，这样可以很好的封装和整合代码，下面是匿名内部类的方式实现：
```java
printPersonsOlderThan(list, new CheckPerson() {
        @Override
        public boolean test(Person person) {
          return p.getGender() == Person.Sex.MALE
                        && p.getAge() >= 18
                        && p.getAge() <= 25;

        }
    });
```
像上面的定义的CheckPerson接口这样的只有一个抽象方法，传入一个对象返回boolean的结果在java中很普遍，所以jdk中已经提前内置了标准的接口Predicate（位于java.util.function中），可以直接使用，同时function包中还有很多的内置接口，以后可以看下使用方法，很简单。上面的定义改为：
```java
public static void printPersonsOlderThan(List<Person> list, Predicate<Person> checkPerson) {
       for (Person p : list) {
           if (checkPerson.test(p)) {
               p.printPerson();
           }
       }
}
```
来看lambda简化后的代码调用：
```java
printPersonsOlderThan(
              list,
              person -> person.getAge()>10
                        && person.gender == Person.Sex.FEMALE
                        && person.getAge()<50
              );
```
可以替换掉函数内部的printPerson方法，更改匹配之后的行为，在java中提供了Consumer<T>接口，内部提供了accept<T t>方法。代码如下：
```java
public static void processPerson(List<Person> list, Predicate<Person> predicate, Consumer<Person> consumer) {
        for (Person person : list) {
            if (predicate.test(person)) {
                consumer.accept(person);
            }
        }
    }
```
lambda之后的调用格式为：
```java
processPerson(list,
               person -> person.getAge() > 10
                       && person.gender == Person.Sex.FEMALE
                       && person.getAge() < 50,
               person -> person.printPerson()
               );
```
再次变更方法，如果需要获取联系人的name并打印出来，你需要一个抽象方法传入person对象并输出对应的name，可以使用java提供的Function的apply<T t , R r>方法,代码如下：
```java
public static void processPersonWithFunction(List<Person> list, Predicate<Person> predicate, Function<Person, String> mapper, Consumer<String> consumer) {
        for (Person p : list) {
            if (predicate.test(p)) {
                String data = mapper.apply(p);
                consumer.accept(data);
            }
        }
    }
```
lambda之后调用如下：
```java
processPersonWithFunction(list,
               person -> person.getAge() > 10
                       && person.gender == Person.Sex.FEMALE
                       && person.getAge() < 50,
               person -> person.name,
               name -> {
                  //这里是为了说明lambda里面可以包含多行语句的常用格式
                   System.out.println("name="+name);
                   System.out.println("name2="+name + "test");
               }
               );
```
经过上面的一番需求变动，我们可以写出通用范型格式的处理element方法：
```java
public static <X,Y> void processElements(Iterable<X> source, Predicate<X> predicate , Function<X,Y> mapper , Consumer<Y> consumer) {
        for (X x : source) {
            if (predicate.test(x)) {
                Y data = mapper.apply(x);
                consumer.accept(data);
            }
        }
    }
```
lambda调用如下：

```java
processElements(list,
                person -> person.getAge() > 10
                        && person.gender == Person.Sex.FEMALE
                        && person.getAge() < 50,
                person -> person.name,
                name -> {
                    System.out.println("name="+name);
                    System.out.println("name2="+name + "test");
                }
        );
```
在jdk8中新增了聚合操作的概念，可以简化上面的processElement方法，代码如下：
```java
list
    .stream()
    .filter(person -> person.getAge() > 10
                && person.gender == Person.Sex.FEMALE
                && person.getAge() < 50)
    .map(person1 -> person1.name)
    .forEach(name -> System.out.println("name="+name));
```
上面的操作和rxjava的处理模式是一样的，都是基于事件流的方式。参考 [Aggregate Operations（聚合操作）][67f2fd52]

  [67f2fd52]: https://docs.oracle.com/javase/tutorial/collections/streams/index.html "Aggregate Operations(聚合操作)"


## lambda语法
- 一个括号包含的逗号分隔的参数集合 类似 （String a，Integer b）,在lambda中可以省略数据类型，（a,b）,如果只有一个参数可以省略括号 省略为 a。
- 一个指向箭头  ->
- 一段代码块，可以是一行代码也可以是一个代码片段，如果是方法块需要包含在｛｝中，如果只有一行可以省略｛｝，如果是返回整个表达式，可以省略return关键字

一个普通示例：
```java
(Person person) -> {
                   return person.getAge() > 10
                       && person.gender == Person.Sex.FEMALE
                       && person.getAge() < 50;}
```
经过省略之后变成：
```java
person -> person.getAge() > 10
    && person.gender == Person.Sex.FEMALE
    && person.getAge() < 50
```
## Target Typing
如何确定一个lambda表达式的类型呢？java编译器通过以下的匹配来决定的：
- 变量声明
- 返回值
- Assignments（?）
- 数组初始化
- 方法或者构造函数变量
- lambda的body
- 条件判断
- 转型表达式

举例，比如Runable和Callable<V>，分别通过invoke方法来调用
```java
void invoke(Runnable r) {
    r.run();
}

<T> T invoke(Callable<T> c) {
    return c.call();
}
```
lambda调用：
```java
String s = invoke(() -> "done");
```
通过返回值确定是调用的Callable。
## method references (方法引用)
方法引用就是我们经常看到的lambda的"::"这种感觉特别诡异的lambda格式，通常的使用情况是这样的：
如果在lambda表达式中直接去调用其他的方法，那么就可以省略为“::”这种格式。比如：
```java
class PersonAgeComparator implements Comparator<Person> {
    public int compare(Person a, Person b) {
        return a.getBirthday().compareTo(b.getBirthday());
    }
}

Arrays.sort(rosterAsArray, new PersonAgeComparator());
```
封装成方法：
```java
static <T> void sort(T[] a, Comparator<? super T> c)
```
lambda调用：
```java
Arrays.sort(rosterAsArray,
    (Person a, Person b) -> {
        return a.getBirthday().compareTo(b.getBirthday());
    }
);
```
这里在lambda中可以直接调用person的comparebyage方法
```java
Arrays.sort(rosterAsArray,
    (a, b) -> Person.compareByAge(a, b)
);
```
上面这种情况就满足方法引用的情况，可以直接省略为：
```java
Arrays.sort(rosterAsArray, Person::compareByAge);
```
再比如，传入一个外部比较器：
```java
class ComparisonProvider {
    public int compareByName(Person a, Person b) {
        return a.getName().compareTo(b.getName());
    }

    public int compareByAge(Person a, Person b) {
        return a.getBirthday().compareTo(b.getBirthday());
    }
}
ComparisonProvider myComparisonProvider = new ComparisonProvider();
Arrays.sort(rosterAsArray, myComparisonProvider::compareByName);
```
通常的方法参考有下面4中情况：
kind  |example  
--|--
引用静态方法  | ContainClass::staticMethodName  
引用特定对象的一个实例方法  |containingObj:instanceMethodName  
引用任意对象的一个特定类型的实例方法 | ContainingType:methodName  
引用一个构造函数  | ClassName:new  

这里面第三个不太好理解，根据字面的理解就是只要是对象中的实例方法符合特定类型就可以使用方法引用。比如
```java
String[] stringArray = { "Barbara", "James", "Mary", "John",
               "Patricia", "Robert", "Michael", "Linda" };
       Arrays.sort(stringArray, new Comparator<String>() {
           @Override
           public int compare(String o1, String o2) {
               return o1.compareToIgnoreCase(o2);
           }
       });
```
经过lambda后，可以缩写为：
```java
Arrays.sort(stringArray, (o1, o2) -> o1.compareToIgnoreCase(o2));
```
满足方法引用的的情况3，可以简写为：
```java
Arrays.sort(stringArray, String::compareToIgnoreCase);
```
## 总结
lambda实际上就是匿名方法的一种简写，适用于匿名内部类的简写等情况。

## 参考
[Nested Classes][f485a38a]

  [f485a38a]: https://docs.oracle.com/javase/tutorial/java/javaOO/nested.html "Nested Classes"
