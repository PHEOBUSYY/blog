layout: "post"
title: "java design pattern - interpreter pattern"
date: "2016-06-14 21:12"
---

<!-- TOC depthFrom:2 depthTo:6 withLinks:1 updateOnSave:1 orderedList:0 -->

- [场景](#场景)
- [解释器模式](#解释器模式)
- [参考资料](#参考资料)

<!-- /TOC -->

## 场景
  解释器模式在编程中很少用到，一般是用来自己定义一套简单的语法语句通过解释器解释执行，在这里简单的实现一下完事。

## 解释器模式

  解释器模式是一种行为模式，用来通过解释器来解析简单语句并执行。
  比如可以通过解释器来模拟一个简单的四则运算。
  解释器模式主要由4个角色组成：

- abstractExpression 定义了通用的解释行为方法 inerpreter(context)
- TerminalExpression 具体值参数，在四则运算中就是数字
- NonTerminalExpression 操作符 ，对应四则运算的加减乘除
- context 上下文对象，用来保存一些常用的值等中间信息

![interpreter pattern](/images/2016/06/interpreter_pattern_uml.jpg)

<!--more-->

下面是一个四则运算的demo：

```java
public interface Expression {
    int interpreter(Context context);
}
```
```java
public class IntValue implements Expression {
    private int value ;

    public IntValue(int value) {
        this.value = value;
    }

    @Override
    public int interpreter(Context context) {
        return value;
    }
}
```
```java
public abstract class Symbol implements Expression {
    protected Expression intValue1,intValue2;

    public Symbol(Expression intValue1, Expression intValue2) {
        this.intValue1 = intValue1;
        this.intValue2 = intValue2;
    }
}

```
```java
public class AddOperation extends Symbol{
    public AddOperation(Expression intValue1, Expression intValue2) {
        super(intValue1, intValue2);
    }

    @Override
    public int interpreter(Context context) {
        return intValue1.interpreter(context) + intValue2.interpreter(context);
    }
}
```
```java
public class MultiOperation extends Symbol {

    public MultiOperation(Expression intValue1, Expression intValue2) {
        super(intValue1, intValue2);
    }

    @Override
    public int interpreter(Context context) {
        return intValue1.interpreter(context) * intValue2.interpreter(context);
    }
}
```
```java
public class DivOperation extends Symbol {

    public DivOperation(Expression intValue1, Expression intValue2) {
        super(intValue1, intValue2);
    }

    @Override
    public int interpreter(Context context) {
        return intValue1.interpreter(context) / intValue2.interpreter(context);
    }
}

```
```java
public class RemoveOperation extends Symbol {
    public RemoveOperation(Expression intValue1, Expression intValue2) {
        super(intValue1, intValue2);
    }

    @Override
    public int interpreter(Context context) {
        return intValue1.interpreter(context) - intValue2.interpreter(context);
    }
}

```
```java
public class Processor {
    public static int process(String expression,Context context){
        Stack<Expression> stack = new Stack<>();
        for (int i = 0; i < expression.split(" ").length; i++) {
            String ex = expression.split(" ")[i];
            if (ex.equals("*")) {
                String next = expression.split(" ")[++i];
                Expression pop = stack.pop();
                IntValue intValue2 = new IntValue(Integer.parseInt(next));
                stack.push(new MultiOperation(pop, intValue2));
            } else if (ex.equals("/")) {
                String next = expression.split(" ")[++i];
                Expression pop = stack.pop();
                IntValue intValue2 = new IntValue(Integer.parseInt(next));
                stack.push(new DivOperation(pop, intValue2));
            } else if (ex.equals("+")) {
                String next = expression.split(" ")[++i];
                Expression pop = stack.pop();
                IntValue intValue2 = new IntValue(Integer.parseInt(next));
                stack.push(new AddOperation(pop, intValue2));
            } else if (ex.equals("-")) {
                String next = expression.split(" ")[++i];
                Expression pop = stack.pop();
                IntValue intValue2 = new IntValue(Integer.parseInt(next));
                stack.push(new RemoveOperation(pop, intValue2));
            } else {
                stack.push(new IntValue(Integer.parseInt(ex)));
            }
        }
        return stack.pop().interpreter(context);
    }
}

```
```java
public class Test {
    public static void main(String[] args) {
        Context context = new Context();
        String string = "1 + 1 * 2";
        System.out.println("Process="+Processor.process(string,context));
    }
}
```
```java
Process=4
```

## 参考资料
[Interpreter pattern][14d15308]

  [14d15308]: https://en.wikipedia.org/wiki/Interpreter_pattern "Interpreter pattern"

[Design Patterns - Interpreter Pattern][b7ce94ad]

  [b7ce94ad]: http://www.tutorialspoint.com/design_pattern/interpreter_pattern.htm "Design Patterns - Interpreter Pattern"
