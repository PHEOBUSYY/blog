layout: "post"
title: "java design pattern - proxy pattern"
date: "2016-06-01 16:32"
---
<!-- TOC depthFrom:2 depthTo:6 withLinks:1 updateOnSave:1 orderedList:0 -->

- [场景](#场景)
- [代理模式](#代理模式)
- [动态代理](#动态代理)
- [参考资料](#参考资料)

<!-- /TOC -->
## 场景
  在编程中，如果要实现AOP编程或者用到要对对象方法的权限控制的时候，可以使用代理模式

## 代理模式
  代理模式是一种结构化模式，可以通过调用代理类来间接的调用真实对象的方法，同时可以在调用的同时增加一些扩张。
  代理模式由3个角色构成：
- subject 抽象接口
- realSubject 真实对象
- proxy 代理类

![proxy pattern](/images/2016/06/proxy_pattern_uml.png)
<!--more-->

  通过uml可以看到，首先是真实对象和代理类都实现subject接口，其次在代理类中有真实对象的引用，当调用proxy的对应方法的时候实际上是调用了
  真实对象的方法，同时我们也可以在调用代理的时候新增一些逻辑。
  下面是一个示例demo：
```java
public interface Subject {
    void doAction();
}
```
```java
public class RealSubject implements Subject {

    @Override
    public void doAction() {
        System.out.println("RealSubject doAction");
    }

}
```
```java
public class Proxy implements Subject {
    private RealSubject realSubject;

    @Override
    public void doAction() {
        if (realSubject == null) {
            realSubject = new RealSubject();
        }
        //AOP概念的体现
        preAction();
        realSubject.doAction();
        afterAction();
    }

    public void preAction() {
        System.out.println("preAction!!! ");
    }

    public void afterAction() {
        System.out.println("afterAction!!!");
    }
}
```
```java
public class Test {
    public static void main(String[] args) {
        Proxy proxy = new Proxy();
        proxy.doAction();
    }
}
```
```java
preAction!!!
RealSubject doAction
afterAction!!!
```
## 动态代理
  上面的代码实现属于静态代理，如果我们要对所有实现类做代理的话每次都新建一个代理类会很费事，所以在java中提出了动态代理的相关api，通过反射来实现动态代理

![dynamic proxy ](/images/2016/06/dynamic_proxy_pattern_uml.jpg)

![dynamic proxy 2](/images/2016/06/dynamic_proxy_pattern_uml_2.jpg)

  首先新的代理类要实现InvocationHandler接口，然后在里面的invoke方法中执行想要的逻辑，最后通过Proxy.newProxyInstance获取想要的接口对象，然后调用接口对象方法

```java
public class DynamicProxy implements InvocationHandler {
    //被代理对象
    private Object object;

    public DynamicProxy(Object object) {
        this.object = object;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        Object result;
        System.out.println("preInvoke !!!");
        result = method.invoke(object, args);
        System.out.println("afterInvoke!!!");
        return result;
    }
}
```
```java
RealSubject realSubject = new RealSubject();
DynamicProxy dynamicProxy = new DynamicProxy(realSubject);

Subject subject = (Subject) java.lang.reflect.Proxy.newProxyInstance(realSubject.getClass().getClassLoader(), realSubject.getClass().getInterfaces(), dynamicProxy);
subject.doAction();
```
```java
preInvoke !!!
RealSubject doAction
afterInvoke!!!
```

## 参考资料

[java设计模式之代理模式(Proxy)][6a395f1b]

  [6a395f1b]: http://blog.csdn.net/liangbinny/article/details/18656791 "java设计模式之代理模式(Proxy)"

[（Dynamic Proxy）动态代理模式的Java实现][96d0c340]

  [96d0c340]: http://haolloyin.blog.51cto.com/1177454/333257/ "（Dynamic Proxy）动态代理模式的Java实现"
