layout: "post"
title: "java design pattern - mediator pattern"
date: "2016-06-13 22:09"
---
<!-- TOC depthFrom:2 depthTo:6 withLinks:1 updateOnSave:1 orderedList:0 -->

- [场景](#场景)
- [中介者模式](#中介者模式)
- [参考资料](#参考资料)

<!-- /TOC -->

## 场景
  在编程中经常会遇到两个或者多个复杂对象相互通讯的问题，如果对象之后互相引用的话会造成耦合严重，后续无法改动的问题
  这个时候可以考虑通过中介者模式来从中间完成中介的作用，来降低耦合度。

## 中介者模式
  中介者模式是一种结构话模式，旨在通过一个中介者对象来完成负责对象之间的通讯。
  主要由4个角色组成：
- mediator 中介者接口，定义了通讯的行为方式
- ConcreteMediator 具体实现中介，内部有所有负责对象的引用
- User 抽象的复杂对象，内部引用了中介对象
- ConcreteUser  具体复杂对象，内部的行为调用了中介对象的行为   

![mediator pattern](/images/2016/06/mediator_uml.gif)

<!--more-->

下面是一个简单的demo实现：

```java
public interface Mediator {
    void sendMsg(String msg, User user);
}
```

```java
public class ConcreteMediator implements Mediator {
    protected ConcreteUser1 concreteUser1;
    protected ConcreteUser2 concreteUser2;

    public void setConcreteUser1(ConcreteUser1 concreteUser1) {
        this.concreteUser1 = concreteUser1;
    }

    public void setConcreteUser2(ConcreteUser2 concreteUser2) {
        this.concreteUser2 = concreteUser2;
    }

    @Override
    public void sendMsg(String msg, User user) {
        if (user == concreteUser1) {
            concreteUser1.notify(msg);
        } else {
            concreteUser2.notify(msg);
        }
    }
}
```

```java
public abstract class User {
    protected Mediator mediator;

    public User(Mediator mediator) {
        this.mediator = mediator;
    }
    public abstract void sendMsg(String msg);

    public abstract void notify(String msg);
}
```

```java
public class ConcreteUser1 extends User {
    public ConcreteUser1(Mediator mediator) {
        super(mediator);
    }

    @Override
    public void notify(String msg) {
        System.out.println("ConcreteUser1 said ="+msg);
    }

    @Override
    public void sendMsg(String msg) {
        mediator.sendMsg(msg,this);
    }

}
```

```java
public class ConcreteUser2 extends User {
    public ConcreteUser2(Mediator mediator) {
        super(mediator);
    }

    @Override
    public void sendMsg(String msg) {
        mediator.sendMsg(msg,this);
    }

    @Override
    public void notify(String msg) {
        System.out.println("ConcreteUser2 said="+msg);
    }

}
```

```java
public class Test {
    public static void main(String[] args) {
        ConcreteMediator mediator = new ConcreteMediator();
        ConcreteUser1 concreteUser1 = new ConcreteUser1(mediator);
        ConcreteUser2 concreteUser2 = new ConcreteUser2(mediator);

        mediator.setConcreteUser1(concreteUser1);
        mediator.setConcreteUser2(concreteUser2);

        concreteUser1.sendMsg("Hello");
        concreteUser2.sendMsg("Hello too!");
    }
}
```

## 参考资料
[Mediator pattern][41937fbc]

  [41937fbc]: https://en.wikipedia.org/wiki/Mediator_pattern "Mediator pattern"
