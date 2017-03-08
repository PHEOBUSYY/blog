layout: "post"
title: "java design pattern - state pattern"
date: "2016-05-26 09:28"
---
<!-- TOC depthFrom:2 depthTo:6 withLinks:1 updateOnSave:1 orderedList:0 -->

- [场景](#场景)
- [状态模式](#状态模式)
- [实现过程](#实现过程)
- [另一种实现方式](#另一种实现方式)
- [总结](#总结)
- [参考资料](#参考资料)

<!-- /TOC -->



## 场景
  在程序中要根据对象的不同状态来执行不同的逻辑的时候可以考虑使用状态模式。下面来看没有使用状态模式的代码
  ```java
  public class NoUseStatePattern {
    public int value;

    public void handle() {
        if (value < 100) {
            System.out.println("value begin=" + value);
        } else if (value >= 100 && value <= 200) {
            System.out.println("value middle=" + value);
        } else {
            System.out.println("value end=" + value);
        }
    }

    public static void main(String[] args) {
        NoUseStatePattern noUseStatePattern = new NoUseStatePattern();
        noUseStatePattern.value = 10;
        noUseStatePattern.handle();

        noUseStatePattern.value = 120;
        noUseStatePattern.handle();

        noUseStatePattern.value = 220;
        noUseStatePattern.handle();
    }
}
  ```
  在这里要根据value的值来执行不同的逻辑，目前来看没什么大问题，但是如果后续要增加多个value的判断或者根据value的不同执行更复杂的
  逻辑的时候，上面的代码就会有点力不从心了，这个时候我们可以使用状态模式来替换上面的逻辑。

  ```java
  public class UserStatePattern {
    private int value;
    private State state;

    public State getState() {
        return state;
    }

    public void setState(State state) {
        this.state = state;
    }

    public void handle(){
        if (getValue() < 100) {
            setState(new BeginState());
        } else if (getValue() >= 100 && getValue() < 200) {
            setState(new MiddleState());
        } else {
            setState(new EndState());
        }
        state.handle(this);
    }

    public int getValue() {
        return value;
    }

    public void setValue(int value) {
        this.value = value;
    }

    public static void main(String[] args) {
        UserStatePattern userStatePattern = new UserStatePattern();
        userStatePattern.setValue(10);
        userStatePattern.handle();
        userStatePattern.handle();
        userStatePattern.handle();
    }
}
  ```
  看上去很简单多不对，所白了就是把if else判断封装了一下，方便以后扩展，同时也更加的灵活。
  <!--more-->

## 状态模式
  状态模式是一种行为模式，可以理解为在面向对象中通过切换不同的状态来调用不同的逻辑。也就是说对象的行为取决于对象内部的不同状态
  状态模式由三个角色组成，包括：抽象状态接口，具体状态实现，上下文对象。其中抽象接口是对行为的抽象定义，具体状态实现于抽象状态接口
  最后在上下文中来实现状态转化和状态的方法调用。

![状态模式](/images/2016/05/State_Design_Pattern_UML_Class_Diagram.svg.png)

## 实现过程
  首先抽象出状态统一接口,来统一行为

  ```java
  public interface IState {
    void handle(UserStatePattern userStatePattern);
  }
  ```
  然后不同状态的类去实现状态接口，这里我们定义三个状态，开始，中间，结束
  ```java
  public class BeginIState implements IState {
    @Override
    public void handle(UserStatePattern userStatePattern) {
        System.out.println("state=" + this.getClass().getSimpleName());
        userStatePattern.setValue(120);
    }
  }
  ```

  ```java
  public class MiddleIState implements IState {
    @Override
    public void handle(UserStatePattern userStatePattern) {
        System.out.println("state=" + this.getClass().getSimpleName());
        userStatePattern.setValue(220);
    }
  }
  ```
  ```java
  public class EndIState implements IState {
    @Override
    public void handle(UserStatePattern userStatePattern) {
        System.out.println("state="+this.getClass().getSimpleName());
    }
}

  ```
  可以根据对应的场景来实现对应的业务逻辑。
  最后，通过一个上下文对象来调用不同的状态调用不同的状态实现，参看上面UserStatePattern。
## 另一种实现方式
    通过上面的例子，可以看到通过上下午对象的内部handle方法来分发状态，那么我也可以在具体的状态实现来调用其他的状态，也就是说
    可以在状态内部实现状态的转换。

  ```java
    public class BeginIState implements IState {
    @Override
    public void handle(UserStatePattern userStatePattern) {
        System.out.println("state=" + this.getClass().getSimpleName());
//        userStatePattern.setValue(120);
        userStatePattern.setIState(new MiddleIState());
    }
  }

  ```

    这也是状态模式的一种实现方法，具体还得看业务需求。
## 总结
  通过上面的demo我们看到状态模式非常简单，就是为了减少if else中的逻辑判断来进行的抽象封装。对应着设计模式的开闭原则。

## 参考资料
[state_pattern wiki][4050c426]

  [4050c426]: https://en.wikipedia.org/wiki/State_pattern "state_pattern wiki"
[处理对象的多种状态及其相互转换——状态模式][656bf5b2]

  [656bf5b2]: http://blog.csdn.net/lovelion/article/details/8523062 "处理对象的多种状态及其相互转换——状态模式"
