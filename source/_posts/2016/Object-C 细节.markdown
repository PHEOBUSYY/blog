layout: "post"
title: "Object-C 细节"
date: "2016-10-17 14:24"
---
## 细节

- NSMutableDictionary的allKey和allValue方法返回的都是不可变数组
- 可以通过NSMutableArray 的arrayToArray来转换数组为可变数组
- 可以通过NSString的stringByReplacingOccurrencesOfString方法来替换字符串


## 不通过系统自动创建xib的方式来新建xib的时候遇到的问题
  当新建xib的时候，系统不会自动把controller和xib做关联，第一步就是把xib文件的class指定为对应的controller
  第二部就是把xib中的view和controller中的view关联，这样就不会报错了

  之前之所以报错，是应为xib中的view和controller的view没有绑定在一起导致的，而直接在创建controller的时候创建xib，系统会自动把两个view绑定在一起
  所以不会报错！
  http://stackoverflow.com/questions/4763519/loaded-nib-but-the-view-outlet-was-not-set-new-to-interfacebuilder
