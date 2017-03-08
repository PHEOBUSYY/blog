layout: "post"
title: "java design pattern - facade pattern"
date: "2016-05-28 15:42"
---
<!-- TOC depthFrom:2 depthTo:6 withLinks:1 updateOnSave:1 orderedList:0 -->

- [场景](#场景)
- [外观模式](#外观模式)
- [总结](#总结)
- [参考资料](#参考资料)

<!-- /TOC -->

## 场景

   在编程中当我们遇到一个非常复杂或者难懂的系统的时候，感到很棘手，很多情况是由于系统内部的逻辑相互依赖，或者无法获取到源码所造成的
   在这种情况下我们可以使用外观模式或者叫做门面模式来处理。
   外观模式提供了一个简单接口供外部调用，而在接口内部来封装了外部要求调用复杂系统的不同行为逻辑，从而实现了外部对复杂系统的隔离。达到解耦合的目的。
   下面是一个模拟没有使用外观模式的demo：

   ```java
   public class NoUserFacade {
    public static void main(String[] args) {
        PartA partA = new PartA();
        PartB partB = new PartB();
        PartC partC = new PartC();
        partA.doSomething();
        partB.doSomething();
        partC.doSomething();
    }

    public static class PartA {
        public void doSomething() {
            System.out.println<!-- TOC depthFrom:2 depthTo:6 withLinks:1 updateOnSave:1 orderedList:0 -->

- [场景](#场景)
- [外观模式](#外观模式)
- [总结](#总结)
- [参考资料](#参考资料)

<!-- /TOC -->("partA doSomething");
        }
    }

    public static class PartB {
        public void doSomething() {
            System.out.println("partA doSomething");
        }
    }

    public static class PartC {
        public void doSomething() {
            System.out.println("partA doSomething");
        }
    }

  }
   ```

  在这里用partA,partB,partC来表示复杂系统的内部行为逻辑，如果我们在用的时候就象在main函数中的调用，很有能是非常的混乱，同时也对测试造成了很大的困难。下面我们使用外观模式来解决上面遇到的问题。

## 外观模式
  外观模式是一种结构型模式，它提供了一个简单的接口给外部调用，外观模式可以：
- 使一个library或者类库方便调用，易懂，方便测试
- 减少了对类库内部的逻辑依赖，使的系统更加的灵活
- 通过一个设计良好的接口来封装那些设计不规范或者使用不便的API

![facade pattern](/images/2016/05/Facade_design_pattern_in_UML.png)

<!--more-->

  下面是使用了外观模式之后的demo：

  ```java
  public class FacadeMaker {
      public void doSomething(){
          new PackageA().doSomething();
          new PackageB().doSomething();
          new PackageC().doSomething();
      }
      public static class PackageA{
          public void doSomething(){
              System.out.println("packageA doSomething");
          }
      }

      public static class PackageB{
          public void doSomething(){
              System.out.println("packageB doSomething");
          }
      }

      public static class PackageC{
          public void doSomething(){
              System.out.println("packageC doSomething");
          }
      }

      public static void main(String[] args) {
          new FacadeMaker().doSomething();
      }
  }
  ```
  可以看到通过FacadeMaker的doSomething来对系统内部的调用进行封装，外部调用的时候直接通过doSomething来和下面的系统完成交互
  这样增加了程序的易读性和可测试性。

## 总结

  外观模式给我的感觉就像是常用“组合”方式，就是在以前的调用的地方再封装一下，由于以前的系统的复杂性或者不可读性，这个时候就可以使用外观模式
  它和装饰模式的区别更多的像一种多对少的区别，装饰模式用来装饰一个目标类，然后供外部调用装饰类，外观模式 可能是多个目标类的组合调用，供外部
  调用。

## 参考资料

[Facade pattern][698a31b8]

  [698a31b8]: https://en.wikipedia.org/wiki/Facade_pattern "Facade pattern"

[Design Patterns - Facade Pattern][717e5a13]

  [717e5a13]: http://www.tutorialspoint.com/design_pattern/facade_pattern.htm "Design Patterns - Facade Pattern"
