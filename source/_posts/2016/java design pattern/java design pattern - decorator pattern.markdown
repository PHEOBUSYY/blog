layout: "post"
title: "java design pattern - decorator pattern"
date: "2016-05-28 18:08"
---
<!-- TOC depthFrom:2 depthTo:6 withLinks:1 updateOnSave:1 orderedList:0 -->

- [场景](#场景)
- [装饰模式](#装饰模式)
- [总结](#总结)
- [参考资料](#参考资料)

<!-- /TOC -->
## 场景

   在编程中，经常会遇到要对原来的某个对象进行功能上的扩展，这个时候如果直接在原来的类基础上直接修改的话违反了开闭原则
   同时可能会影响到原来的功能。这个时候我们可以通过装饰模式来达到功能上的扩展，而不影响原来的功能。

## 装饰模式

  装饰模式顾名思义就是对一个对象进行装饰（wrap）来达到功能扩展的目的。一般由4个角色来组成，分别为：  
  - Component 要扩展对象的实现接口  
  - ConcreteComponent 要扩展的具体对象  
  - Decorator 继承于Component 用来装饰的基础类  
  - ConcreteDecorator 具体的装饰类，根据功能对Decorator中的方法进行扩展  

  实现的步骤如下：
  - 新建Decorator继承或实现于Component
  - 在Decorator中添加一个Component类型的变量
  - 在Decorator的构造函数中传入一个Component对象，来完成上面变量的初始化
  - 在Decorator中实现所有Component中定义的方法，里面调用Component变量中对应的方法
  - ConcreteDecorator继承Decorator，在里面对需要调整的方法进行复写修改

![decorator pattern](/images/2016/05/decorator_pattern_uml.png)
<!--more-->
  下面是一个简单的demo实现：

  基础接口
  ```java
  public interface Component {
    void operator();
  }
  ```
  要调整的对象
  ```java
  public class ConcreteComponent implements Component {
    @Override
    public void operator() {
        System.out.println("ConcreteComponent operator");
    }
  }
  ```
  装饰基类
  ```java
  public class Decorator implements Component {
    private Component component;

    public Decorator(Component component) {
        this.component = component;
    }

    @Override
    public void operator() {
        this.component.operator();
    }
}
  ```
  具体装饰类A
  ```java
  public class ConcreteDecoratorA extends Decorator{
    public ConcreteDecoratorA(Component component) {
        super(component);
    }

    @Override
    public void operator() {
        super.operator();
        System.out.println("ConcreteDecoratorA  operator");
    }
  }
  ```
  具体装饰类B
  ```java
  public class ConcreteDecoratorB extends Decorator {
    public ConcreteDecoratorB(Component component) {
        super(component);
    }

    @Override
    public void operator() {
        super.operator();
        System.out.println("ConcreteDecoratorB operator ");
    }
  }
  ```
  测试类
  ```java
  public class Test {
    public static void main(String[] args) {
        ConcreteComponent concreteComponent = new ConcreteComponent();
        ConcreteDecoratorA concreteDecoratorA = new ConcreteDecoratorA(concreteComponent);
        ConcreteDecoratorB concreteDecoratorB = new ConcreteDecoratorB(concreteDecoratorA);
        concreteDecoratorB.operator();
    }
  }
  ```
  输出：
  ```java
  ConcreteComponent operator
  ConcreteDecoratorA  operator
  ConcreteDecoratorB operator

  ```
## 总结
  可以看到装饰模式在没有影响以前逻辑的基础上，完成了功能扩展，在java中IO相关的api大量的用到了装饰模式。
  同时我们可以发现如果没有Component或者只有一个要调整的ConcreteDecorator的时候装饰模式会退化为一个简单
  的继承方式，也就是继承一下要要调整类，在里面复写要扩展的方法。另外，装饰模式我觉得是AOP编程概念的一种具体实现。

## 参考资料

[Design Patterns - Decorator Pattern][e92abeae]

  [e92abeae]: http://www.tutorialspoint.com/design_pattern/decorator_pattern.htm "Design Patterns - Decorator Pattern"

[Decorator pattern][f5bc2495]

  [f5bc2495]: https://en.wikipedia.org/wiki/Decorator_pattern "Decorator pattern"
