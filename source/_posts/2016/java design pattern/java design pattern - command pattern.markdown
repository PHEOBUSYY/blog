layout: "post"
title: "java design pattern - command pattern"
date: "2016-06-15 21:13"
---

<!-- TOC depthFrom:2 depthTo:6 withLinks:1 updateOnSave:1 orderedList:0 -->

- [场景](#场景)
- [命令模式](#命令模式)
- [参考资料](#参考资料)

<!-- /TOC -->

## 场景
  在编程中，经常会遇到对一系列操作的封装，比如筛选的时候各种条件的处理，可以把每一种查询条件封装成统一的
  命令对象，最后装入一个集合中，完成筛选操作，这个显著的场景就可以使用命令模式来完成。各个操作彼此独立，而且
  遵循统一的接口，便于遍历和统一处理。

## 命令模式

  命令模式是一种结构模式，旨在对一系列操作做封装，统一调用。
  命令模式由4个角色组成：

- command 命令接口
- concreteCommand 具体的命令，内部通过行为方法调用了receiver
- receiver 具体的业务逻辑对象
- invoker command的调用者

![command pattern](/images/2016/06/command_pattern_uml.gif)

<!--more-->

下面是一个简单的demo实现：

```java
public interface Command {
    void doAction();
}

```
```java
public class ConcreteCommand implements Command {
    private Receiver receiver;

    public ConcreteCommand(Receiver receiver) {
        this.receiver = receiver;
    }

    @Override
    public void doAction() {
        receiver.action();
    }
}
```
```java
public class Receiver {
    public void action(){
        System.out.println("receiver action!!!");
    }
}

```
```java
public class Invoker {
    private Command command;

    public Invoker(Command command) {
        this.command = command;
    }

    public void executeCommand() {
        command.doAction();
    }
}
```
```java
public class Test {
    public static void main(String[] args) {
        Receiver receiver = new Receiver();
        Command command = new ConcreteCommand(receiver);
        Invoker invoker = new Invoker(command);
        invoker.executeCommand();
    }
}

```
## 参考资料
[Design Patterns - Command Pattern][3a8596cb]

  [3a8596cb]: http://www.tutorialspoint.com/design_pattern/command_pattern.htm "Design Patterns - Command Pattern"

[Command pattern  ][9a0334d7]

  [9a0334d7]: https://en.wikipedia.org/wiki/Command_pattern "Command pattern"
