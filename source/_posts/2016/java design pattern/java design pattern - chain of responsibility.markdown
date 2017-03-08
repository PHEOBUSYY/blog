layout: "post"
title: "java design pattern - chain of responsibility"
date: "2016-06-13 21:36"
---

<!-- TOC depthFrom:2 depthTo:6 withLinks:1 updateOnSave:1 orderedList:0 -->

- [场景](#场景)
- [责任链模式](#责任链模式)
- [参考资料](#参考资料)

<!-- /TOC -->

## 场景
  在编程中，经常会遇到链式处理的结构，比如一个请假流程，需要逐级审批，符合权限的管理角色来决定是否可以处理请求，如果可以
  处理就直接处理，不能处理就交由上级或下一个指定人员来处理，这种情况就可以使用责任链模式。

## 责任链模式
  责任链模式是一种结构模式，主要通过Handler抽象方法来完成一个链式操作。主要由两个角色组成：
- Handler 抽象的处理类，内部实现setSuccessor方法和抽象的handleRequest方法
- ConcreteHandler  Handler的具体实现

![chain of responsibility](/images/2016/06/chain_of_responsibility_uml.gif)

<!--more-->
下面是一个简单的demo实现：
```java
public abstract class Handler {
    protected Handler successor;

    public void setSuccessor(Handler successor) {
        this.successor = successor;
    }

    public abstract void handleRequest (int request);
}
```
```java
public class ConcreteHandler1 extends Handler {

    @Override
    public void handleRequest(int request) {
        if (request > 0 && request < 10) {
            System.out.println("ConcreteHandler1 handle the request");
        } else {
            if (successor != null) {
                successor.handleRequest(request);
            }
        }
    }
}
```
```java
public class ConcreteHandler2 extends Handler {

    @Override
    public void handleRequest(int request) {
        if (request >= 10 && request < 20) {
            System.out.println("ConcreteHandler2 handle the request");
        } else {
            if (successor != null) {
                successor.handleRequest(request);
            }
        }
    }
}
```
```java
public class ConcreteHandler3 extends Handler {

    @Override
    public void handleRequest(int request) {
        if (request >= 20 && request < 30) {
            System.out.println("ConcreteHandler3 handle the request");
        } else {
            if (successor != null) {
                successor.handleRequest(request);
            }
        }
    }
}

```
```java
public class Test {
    public static void main(String[] args) {
        Handler concreteHandler1 = new ConcreteHandler1();
        Handler concreteHandler2 = new ConcreteHandler2();
        Handler concreteHandler3 = new ConcreteHandler3();
        concreteHandler1.setSuccessor(concreteHandler2);
        concreteHandler2.setSuccessor(concreteHandler3);
        concreteHandler1.handleRequest(2);
        concreteHandler1.handleRequest(12);
        concreteHandler1.handleRequest(21);
    }
}
```
```java
ConcreteHandler1 handle the request
ConcreteHandler2 handle the request
ConcreteHandler3 handle the request
```

## 参考资料
[Chain of Responsibility][723a0580]

  [723a0580]: http://www.dofactory.com/net/chain-of-responsibility-design-pattern "Chain of Responsibility"
[Chain-of-responsibility pattern][d6efe906]

  [d6efe906]: https://en.wikipedia.org/wiki/Chain-of-responsibility_pattern "Chain-of-responsibility pattern"
