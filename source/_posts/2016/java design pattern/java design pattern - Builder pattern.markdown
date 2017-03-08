layout: "post"
title: "java design pattern - Builder pattern"
date: "2016-05-27 16:07"
---

<!-- TOC depthFrom:2 depthTo:6 withLinks:1 updateOnSave:1 orderedList:0 -->

- [场景](#场景)
- [建造者模式](#建造者模式)
- [对象内部builder实现](#对象内部builder实现)
- [总结](#总结)
- [参考资料](#参考资料)

<!-- /TOC -->

## 场景
  在平时我们会遇到复杂对象的构造过程，对象内部的组成部分属于多个或者一个复杂对象的情况，同时又很在意内部组成的组装顺序
  这个时候，我们可以使用构造者模式来把对象内部组成的创建和组装来划分开，达到解耦合。同时对外部隐藏了对象内部的具体实现。


  ```java
  public class GaiFan {
    private String cai;
    private String rice;

    public String getCai() {
        return cai;
    }

    public void setCai(String cai) {
        this.cai = cai;
    }

    public String getRice() {
        return rice;
    }

    public void setRice(String rice) {
        this.rice = rice;
    }

    @Override
    public String toString() {
        return "GaiFan{" +
                "cai='" + cai + '\'' +
                ", rice='" + rice + '\'' +
                '}';
    }
}
  ```

  ```java
  public class NoUserBuilder {
    public static void main(String[] args) {
        GaiFan gaifan = new GaiFan();
        gaifan.setCai("宫保鸡丁");
        gaifan.setRice("宫保鸡丁米饭");
        System.out.println("gaifan="+gaifan);
        gaifan = new GaiFan();
        gaifan.setCai("鱼香肉丝");
        gaifan.setRice("鱼香肉丝米饭");
        System.out.println("gaifan="+gaifan);
    }
  }
  ```
  在这里我们使用盖饭来表示要构造的对象。盖饭由两个重要部分组成，菜和米饭，在没有使用建造者模式的时候直接创建盖饭对象
  同时设置对应的属性，在这里首先是暴露了盖饭内部的做法实现，同时如果菜和米饭的创建十分复杂的时候这里的逻辑会很混乱
  再如果要调整盖饭制作顺序的话，这个单独的创建对象显然无法胜任。这个时候我们可以使用建造者模式来帮组我们解决问题。

## 建造者模式

  建造者模式是一种对象创建模式，不同于工厂和抽象工厂模式，建造者模式主要着眼于对象的内部组成创建和构造顺序。建造者模式由4个主要角色构成：
  要创建的对象（Product），抽象的构造对象（Builder），具体的构造对象（concrete builder），导演类（Derector）。其中Builder抽象定义了对象
  的构造顺序，具体的构造对象内部实现具体要构造对象的各个组成的创建，导演类负责根据不同创建者来生成不同的对象。

![builder pattern](/images/2016/05/design_pattern_builder.png)
<!--more-->
  下面是一个盖饭创建的demo实现：

  抽象的盖饭建造者，定义了盖饭创建过程
  ```java
  public abstract class GaiFanBuilder {

    public GaiFan gaiFan;

    public abstract void createCai();

    public abstract void createRice();

    public GaiFan getGaiFan(){
        return gaiFan;
    }
  }
  ```
  具体的盖饭创建者继承抽象创建者
  ```java
  public class GongbaoJidingBuilder extends GaiFanBuilder {
    public GongbaoJidingBuilder() {
        gaiFan = new GaiFan();
    }

    @Override
    public void createCai() {
        gaiFan.setCai("炒菜 ---- 宫保鸡丁");
    }

    @Override
    public void createRice() {
        gaiFan.setRice("米饭  ------ 宫保鸡丁米饭");
    }
  }
  ```

  ```java
  public class YuXiangRouSiBuilder extends GaiFanBuilder {
    public YuXiangRouSiBuilder() {
        gaiFan = new GaiFan();
    }

    @Override
    public void createCai() {
        gaiFan.setCai("炒菜 ---- 鱼香肉丝");
    }

    @Override
    public void createRice() {
        gaiFan.setRice("米饭  ------ 普通米饭");
    }
  }
  ```

  导演类负责构造盖饭的顺序并提供盖饭对象

  ```java
  public class Director {
    public GaiFan order(GaiFanBuilder gaiFanBuilder) {
        //这里可以调整顺序
        gaiFanBuilder.createCai();
        gaiFanBuilder.createRice();
        return gaiFanBuilder.getGaiFan();
    }
  }
  ```

  测试类
  ```java
  public class Test {
    public static void main(String[] args) {
        Director director = new Director();
        System.out.println("gaifan=" + director.order(new YuXiangRouSiBuilder()));
        System.out.println("gaifan=" + director.order(new GongbaoJidingBuilder()));

    }
  }
  ```
  输出：
  ```java
  gaifan=GaiFan{cai='炒菜 ---- 鱼香肉丝', rice='米饭  ------ 普通米饭'}
  gaifan=GaiFan{cai='炒菜 ---- 宫保鸡丁', rice='米饭  ------ 宫保鸡丁米饭'}

  ```
## 对象内部builder实现

  在平时我们会经常用到的内部builder实现，如果对象内部属性有很多，但是有一部分是可以通过外部赋值，有一部分可以默认定义的话，我们可以把要外部赋值的属性交给内部builder来实现，同时在builder内部完成链式设置属性。这样在一定程度可以精简代码，参看《effect in java》。

  ```java
  public static class InnerBuilder{
        private String cai;
        private String rice;

        public InnerBuilder setCai(String cai) {
            this.cai = cai;
            return this;
        }


        public InnerBuilder setRice(String rice) {
            this.rice = rice;
            return this;
        }
        public GaiFan build(){
            GaiFan gaiFan = new GaiFan();
            gaiFan.setCai(cai);
            gaiFan.setRice(rice);
            return gaiFan;
        }
    }
  ```

## 总结
  可以看到，通过builder类来分离了对象组成，通过Director来完成对象组装。这个就是建者模式的核心所在。在要考虑对象各个组成部分的顺序的时候
  可以尝试使用创建者模式。

## 参考资料
[《JAVA与模式》之建造模式][7dc79798]

  [7dc79798]: http://www.cnblogs.com/java-my-life/archive/2012/04/07/2433939.html "《JAVA与模式》之建造模式"

[Design Patterns - Builder Pattern][f03f9a72]

  [f03f9a72]: http://www.tutorialspoint.com/design_pattern/builder_pattern.htm "Design Patterns - Builder Pattern"
