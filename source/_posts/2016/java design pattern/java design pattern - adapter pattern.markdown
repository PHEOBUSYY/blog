layout: "post"
title: "java design pattern - adapter pattern"
date: "2016-05-28 20:18"
---

<!-- TOC depthFrom:2 depthTo:6 withLinks:1 updateOnSave:1 orderedList:0 -->

- [场景](#场景)
- [适配器模式](#适配器模式)
- [总结](#总结)
- [参考资料](#参考资料)

<!-- /TOC -->
## 场景
  在编程中我们经常会遇到驴头不对马嘴的情况，比如以前是继承A的接口的对象，现在外部调用的时候需要该对象已B接口的形式来调用
  ，这个时候我们可以让对象同时集成A和B接口来达到目的，不过，这样又违反了开闭原则，这个时候我们可以使用适配器模式来达到目的。
## 适配器模式  
  适配器模式是一种结构模式，可以帮助我们把对象以不同的接口方式来调用。主要由3个角色组成：
  - Target 外部调用要求的接口方式
  - Adapter 中间用来协调的适配器
  - Adaptee 原始对象

  首先，我们让Adapter继承实现Target接口，其次在Adapter中构造Adaptee对象，然后在Target方法中调用Adaptee中相应的方法。过程非常简单。

![adapter pattern](/images/2016/05/adapter_pattern_uml.png)
<!--more-->
  下面是适配器模式的一个简单实现：
  ```java
  public interface Target {
    void request();
}
  ```
  ```java
  public class Adaptee {
    public void doSomething() {
        System.out.println("Adaptee doSomething!!!");
    }
  }
  ```
  ```java
  public class Adapter implements Target {
    private Adaptee adaptee;

    public Adapter(Adaptee adaptee) {
        this.adaptee = adaptee;
    }

    @Override
    public void request() {
        this.adaptee.doSomething();
    }
  }

  ```
  ```java
  public class Test {
    public static void main(String[] args) {
        Target target = new Adapter(new Adaptee());
        target.request();
    }
  }
  ```

## 总结
    适配器是一种非常简单的设计模式，一般是用来在后期调用时发现对象的接口不匹配的时候使用，相当于一种“补充”的手段
    在双方都不太容易修改的时候可以使用适配器模式。
## 参考资料

[Adapter pattern][fd2d45db]

  [fd2d45db]: https://en.wikipedia.org/wiki/Adapter_pattern "Adapter pattern"

[Design Patterns - Adapter Pattern][d28a5eb2]

  [d28a5eb2]: http://www.tutorialspoint.com/design_pattern/adapter_pattern.htm "Design Patterns - Adapter Pattern"
