---
layout: "post"
title: "android的clean架构分析"
date: "2016-02-19 16:00"
tags: [java, blog, github]
---

## 结构分层

![  clean-architecture1](/images/2016/02/clean_architecture1.png)

  首先是分层，分层的目的就是为了解耦合，把整个程序根据一定的方式来划分为不同的层次，每个层次相互独立。
  在clean-architecture(以下简称clean)中，把程序分为了四层，分别是：
  * Frameworks and Drivers:简称UI层，这里是所有具体实现：包括UI,工具类,框架,等等

  * Interface Adapters:适配层，顾名思义就是用来为上层UI层和下层将要说到的用例层做通讯的，并且把上下两层中对象做相互转化，比如我们底层的数据库表结构和服务器传过来的数据结构不太一致的时候
就需要我们做相互的转化来达成数据流动

  * Use Cases:简称用例层，这一层主要是处理底层的业务对象的，比如数据库的CRUD等操作，网络请求等等

  * Entities:简称实例层，主要是业务逻辑对象（纯java逻辑对象）

  实际上核心分层思想还是将UI和逻辑分离了，这里的UI层和接口适配层实际上就是干这事儿的，常用的实现是通过MVP来完成的，这里的UI层对应view层，Entities对应module层，中间的适配层和用例层对应Presenter层，核心的业务逻辑应该尽量交给纯java来实现，这样的好处是可以不依赖于环境，内部可以通过各种设计模式来重构和迭代等等。同时在分工上也可以根据不同人员的能力来完成不同的层次逻辑。
<!--more-->
## android程序分层
![  clen-architecture-android](/images/2016/02/clean_architecture_android.png)

  其次是程序的分层，目标是尽量让内层队外层一无所知，也就是所说的狄米特法则（最少知道原则），这样测试的时候可以不依赖与具体的android环境，方便单元测试。
  为了达成这一目标，在android中分层了3个层次，值得一提的是每个层次都有属于自己的数据模型以达到独立性的目的，在代码中通过一个mapping转化器来转化各层之间的数据对象。这点代价是为了避免各个层之间的数据交叉使用的。三个分层分别是：

  * Presentation Layer:表现层，在这一层中包含有UI，动画，当前仅仅使用mvp了，你也可以替换为mvvc或者mvc等，在这一层中，activity和fragment仅仅是view视图，
没有业务逻辑在里面，仅仅是用来渲染视图用的。

![    ](/images/2016/02/clean_architecture_mvp.png)

  * Domain Layer:业务层，业务逻辑发成在这个层次里面，在android中你会看到通过inteactors（User Cases）来实现的，在这一层中没有任何关于andorid的代码，都是java模块
，任何外部调用这一层都是通过接口来完成的。

![  ](/images/2016/02/clean_architecture_domain.png)

  * Data Layer:数据层，在app中所有需要的数据都在一层中，通过一个叫做UserRepository的工厂来提供数据，
内部根据不同的需要来从不同的途径中获取数据，这样外部不用关心内部的具体获取细节。一般的android程序不外乎网络，数据库，sdcard，sp，内层的读取等途径。

![    ](/images/2016/02/clean_architecture_data.png)

## android clean-architecture-sample-app分析
  最后，是一个clean架构demo的分析，结合上面的分层介绍来对照程序结构。
### 程序结构

![  ](/images/2016/02/clean-architecture-demo.png)

  首先是核心业务逻辑domain层，里面有用到java写的线程池做异步，封装的interactor对象来做各种业务操作
  其次是网络相关，里面通过retrofit+gson来处理网络请求
  然后是表现层presentation，里面是mvp的所有逻辑，包括ui，presenter等
  再后是数据存储层，storage，里面是所有的数据对象包括数据库的各种操作，里面用到了常用orm框架DBflow
  最后是一些辅助的工具类等等。

### 数据流转演示
  比如刚开始加载当天的cost数据，在MainActivity中初始化了MainPresenter对象
  ```java
  mMainPresenter = new MainPresenterImpl(
                ThreadExecutor.getInstance(),
                MainThreadImpl.getInstance(),
                this,
                new CostRepositoryImpl(this)
        );
  ```
  注意看prsenter的构造方法，里面有4个参数，分别对应：
  . 线程池
  . 主线程对象里面包装了handler
  . 回调接口
  . 底层存储接口对象
  然后是在开始加载数据，通过
  ```java
  @Override
    protected void onResume() {
        super.onResume();
        mMainPresenter.resume();
    }
  ```
  进入到了prsenter的内部getAllCosts方法
  ```java
  @Override
  public void getAllCosts() {
      // get all costs
      GetAllCostsInteractor getCostsInteractor = new GetAllCostsInteractorImpl(
              mExecutor,
              mMainThread,
              mCostRepository,
              this
      );
      getCostsInteractor.execute();
  }
  ```
  交给ConstsInteractor来执行，我们来看AbstractInteractor的execute方法
  ```java
  public void execute() {

      // mark this interactor as running
      this.mIsRunning = true;

      // start running this interactor in a background thread
      mThreadExecutor.execute(this);
  }
  ```
  在这里交给mThreadExecutor方法来执行
  ```java
  @Override
    public void execute(final AbstractInteractor interactor) {
        mThreadPoolExecutor.submit(new Runnable() {
            @Override
            public void run() {
                // run the main logic
                interactor.run();

                // mark it as finished
                interactor.onFinished();
            }
        });
    }
  ```
  在这里调用了Interactor的run方法，这里的run方法对应每个interactor的具体实现，来看这里GetAllConstsInteractor的run方法实现

  ```java
  @Override
   public void run() {
       // retrieve the costs from the database
       final List<Cost> costs = mCostRepository.getAllCosts();

       // sort them so the most recent cost items come first, and oldest comes last
       Collections.sort(costs, mCostComparator);

       // Show costs on the main thread
       mMainThread.post(new Runnable() {
           @Override
           public void run() {
               mCallback.onCostsRetrieved(costs);
           }
       });
   }
  ```
  里面完成了获取逻辑，并通过mainThread来回调接口，mainThread的具体实现如下：
  ```java
  public class MainThreadImpl implements MainThread {

    private static MainThread sMainThread;

    private Handler mHandler;

    private MainThreadImpl() {
        mHandler = new Handler(Looper.getMainLooper());
    }

    @Override
    public void post(Runnable runnable) {
        mHandler.post(runnable);
    }

    public static MainThread getInstance() {
        if (sMainThread == null) {
            sMainThread = new MainThreadImpl();
        }

        return sMainThread;
    }
  }

  ```
  内部维护了主线程的Handler，来回调主线程。

  以上就是demo的具体调用过程，结构还是很简单的，上层mvp，底层交给线程池，通过handler完成回调。
  之后作者又改进了下，用到了dagger2，RxJava等框架，这个之后再详细介绍


## 参考资料
[Architecting Android…The clean way?][48b5fe30]

  [48b5fe30]: http://fernandocejas.com/2014/09/03/architecting-android-the-clean-way/ "Architecting Android…The clean way?"

[ 一种更清晰的Android架构(译)][7422657c]

  [7422657c]: http://blog.csdn.net/bboyfeiyu/article/details/44560155 "一种更清晰的Android架构(译)"
