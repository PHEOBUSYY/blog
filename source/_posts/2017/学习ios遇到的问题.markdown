---
layout: "post"
title: "学习IOS遇到的问题"
date: "2017-03-20 21:31"
---
下面是在学习IOS过程中遇到的问题以及对应的解决方案：
<!--more-->
### 调用 *addObjectsFromArray* 后报错 : unrecognized selector sent to instance XXXXXX error

在刚开始学习的TableView的过程中，在获取第二页或者之后的数据的时候想当然的认为直接把获取到的网络数组直接添加到当前的数据源数组中就可以了。不过试了好好多次一直在报错。

开始以为是返回数据的问题，打印了下发现是就是一个正常的NSArray，然后怀疑本身的数据源为空，打印了下发现是有值的。这就诡异了按照API说明：
```c
- (void)addObjectsFromArray:(NSArray<ObjectType> *)otherArray;
```
只要是个数组就可以啊。后来发现了问题所在，来看下代码的实现逻辑：
```c
if (self.page2 > 1) {
            [self.data2 addObjectsFromArray:responseObject];
            [self.footer endRefreshing];
        }else{
             self.data2 = responseObject;
            [self.header endRefreshing];
        }
```
首先是第一次获取数据的时候，直接把 *responseObject* 这个数组赋给了 *data2*，然后第二次调用 *addObjectsFromArray* 完成添加。问题出现在了第一次赋值的时候，这里 *responseObject* 是一个不可变的NSArray，经过了第一次的赋值后，把 *data2* 也给变成了不可变的了。这个时候再调用 *addObjectsFromArray* 就会报错了。也算是自己的粗心大意吧。

下面是修改方案：
```c
if (self.page2 > 1) {
            [self.data2 addObjectsFromArray:responseObject];
            [self.footer endRefreshing];
        }else{
             self.data2 =  [[NSMutableArray alloc] initWithArray: responseObject];
            [self.header endRefreshing];
        }

```
问题解决。

### 如何强制要求view刷新
今天在实现一个自定义view的时候遇到一个场景：比如这个自定义view有3个按钮，需要根据一个属性动态的计算view中按钮的位置和显示的按钮个数。
之前在学习别人的自定义view的时候把所有的view初始化alloc和frame都放在了自定义view的 *initWithFrame* 方法中。之所以要放在这里而不放在 *init* 方法中的原因是 *init* 最终还是会调用 *initWithFrame* 方法，所以写在 *initWithFrame* 中是合适的。
但这样就无法实现上面说的动态调整view内容的目的了。后来上网查了下资料发现应该是通过view的 *layoutSubView* 方法来控制其中所有的子view的frame。应为这个方法主要在一下场景下触发：
1. 初始化（init）并不会触发这个方法。（LOL）
2. 调用 *addSubView* 的时候会触发。
3. 设置view *setFrame* 的时候会触发。前提是view的前后frame值发生了变化。
4. 滚动ScrollView的时候会触发scrollView的这个方法。
5. 旋转屏幕的时候会触发父view（当前viewController的view）中的这个方法。
6. view的大小发生变化的时候会触发view的父类的这个方法。

同时在官方文档中也说明，不要直接调用 *layoutSubView* 来更新view，应该通过 *setNeedsLayout* 来告诉系统在下一次更新UI的时候刷新view。如果需要立即刷新view的话，可以在调用完 *setNeedsLayout* 方法后调用 *layoutIfNeeded* 方法。

### sdwebimage不显示图片
今天新写了个工程，想通过sdwebimage加载网络图片，结果图片就是不出来，看了打印日志提示：
>The resource could not be loaded because the App Transport Security policy requires the use of a secure connection

细细一看发现图片的url不是https的，难怪了，把这个给忽略了。默认情况下在IOS9上是不支持http的。如果要增加支持需要在 **Info.plist** 中增加如下代码：
```
<key>NSAppTransportSecurity</key>
<dict>
    <key>NSAllowsArbitraryLoads</key>
    <true/>
</dict>
```

添加之后clean工程，图片就正常显示了。同理，在网络请求如果没有增加这个配置的话也是不行的。
