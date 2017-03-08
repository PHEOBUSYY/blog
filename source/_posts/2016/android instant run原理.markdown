layout: "post"
title: "android Instant Run原理"
date: "2016-11-28 20:10"
---
  起因：在android6.0的手机上，app第一次安装的时候启动特别的慢，会黑屏很长时间，开始是以为在application中干的事情太多了，可能还有写耗时的操作影响了启动时长，在最底层的application的onCreate方法加了一句log，然后重新安装，发现log居然很久之后才打印出来，那么排除了是application里面的操作影响，后来上网搜索了一下，说是Instant Run造成的，故特意查找了些资料，研究了其内部原理

### Instant Run 官方说明
>  使用Instant Run，您无需构建新的apk，就可以将更改*推送*至方法，将现有应用资源*推送*至正在运行的应用，所以几乎立刻就能看到代码的更改。

这里说明了instant的两个特性，一个是不用生成新的apk几乎立刻就可以看到代码的更改，另一个说是*推送*至设备。后面会详细说明。

### 传统的代码修改及编译部署流程

![andorid 部署流程](/images/2016/11/1.png)

构建整个apk->部署app->app重启->重启activity
而Instant Run会用更少的时间

### Instant Run编译与部署流程

![Instant Run部署流程](images/2016/11/2.png)
只构建修改的部分->部署修改的dev或资源->热部署，温部署，冷部署

代码更改  |  Instant Run行为
--|--
更改现有方法的代码实现  |  通过热交换支持：这是最快的交换类型，使更改能够更快的显示。您的应用保持运行，下次调用存根方法时会使用具有新实现的存根方法
更改或移除现有资源  |  通过温交换支持：这种交换速度也非常快，但Instant Run在将更改的资源推送至您的应用时必须重新启动当前的行为。您的应用保持运行，只是会出现小的提示，这是正常情况
结构行代码更改，例如：添加移除或更改 注释，实例字段，静态字段，静态方法签名，实例方法签名，更改当前类从其继承的父类，更改实现的界面列表，更改类的静态初始值设定项，对使用动态资源ID的布局元素重新排序  |  通过冷交换支持，这种交换速度有点慢，因为尽管不需要新的apk，Instant Run在推送结构性代码更改的时候必须重启整个应用
更改应用manifast，更改应用引用的资源，更改andorid的widget  |  android studio会重新部署，生成新的apk，这是因为manifast等文件是安装在设备中的，无法通过更新的方式修改

### Instant Run原理
  我们知道app的入口在application中，在manifast中的applcationID就是app的唯一标示，也是程序的入口，如果反编译通过运行Instant Run生成的apk会发现里面的构成是这样的

![  Instant Run apk构成](images/2016/11/3.PNG)

着重看里面的两个dex和instant-run.zip文件。首先看一下两个dex的源码：
![](images/2016/11/5.PNG)

可以看到里面就是简单的app的信息，包括application的class和包名
再看第二个dex

![](/images/2016/11/6.PNG)

里面是一堆class文件，不过没有一个是我们的自己的，那可以猜出来我们自己的代码放在了instant-run.zip文件中，解压之后如下图

![](/images/2016/11/7.PNG)

真正的代码在这里了。

同时反编译解码manifast之后，会发现我们的applcationId被替换成instant run的application BootstrapApplication，那么我们可以猜出instant-run代码作为一个宿主程序，将app作为资源加载起来，和插件话一个思路，那么instant run是怎么把代码运行起来的呢？

### 对原始代码的处理
  在第一次构建apk的时候，在每一个类中通过asm修改class文件注入一个$change成员变量，同时在每个方法的顶部增加如下代码
  ```java
  IncrementalChange localIncrementalChange = $change;
        if (localIncrementalChange != null) {
            localIncrementalChange.access$dispatch(
                    "onCreate.(Landroid/os/Bundle;)V", new Object[] { this,
                            ... });
            return;
    }
  ```
  每个方法的都有哦，可以理解如果发生变化，那么原始方法就会调用$change中对应的方法，来达到替换方法的目的
  后续运行的时候，dx补丁类，生成补丁dex，其中被修改类的补丁类是在类原名后面增加$override复制修改类的大部分代码，然后把原始类的$change赋值，这样就可以在调用原始类方法的时候调用补丁类中的对应方法了。

### application的启动
  首先程序的入口是BootstrapApplication，通过它来加载classLoader，而classLoader负责加载dex文件，那么如何才能把我们补丁的dex加载到程序中呢？
  Instant run是通过在程序classLoader的树状结构中插入补丁dex来解决问题，如下图

![  ](images/2016/11/classloader.png)
这里有个概念叫双亲委派模式，是一个典型的应用场景，首先是子类如果要加载dex的时候会通过父类去查找，如果父类没有找到，就会去父类的父类去查找，如果一直没有找到就会在子类中加载，并缓存之，instant run就是给pathClassLoader中插入了一个中间父类loader叫IncrementalClassLoader,这样就解决了类加载的问题。

在BootstrapApplication中先初始化各种loader然后通过反射调用源程序的applcation方法，同时修改
1.替换ActivityThread的mInitialApplication为realApplication  

2.替换mAllApplications 中所有的Application为realApplication  

3.替换ActivityThread的mPackages,mResourcePackages中的mLoaderApk中的application为realApplication。  

这样就解决了类加载的问题。热插件的原理也是通过这个来实现

### 如何修改资源

如果resource.ap_文件有改变，那么新建一个AssetManager对象newAssetManager，然后用newAssetManager对象替换所有当前Resource、Resource.Theme的mAssets成员变量。 2.如果当前的已经有Activity启动了，还需要替换所有Activity中mAssets成员变量

### 如何把更新载入程序

instant run会额外加载一个server，server在不断的监听来发现有补丁更新，通过socket传递给app，然后server来完成热更新，冷更新和温更新，最后重启

1.如果后缀为“.dex”,冷部署处理handleColdSwapPatch  
2.如果后缀为“classes.dex.3”,热部署处理handleHotSwapPatch  
3.其他情况,温部署，处理资源handleResourcePatch  
