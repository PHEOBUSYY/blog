---
layout: "post"
title: "android的flux架构分析"
date: "2016-02-19 16:00"
tags: [android, blog, github]
---


## 引言

flux是facebook在14年提出的前端框架，旨在从架构层面来解决MVC的在复杂场景下越来越复杂内部逻辑繁重等问题。
我们知道在MVC中，通过Controler来控制Modle,view,比如用户点击了view之后，view通知controler，controler来控制model做状态变换，最后再更新view。如图：

![](/images/2016/02/flux_mvc.png)

当逻辑比较复杂或者层次比较复杂的时候就会出现下面这种情况

![](/images/2016/02/flux_mvc_complex.png)

flux架构就是在这种情况下提出的。
<!--more-->

## 基本组成


![](/images/2016/02/flux-arch.png)

flux主要由四部分组成，分别为:
> dispatcher 事件调度中心，flux的中心枢纽，通过它来通知store来接收不同的action
> store 有点类是mvc中的model，封装了应用逻辑和数据的交互
> view  界面UI，根据回调事件从store中获取数据
> Action 和 ActionsCreator disptcher发送的都是Action，一个普通的pojo对象，在view中通过ActionsCreator调用dispatcher来发送不同的action

在flux中，所有的数据都是单向流动的，这样的好处是结构清晰，方便调试，找到对应bug

![](/images/2016/02/flux-simple-f8-diagram-1300w.png)

![](/images/2016/02/flux-simple-f8-diagram-explained-1300w.png)

## android flux

在android中，flux的大体实现如下:

在Activity或者Fragment中，点击一个按钮，通过ActionsCreator发送action到对应的store，store中做数据的处理和转化，处理完成后通过回调通知ui来从store中获取数据状态，最后更新UI。整个过程数据流向都是单向的。

## android flux sample
在本例中，dispatcher内部维护一个store队列，当有ActionsCreator调用dispatcher的时候，挨个遍历队列，通知里面store来接受action，在每个store中接受到action之后，处理数据，之后通过otto回调通知ui更新。

在activity的onCreate中初始化所有的对象
```java
private void initDependencies() {
        dispatcher = Dispatcher.get();
        actionsCreator = ActionsCreator.get(dispatcher);
        store = new MessageStore();
        dispatcher.register(store);
    }


```
来看Diapatcher内部实现,维护一个store队列，并且是单例的，一个典型的观察者模式
```java
public class Dispatcher {
    private static Dispatcher instance;
    private final List<Store> stores = new ArrayList<>();

    public static Dispatcher get() {
        if (instance == null) {
            instance = new Dispatcher();
        }
        return instance;
    }

    Dispatcher() {}

    public void register(final Store store) {
        if (!stores.contains(store)) {
            stores.add(store);
        }
    }

    public void unregister(final Store store) {
        stores.remove(store);
    }

    public void dispatch(Action action) {
        post(action);
    }

    private void post(final Action action) {
        for (Store store : stores) {
            store.onAction(action);
        }
    }
}

```
再看ActionsCreator,很简单的调用关系，注意这里的sendMessage方法只是一个简单写法，可以根据需求来扩张不同的实现
```java
public class ActionsCreator {

    private static ActionsCreator instance;
    final Dispatcher dispatcher;

    ActionsCreator(Dispatcher dispatcher) {
        this.dispatcher = dispatcher;
    }

    public static ActionsCreator get(Dispatcher dispatcher) {
        if (instance == null) {
            instance = new ActionsCreator(dispatcher);
        }
        return instance;
    }

    public void sendMessage(String message) {
        dispatcher.dispatch(new MessageAction(MessageAction.ACTION_NEW_MESSAGE, message));
    }
}
```
然后是store,接收到aciton之后内部通知绑定的UI更新
```java
public abstract class Store {
    private  static final Bus bus = new Bus();

    protected Store() {
    }

    public void register(final Object view) {
        this.bus.register(view);
    }

    public void unregister(final Object view) {
        this.bus.unregister(view);
    }

    void emitStoreChange() {
        this.bus.post(changeEvent());
    }

    public abstract StoreChangeEvent changeEvent();
    public abstract void onAction(Action action);

    public class StoreChangeEvent {}

```
这里通过otto来通知Activity中的UI更新

```java
  @Subscribe
  public void onStoreChange(Store.StoreChangeEvent event) {
      render(store);
  }
  private void render(MessageStore store) {
       vMessageView.setText(store.getMessage());
   }
```

## 参考资料

> [AndroidFlux][555fcc3b]

  [555fcc3b]: http://androidflux.github.io/docs/overview.html#content "AndroidFlux一览"

> [Flux架构入门教程][631840de]  

  [631840de]: http://www.ruanyifeng.com/blog/2016/01/flux.html "Flux架构入门教程"

> [Android App框架Flux][f6a7bd80]  

  [f6a7bd80]: http://www.jianshu.com/p/918719151e72 "Android App框架Flux"

> [demo地址][2356cf2b]

  [2356cf2b]: https://github.com/androidflux/flux "demo地址"
