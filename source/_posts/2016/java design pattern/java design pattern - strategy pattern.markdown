layout: "post"
title: "java design pattern - strategy pattern"
date: "2016-06-14 21:37"
---

<!-- TOC depthFrom:2 depthTo:6 withLinks:1 updateOnSave:1 orderedList:0 -->

- [场景](#场景)
- [策略模式](#策略模式)
- [参考资料](#参考资料)

<!-- /TOC -->

## 场景
  编程中经常会遇到为达到目的有多种策略或者实现方式，这个时候可以使用策略模式来完成。
  比如商店打折有多种策略，又比如要实现一种目标可以有多种方法，这些都可以使用策略模式

## 策略模式

  策略模式是一种非常简单行为模式，旨在提供多种不同实现来满足程序要求，主要由3个角色组成：

- strategy 行为接口
- ConcreteStrategy 不同实现策略
- Context  上下文引用

![strategy pattern](/images/2016/06/Strategy_Pattern_uml.png)

<!--more-->

一个简单的demo实现：
```java
public interface Strategy {
    void doOperator();
}
```
```java
public class Strategy1 implements Strategy {

    @Override
    public void doOperator() {
        System.out.println("Strategy1 doOperator!!!");
    }
}
```
```java
public class Strategy2 implements Strategy {

    @Override
    public void doOperator() {
        System.out.println("Strategy2 doOperator!!!");
    }
}
```
```java
public class Strategy3 implements Strategy {

    @Override
    public void doOperator() {
        System.out.println("Strategy3 doOperator!!!");
    }
}
```
```java

public class Context {
    private Strategy strategy;

    public Context(Strategy strategy) {
        this.strategy = strategy;
    }

    public void operator() {
        strategy.doOperator();
    }
}
```
```java
public class Test{
    public static void main(String[] args) {
        Strategy strategy1 = new Strategy1();
        Strategy strategy2 = new Strategy2();
        Strategy strategy3 = new Strategy3();
        Context context = new Context(strategy1);
        context.operator();
        context = new Context(strategy2);
        context.operator();
        context = new Context(strategy3);
        context.operator();
    }
}

```
```java
Strategy1 doOperator!!!
Strategy2 doOperator!!!
Strategy3 doOperator!!!

```
## 参考资料
[Design Patterns - Strategy Pattern][dd571105]

  [dd571105]: http://www.tutorialspoint.com/design_pattern/strategy_pattern.htm "Design Patterns - Strategy Pattern"

[Strategy pattern][ffd6b6ab]

  [ffd6b6ab]: https://en.wikipedia.org/wiki/Strategy_pattern#Java "Strategy pattern"
