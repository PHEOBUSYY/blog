layout: "post"
title: "java desigh pattern - composite pattern"
date: "2016-06-01 20:25"
---

<!-- TOC depthFrom:2 depthTo:6 withLinks:1 updateOnSave:1 orderedList:0 -->

- [场景](#场景)
- [组合模式](#组合模式)
- [参考资料](#参考资料)

<!-- /TOC -->
## 场景
  在编程中遇到树形结构的时候可以优先考虑组合模式，比如文件管理，比如组织结构等。

## 组合模式

  组合模式是一种结构模式，把一组相似或相同对象组合成树状结构来展示，组合模式使得我们可以对不同的结构对象进行相同的操作。

  组合模式由3个角色组成：
- Component接口或抽象类
- Leaf 叶节点，没有下级对象
- Composite 组合节点，内部包含下属对象的集合

![composite pattern](/images/2016/06/Comosite_pattern_uml.png)
<!--more-->


  其中leaf和composite都是继承于Component，实现其中约定好的方法，leaf不包含下级节点，composite通过内部容器包含下级节点。
  这样二者就统一了行为约束，方便调用处理。
  下面是一个简单的demo：

  ```java
  public abstract class Component {
    public String name;

    public Component(String name) {
        this.name = name;
    }

    public abstract void display(int depth);

    public abstract void add(Component component);

    public abstract void remove(Component component);

    public String repeat(int num, String string) {
        StringBuffer stringBuffer = new StringBuffer();
        for (int i = 0; i < num; i++) {
            stringBuffer.append(string);
        }
        return stringBuffer.toString();
    }
  }
  ```
  ```java
  public class Leaf extends Component {
    public Leaf(String name) {
        super(name);
    }

    @Override
    public void display(int depth) {
        System.out.println(repeat(depth,"-")+ name);
    }

    @Override
    public void add(Component component) {
        throw new UnsupportedOperationException("不支持的操作");
    }

    @Override
    public void remove(Component component) {
        throw new UnsupportedOperationException("不支持的操作");
    }
}
  ```
  ```java
  public class Composite extends Component {
    private List<Component> list = new ArrayList<>();
    public Composite(String name) {
        super(name);
    }

    @Override
    public void display(int depth) {
        for (Component component : list) {
            component.display(depth+2);
        }
    }

    @Override
    public void add(Component component) {
        list.add(component);
    }

    @Override
    public void remove(Component component) {
        list.remove(component);
    }
  }
  ```
  ```java
---Leaf A
---Leaf B
-----Leaf XA
-----Leaf XB
-------Leaf XYA
-------Leaf XYB
---Leaf C

  ```

## 参考资料
[Composite Pattern][1441af20]

  [1441af20]: https://en.wikipedia.org/wiki/Composite_pattern "Composite Pattern"

[设计模式读书笔记-----组合模式][6a93134c]

  [6a93134c]: http://www.cnblogs.com/chenssy/p/3299719.html "设计模式读书笔记-----组合模式"
