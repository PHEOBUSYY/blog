layout: "post"
title: "java design pattern - flyweight pattern"
date: "2016-06-01 13:49"
---
<!-- TOC depthFrom:2 depthTo:6 withLinks:1 updateOnSave:1 orderedList:0 -->

- [场景](#场景)
- [享元模式](#享元模式)
- [总结](#总结)
- [参考资料](#参考资料)

<!-- /TOC -->
## 场景
  在编程中如果某些对象数目过多会对系统性能造成很大的影响，比如在java中如果只是一味的创建新的对象话，对内存会有很大的压力
  这个时候我们可以考虑通过共享数据对象的方法来缓解这一问题，在这里就用到了享元模式。

  下面是正常的没有使用享元模式的demo：
  ```java
  public class NoUserFlyWeight {
    private String state;

    public NoUserFlyWeight(String state) {
        this.state = state;
    }

    public void operator() {
        System.out.println("NoUserFlyWeight state =" + state);
    }

    public static void main(String[] args) {
        NoUserFlyWeight noUserFlyWeight = new NoUserFlyWeight("state1");
        noUserFlyWeight.operator();
        noUserFlyWeight = new NoUserFlyWeight("state2");
        noUserFlyWeight.operator();

        noUserFlyWeight = new NoUserFlyWeight("state3");
        noUserFlyWeight.operator();

        noUserFlyWeight = new NoUserFlyWeight("state1");
        noUserFlyWeight.operator();

        noUserFlyWeight = new NoUserFlyWeight("state3");
        noUserFlyWeight.operator();
    }
  }

  ```
  在这里每次使用noUserFlyWeight的时候都创建了新的对象，对系统性能会有很大的影响，同时不太满足单例的情景，因为需要使用不同状态“state”。

## 享元模式

  享元模式是一样结构化模式，旨在通过共享技术来重复利用相同或相似的对象。比如在java中String字符串是final的，就是通过享元模式来利用相同字符的字符串，在常量池中
  共享相同的字符串来达到减少内存的目的。
  享元模式里面有两个概念，外部状态和内部状态。
  对象要共享的话就涉及到如何归纳那些对象属性可以共享，就是创建出来打大家都一样，这样就可以共享。那些属性不能共享，需要把它转换成外部状态，通过方法调用来实现，也就是从细粒度上来划分对象的属性。可以共享的就内部状态，对外开放通过方法调用就是外部状态。


![flyweight pattern](/images/2016/06/flyweight_pattern_uml.png)
<!--more-->
  享元模式主要由4类角色构成。
- Flyweight接口 用来开放外部状态的接口
- ConcreteFlyweight 实现类 实现对外状态的具体类
- UnSharedFlyweight 实现类 实现非对外状态的具体类
- FlyweightFactory 工厂类 创建并管理共享对象的工厂类

  下面是一个享元模式的简单实现：
  ```java
  public interface FlyWeight {
      /**
       * 外部状态调用方法
       * @param state 外部状态
       */
      void operator(String state);
  }
  ```
  ```java
  public class ConcreteFlyWeight implements FlyWeight{
    @Override
    public void operator(String state) {
        System.out.println("FlyWeight state ="+state);
    }
  }
  ```
  在享元模式中，最重要的就是用来维护和创建对象的工厂类，它负责对象的共享管理，一般内部维护一个集合，通过条件来查找有没有想要的对象，如果有就返回已经有的
  没有就创建新的，供下次使用。

  ```java
  public class FlyWeightFactory {
    private HashMap<String, FlyWeight> map = new HashMap<>();

    public FlyWeight get(String key) {
        if (map.containsKey(key)) {
            return map.get(key);
        }
        ConcreteFlyWeight concreteFlyWeight = new ConcreteFlyWeight();
        map.put(key, concreteFlyWeight);
        return concreteFlyWeight;
    }
  }
  ```
  ```java
  public class Test {
    private static final String KEY_1 = "1";
    private static final String KEY_2 = "2";
    private static final String KEY_3 = "3";

    public static void main(String[] args) {
        FlyWeightFactory flyWeightFactory = new FlyWeightFactory();
        FlyWeight flyWeight = flyWeightFactory.get(KEY_1);
        flyWeight.operator("state1");

        flyWeight = flyWeightFactory.get(KEY_2);
        flyWeight.operator("state2");

        flyWeight = flyWeightFactory.get(KEY_3);
        flyWeight.operator("state3");

        flyWeight = flyWeightFactory.get(KEY_2);
        flyWeight.operator("state2");

        flyWeight = flyWeightFactory.get(KEY_1);
        flyWeight.operator("state1");
    }
  }
  ```
  输出：

  ```java
  FlyWeight state =state1
  FlyWeight state =state3
  FlyWeight state =state2
  FlyWeight state =state1
  ```
  通过test的调用，我们实际上使用了3个对象，其中state1和state2共享了两次。
## 总结
  享元模式是一种共享对象的设计模式，在遇到需要大量重复创建对象的时候可以考虑使用，只要对对象内部状态和外部状态做良好的划分就可以了，我的理解是
  不影响对象内部属性的可以设计为外部状态通过flyweight接口声明，从来达到我们想要的目的。享元模式有点像缓存。

## 参考资料

[Flyweight pattern][be1439c8]

  [be1439c8]: https://en.wikipedia.org/wiki/Flyweight_pattern "Flyweight pattern"

[《JAVA与模式》之享元模式][5d23bdc4]

  [5d23bdc4]: http://www.cnblogs.com/java-my-life/archive/2012/04/26/2468499.html "《JAVA与模式》之享元模式"
