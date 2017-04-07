---
layout: "post"
title: "Coding(IOS)项目解析"
date: "2017-04-05 08:56"
---
### 需要用到的知识
  1. ReactiveCocoa的使用
  2. KVO的使用原理
  3. 如何给一个类动态添加属性和方法
  4. RDVTabController的实现原理
  5. RAC和RACObserve语法糖
  6. weaky和strongy语法糖
  7. IOS的runtime相关知识

#### KVO介绍
IOS观察者模式的实现。  观察某一个类的某个属性，当该属性发生变化的时候回调通知。
必须满足两个条件：
1. 必须是通过调用setter方法来修改值才能获取到监听，比如通过调用self.value++这种方式是可以监听到，而如果调用_value++是无法监听到的
2. 在self中实现监听方法

```
 -(void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary<NSKeyValueChangeKey,id> *)change context:(void *)context
```
3. 记得在最后页面消失或者dealloc中解除监听

### xcode中如何创建category
首先是cmd+n调出创建对话框，然后选择 *Objective-c File* 在里面选择 *FileType* 为 *category* ，顶上的那一栏输入名称就可以了。
注意这里的名称直接写catetory的名称就可以了，不要加上原始类的名称。xcode会自动添加两者的名称为最终名称的。

### 如何给category添加属性property
首先，官方是不支持给category直接添加属性的，不过可以runtime的关联方法来动态的添加属性，具体实现如下：
1. 在category的interface部分声明property。
```Objective-c
@property(strong,nonamatic) NSString *proValue;
```
2. 在category的实现implementation部分，实现get和set方法
```Objective-c
-(void)setProValue:(NSString *)proValue
{
    objc_setAssociatedObject(self, @selector(proValue), proValue, OBJC_ASSOCIATION_ASSIGN);
}
-(NSString *)proValue
{
    return objc_getAssociatedObject(self, @selector(proValue));
}
```
通过关联来达到一个暂存的属性的目的，这里把方法 *proValue* 的地址作为key来保存，省去了再额外声明一个属性名称的过程。
3. 和正常的属性使用一样，直接调用就可以啦。
```Objective-c
self.proValue = @"test";
NSLog(@"the proValue is %@",self.proValue);
```
### 
