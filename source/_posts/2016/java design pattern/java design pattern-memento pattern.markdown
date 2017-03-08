layout: "post"
title: "java design pattern-memento pattern"
date: "2016-05-26 16:22"
---

<!-- TOC depthFrom:2 depthTo:6 withLinks:1 updateOnSave:1 orderedList:0 -->

- [场景](#场景)
- [备忘录模式](#备忘录模式)
- [实现过程](#实现过程)
- [总结](#总结)
- [参考资料](#参考资料)

<!-- /TOC -->


## 场景
  在编程中经常遇到要回退某一个对象状态的场景，比如游戏中在打架前存盘，如果失败了可以回退到之前的状态，又比如执行了错误
  操作之后要回退到之前的某一个操作等等，总之就是在之前保存了对象状态，用户后悔了之后可以回滚，一种“后悔药”的感觉
  如果不用设计模式直接实现的话，是这个样子的：

  ```java
  public class NoMementoUse {
   private String state;

    public String getState() {
        return state;
    }

    public void setState(String state) {
        this.state = state;
    }

    public static void main(String[] args) {
        //用来保存状态
        String preState = "";
        NoMementoUse noMementoUse = new NoMementoUse();
        noMementoUse.setState("state first");
        preState = noMementoUse.getState();
        noMementoUse.setState("state second");
        //后悔了
        noMementoUse.setState(preState);
        System.out.println("state="+noMementoUse.getState());
    }
  }
  ```
  在上面的实现中，通过一个preState来保存状态，后悔了之后把preState回写到对象中，这里有两个问题：一个是暴露了对象中state属性，另一个是如果要实现更复杂的回退
  逻辑的话，不好扩展。在这种场景下，可以使用备忘录模式。
  <!--more-->

## 备忘录模式
  备忘录模式是一种行为模式，是一种具备保存对象之前状态能力的模式。
  备忘录模式由三个对象组成：一个是原始对象（originator），一个是备忘对象（memento），一个备忘管理对象（careTaker）。
  其中，原始对象就是要需要保存状态的对象，备忘对象是对要保存状态的抽象，备忘管理对象用来管理备忘对象供原始对象使用。

![memento pattern](/images/2016/05/memento_pattern_uml_diagram.jpg)

##实现过程
  首先是原始对象：

  ```java
  public class Originator {
    private String state;

    public Memento saveToMemento() {
        return new Memento(state);
    }

    public void restoreFromMemento(Memento memento) {
        this.state = memento.getState();
    }

    public String getState() {
        return state;
    }

    public void setState(String state) {
        this.state = state;
    }
  }
  ```
  通过restoreFromMemento和saveToMemento方法来存放状态和恢复状态。状态对应下面的Memento对象，memento对象内部
  可以扩展更复杂的属性和逻辑。
  ```java
  public class Memento {
    public Memento(String state) {
        this.state = state;
    }

    private String state;

    public String getState() {
        return state;
    }

    public void setState(String state) {
        this.state = state;
    }
  }

  ```
  然后是状态管理对象；

  ```java
  public class CareTaker {
    private List<Memento> mementoList = new ArrayList<>();

    public void add(Memento memento) {
        mementoList.add(memento);
    }

    public Memento get(int index) {
        return mementoList.get(index);
    }
  }
  ```
  通过状态管理对象来管理memento，可以实现更复杂的逻辑。

  最后，调用实现：
  ```java
  public class Test {
    public static void main(String[] args) {
        Originator originator = new Originator();
        CareTaker careTaker = new CareTaker();

        originator.setState("first state");
        originator.setState("second state");
        careTaker.add(originator.saveToMemento());
        originator.setState("three state");
        careTaker.add(originator.saveToMemento());

        originator.restoreFromMemento(careTaker.get(0));
        System.out.println("state="+originator.getState());
        originator.restoreFromMemento(careTaker.get(1));
        System.out.println("state="+originator.getState());
    }
  }
  ```
## 总结
   通过上面的实现来看，备忘录模式非常简单，说白了就是一种状态回退机制的实现，把状态的保存和调用交给一个新的
   类去处理，达到了解耦的目的。比如在创建页面，如果用户做了一些无效操作之后想回滚的话就可以使用备忘录模式。

## 参考资料
[Design Patterns - Memento Pattern][f93f7e38]

  [f93f7e38]: http://www.tutorialspoint.com/design_pattern/memento_pattern.htm "Design Patterns - Memento Pattern"

[Memento pattern][56344f90]

  [56344f90]: https://en.wikipedia.org/wiki/Memento_pattern "Memento pattern"
