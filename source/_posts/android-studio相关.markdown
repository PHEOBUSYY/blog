layout: "post"
title: "android studio相关"
date: "2017-04-24 10:42"
---
  这里主要记录的是在android studio中的使用技巧以及在使用过程遇到的问题等等。
  <!--more-->
## 修改module名称之后遇到的问题
  由于一个module的在初次命名的时候不太规范，所以今天修改了一下它的名字，之后发生了一个诡异的问题，发现在**android视图**下找不到这个模块了，同时在**project视图**下发现这个module的图标和别的module的图标不一样。别的module图标都是表示lib的小图标，这个改名的module就是一个文件夹图标。

  仔细对比了一下和其他module的区别，发现是改名之后少了 **.iml** 文件。那么问题转移到如何搞出一个 **iml** 文件来。
  最原始的当然是把的module的 **iml** 文件copy过来改一下里面的参数。不过这样比较麻烦，到stackoverflow中发现有个更简单的方法，就是重新打开一下这个module就可以了。具体步骤就是 File --> open 找到这个modle的具体文件目录点击确定就可以了。android studio会依照依赖关系重新的生成一个完整的工程结构。

  
