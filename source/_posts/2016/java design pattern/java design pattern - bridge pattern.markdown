layout: "post"
title: "java design pattern - bridge pattern"
date: "2016-05-30 20:10"
---
<!-- TOC depthFrom:2 depthTo:6 withLinks:1 updateOnSave:1 orderedList:0 -->

- [场景](#场景)
- [桥接模式](#桥接模式)
- [参考资料](#参考资料)

<!-- /TOC -->

## 场景

    在编程中会遇到一个对象本身的定义和它的行为动作都在变化，这种情况下调整代码是非常痛苦的，实际上是在两个维度上面做调整
    这个时候可以使用桥接模式，将两个维度变化分别抽象，最后通过组合的方式把二者组合起来来最小化的调整代码。

## 桥接模式

![bridge pattern](/images/2016/05/bridge_pattern_uml.svg.png)

  桥接模式是一种结构模式，是一种对抽象本质（abstraction）和行为实现(implementaion)的解耦合的设计模式.桥接模式由4个角色组成：
  - 抽象基类（abstraction） 用在类本身变化的抽象
  - 重定义基类（RefinedAbstraction） 类本身具体的实现
  - 行为接口（Implementor） 类行为的抽象接口
  - 具体实现类(ConcreteImplementor) 类行为的具体实现

  比如，手机中有很多软件，同时手机也有很多个品牌，这两个维度都在变化的时候，可以通过桥接模式来实现。最后把抽象和实现通过组合的方式绑定在一起，在这里
  手机的品牌为主，所以是抽象，而软件是一个功能点，所以是行为抽象。
<!--more-->
  ```java
  public abstract class MobileBrand {
    protected MobileSoft mobileSoft;

    public MobileBrand(MobileSoft mobileSoft) {
        this.mobileSoft = mobileSoft;
    }
    public abstract void use();
  }
  ```
  ```java
  public class AppleMobile extends MobileBrand {
    public AppleMobile(MobileSoft mobileSoft) {
        super(mobileSoft);
    }

    @Override
    public void use() {
        mobileSoft.play();
        System.out.println("AppleMobile use");
    }
  }

  public class SamSungMobile extends MobileBrand {
    public SamSungMobile(MobileSoft mobileSoft) {
        super(mobileSoft);
    }

    @Override
    public void use() {
        mobileSoft.play();
        System.out.println("SamSungMobile use");
    }
  }
  ```
  ```java
  public interface MobileSoft {
    void play();
  }
  ```
  ```java
  public class ConcreteMobileSoftA implements MobileSoft{
    @Override
    public void play() {
        System.out.println("ConcreteMobileSoftA play!!!");
    }
  }

  public class ConcreteMobileSoftB implements MobileSoft{
    @Override
    public void play() {
        System.out.println("ConcreteMobileSoftB play!!!");
    }
  }
  ```
  ```java
  public class Test {
    public static void main(String[] args) {
        MobileSoft concreteMobileSoftA = new ConcreteMobileSoftA();
        MobileSoft concreteMobileSoftB = new ConcreteMobileSoftB();
        MobileBrand brand = new SamSungMobile(concreteMobileSoftA);
        brand.use();
        brand = new SamSungMobile(concreteMobileSoftB);
        brand.use();

        brand = new AppleMobile(concreteMobileSoftA);
        brand.use();
        brand = new AppleMobile(concreteMobileSoftB);
        brand.use();
    }
  }
  ```

  输出：
  ```java
  ConcreteMobileSoftA play!!!
  SamSungMobile use
  ConcreteMobileSoftB play!!!
  SamSungMobile use
  ConcreteMobileSoftA play!!!
  AppleMobile use
  ConcreteMobileSoftB play!!!
  AppleMobile use

  ```

  内部的组合是关键点。

## 参考资料
[Bridge Pattern][226b3655]

  [226b3655]: https://en.wikipedia.org/wiki/Bridge_pattern "Bridge Pattern"

[Design Patterns - Bridge Pattern][71fc2053]

  [71fc2053]: http://www.tutorialspoint.com/design_pattern/bridge_pattern.htm "Design Patterns - Bridge Pattern"
