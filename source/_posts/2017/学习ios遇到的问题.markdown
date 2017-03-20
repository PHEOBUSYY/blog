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
