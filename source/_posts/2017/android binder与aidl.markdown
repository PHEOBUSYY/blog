layout: "post"
title: "android binder与AIDL"
date: "2017-02-21 15:59"
---
## android binder与AIDL

  最近重新研究了一下Activity启动流程,里面主要提到了ActivityManagerService这个类来管理Activity的生命周期,而如何和这个类通讯成为了理解Activity启动流程的关键,而实现与ActivityManagerService通讯的基础就是今天要讲到的Binder知识.

  我们知道你通过Luncher点击app图标来启动响应的app的时候,实际上就是相当于在Luncher这个app中取启动另一个app,也就是两个app的通讯,并且这两个app分属于不同的 *线程* ,在android两个不同线程的通讯有个统一的说法叫做 *IPC(inter-process connection)* .我们可以想象一下如果你要让两个陌生人之间互相认识,是不是需要一个中间人,那么在这里的进程就好比两个陌生人,而中间人就是我们这里的IPC,binder机制是IPC的一种实现方式.

<!--more-->

## binder机制相比于其他IPC方式的优点

  在linux的原有IPC主要有下面几种:
  - System V IPC (消息队列/信号量/共享内存)
  - 管道 / FIFO
  - SOCKET
  - 读,写锁定文件

  在手机系统中,我们如果要使两个进程来通讯,要考虑的因素的主要有:
  - 性能
  - 稳定性
  - 安全

  [点我查看Linux IPC](https://www.ibm.com/developerworks/cn/linux/l-ipc/)

  可以这样理解,如果两个进程直接通讯的时候需要消耗大量的内存和电量,那手机是肯定无法支撑的.同时如果其中一个进程发生问题,对另一个进程造成了影响的话这也是不合理的.最后,如果没有一种安全机制来对非法进程进行检查的话,容易发生信息泄漏等安全问题.

  上面提到的这几种IPC方式都或多或少的无法满足移动手机上的IPC要求,比如System V IPC,管道,SOCKET都需要对数据做两次拷贝,先拷贝到内存,然后再逐次的写入或者读出,而共享内存和文件读写的话虽然不需要数据拷贝,但是需要上层的进程完成同步控制,手段比较复杂.另外就是在安全性上他们都无法有效的识别出发送方和接收方的有效身份.基于上面提到的几点,android就提出了使用Binder这种新的IPC方式来满足我们的需求.


  IPC方式|数据拷贝次数
  --|--
  共享内存|0
  Binder|1
  System V IPC/SOCKET/读,写锁定文件|2


## binder的核心原理

![android binder结构](/images/2017/02/android binder结构图.gif)

### 用户空间和内核空间

  linux中的应用都是运行在自己独立的空间中,这个空间称为用户空间,一般来说指的就是应用运行所在的进程.之所以要这样做的目的就是为了保证每个应用的独立性,应用之间互不干扰,相互隔离.不过总会有应用需要用到系统中公共的资源,或者请求调用一些系统的功能,这个时候在linux唯一的实现方式就是 *系统调用* .通过 *系统调用* 这个唯一的接口,应用在内核的监控下执行,这种情况下我们就说应用处于 *内核态* .在linux系统中主要采用了0和3这两个特权级对应用户态和内核态.
  其中内核态对应0,代表最高的执行权限,这个时候可以操作系统的高权限操作.而用户态对应3,代表最低的用户权限,这个时候只能执行应用内部的代码.

### 内核模块和驱动

  想象一下两个独立的应用程序(进程)想要相互通信该如何实现呢?还是上面提到的两个陌生人要认识对方最好的方式就是找个中间人,那么在linux我们可以想到可以把内核作为这个中间人.我们让内核帮我们找到对方,然后调用对方的方法就完成了一次通讯.那么前面提到了:在linux中本来是没有binder这种IPC方式的,如何让linux的内核支持binder呢?我们可以通过动态可加载内核模块(LKM:Loadable Kernel Moudle)这种方式来实现.模块是具有独立的功能,可以被单独编译但是不能直接运行.它在运行时被链接到内核中,作为内核中的一部分.这样,android通过添加一个内核模块来实现binder的相关功能.也就是我们上图中的 *binder驱动* 来帮助我们实现binder这种IPC方式.


### 4个角色
  根据上面的图片我们介绍一下binder中的4个重要角色:
  1. client
  2. server
  3. serverManager
  4. binder驱动

  在binder中,通过定义client和server来明确请求的发送方和接收方.而serviceManager对应是的一个管理者的角色.首先在系统启动的时候server把自己注册到serverManager中,然后每当client请求相关的服务的时候,首先是到ServiceManager中查找对应的server,这个时候并不会把server对象直接返回给client,而是会返回一个server对象的代理对象,这个代理对象并不是真正的server.当client拿到代理对象之后,就可以像真正获取到了server,调用server的方法了,而代理对象又会通过binder驱动去调用真正的server,完成真正的调用.



![  android binder调用关系](/images/2017/02/android binder调用关系图.jpg)

  通过上图可以看到4个角色之间的调用关系.

### binder通讯模型(binder中4个角色,通过打电话的例子来类比)

  我们可以通过一个打电话的例子来说明binder的通讯模型,假设A要给B打电话,那么这里A对应client,B对应server,首先A不知道B的手机号码是什么,需要先去通讯录去查B的电话号码,这里的通讯录对应ServerManager,ServerManager里面有想要服务的对象,当知道B的电话号码后开始拨号,通过基站来把信号传递给B的手机,这里的基站对应Binder驱动.可能唯一的不适合的地方就是在系统中binder驱动只有一份,serverManager也只有一份.

### binder驱动中的对象
  在binder驱动中,有三个对象定义,分别为binder_proc,binder_node,binder_ref.其中binder_proc保存的是进程中的上下文信息,每一个应用空间都有一个binder_proc信息.binder_node叫做binder实体信息,保存的是server在binder驱动中的体现,也就是对应的server的内存地址,这里可以理解成server对象的 *指针* .binder_ref对应上面讲到的Binder代理对象,client实际上获取到的就是这个代理对象.一个binder_node对应多个binder_ref,因为可以多个代理对象来代理真正的binder_node对象.其中binder_ref到binder_node转换就是binder驱动的核心实现了.这里就不展开了.

  如果对binder驱动内部实现感兴趣可以看 [这里](http://wangkuiwu.github.io/2014/09/01/Binder-Introduce/)

## aidl实例

  我们知道,android提供了aidl的方式来完成IPC,实际上aidl就是一个binder具体实现了,我们通过讲解一个demo来说明如何使用aidl,也帮助我们理解binder在android中的具体实现.

  我们最终要实现的效果是在手机会安装好两个app,在其中client的app中点击后可以获取到server的app中的数据,表明我们的aidl调用成功.


  首先,在AS中创建一个新的工程,里面包含了两个模块,一个代表server,一个代表client,在server中创建两个java 对象,一个叫book,一个叫user.

  ![android aidl目录结构](/images/2017/02/android aidl目录结构.png)


  ```java
  public class Book implements Parcelable {
    public int bookId;
    public String name;

    public Book(int bookId, String name) {
        this.bookId = bookId;
        this.name = name;
    }


    @Override
    public int describeContents() {
        return 0;
    }

    @Override
    public void writeToParcel(Parcel dest, int flags) {
        dest.writeInt(bookId);
        dest.writeString(name);
    }
    public static final Creator<Book> CREATOR = new Creator<Book>(){
        @Override
        public Book[] newArray(int size) {
            return new Book[size];
        }

        @Override
        public Book createFromParcel(Parcel source) {
            return new Book(source);
        }
    };

    private Book(Parcel parcel) {
        this.bookId = parcel.readInt();
        this.name = parcel.readString();
    }
}
  ```
  ```java
  public class User implements Parcelable {
    public int bookId;
    public String name;

    public User(int bookId, String name) {
        this.bookId = bookId;
        this.name = name;
    }


    @Override
    public int describeContents() {
        return 0;
    }

    @Override
    public void writeToParcel(Parcel dest, int flags) {
        dest.writeInt(bookId);
        dest.writeString(name);
    }
    public static final Creator<User> CREATOR = new Creator<User>(){
        @Override
        public User[] newArray(int size) {
            return new User[size];
        }

        @Override
        public User createFromParcel(Parcel source) {
            return new User(source);
        }
    };

    private User(Parcel parcel) {
        this.bookId = parcel.readInt();
        this.name = parcel.readString();
    }
}
  ```
  这两个对象可以自由发挥,只要满足都是实现的 *Parcelable* 序列化接口就可以.
  之后我们开始创建aidl文件,右键新建选择AIDL类型,AS会自动帮我们创建好aidl文件夹并新建好aidl文件.这里写创建三个aidl文件,分别book.aidl,user.aidl,IMyAidlInterface.aidl :

  ```java
  package yonyou.com.testbinderipc;
  parcelable Book;
  ```
  ```java
  package yonyou.com.testbinderipc;
  parcelable User;
  ```
  ```java
  package yonyou.com.testbinderipc;
  // Declare any non-default types here with import statements
  import yonyou.com.testbinderipc.Book;
  import yonyou.com.testbinderipc.User;
  interface IMyAidlInterface {

     void addBook(in Book book);

      List<User> getUser();
  }
  ```
  注意:千万不要忘了在IMyAidlInterface.aidl中使用对象的时候显示声明import相应的对象,因为aidl只会自动导入基础数据类型和List.

  这里在IMyAidlInterface中定义两个方法对应我们server具有的行为.注意到这里的addBook中的参数 *in* 表明是一种输入类型,还有两个分别兑现 *out* 和 *inout* .

  aidl写好之后,我们需要一个service类来用来给其他的app调用.
  新建一个android 的service,TestAidlService:
  ```java
  public class TestAidlService extends Service {

    @Nullable
    @Override
    public IBinder onBind(Intent intent) {
        return stub;
    }
    private IMyAidlInterface.Stub stub = new IMyAidlInterface.Stub() {

        @Override
        public void addBook(Book book, String a , String b) throws RemoteException {
            Log.e("yy", "addBook: book = " + book.name);
        }

        @Override
        public List<User> getUser() throws RemoteException {
            return Collections.singletonList(new User(1,"yy"));
        }
    };
  }

  ```
  内部代码非常简单,这里你可能会问,这个 *IMyAidlInterface.Stub* 是哪里来的呢?当我们在完成了aidl文件之后,编译工程,会发现在/build/generated/source/aidl/debug/目录下有一个新生成的文件MyAidlInterface.java.这个就是编译起帮助我们生成的真正的aidl java文件了.来看下内部代码.
  ```java
  package yonyou.com.testbinderipc;

public interface IMyAidlInterface extends android.os.IInterface {
    /**
     * Local-side IPC implementation stub class.
     */
    public static abstract class Stub extends android.os.Binder implements yonyou.com.testbinderipc.IMyAidlInterface {
        private static final java.lang.String DESCRIPTOR = "yonyou.com.testbinderipc.IMyAidlInterface";

        /**
         * Construct the stub at attach it to the interface.
         */
        public Stub() {
            this.attachInterface(this, DESCRIPTOR);
        }

        /**
         * Cast an IBinder object into an yonyou.com.testbinderipc.IMyAidlInterface interface,
         * generating a proxy if needed.
         */
        public static yonyou.com.testbinderipc.IMyAidlInterface asInterface(android.os.IBinder obj) {
            if ((obj == null)) {
                return null;
            }
            android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
            if (((iin != null) && (iin instanceof yonyou.com.testbinderipc.IMyAidlInterface))) {
                return ((yonyou.com.testbinderipc.IMyAidlInterface) iin);
            }
            return new yonyou.com.testbinderipc.IMyAidlInterface.Stub.Proxy(obj);
        }

        @Override
        public android.os.IBinder asBinder() {
            return this;
        }

        @Override
        public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException {
            switch (code) {
                case INTERFACE_TRANSACTION: {
                    reply.writeString(DESCRIPTOR);
                    return true;
                }
                case TRANSACTION_addBook: {
                    data.enforceInterface(DESCRIPTOR);
                    yonyou.com.testbinderipc.Book _arg0;
                    if ((0 != data.readInt())) {
                        _arg0 = yonyou.com.testbinderipc.Book.CREATOR.createFromParcel(data);
                    } else {
                        _arg0 = null;
                    }
                    java.lang.String _arg1;
                    _arg1 = data.readString();
                    java.lang.String _arg2;
                    _arg2 = data.readString();
                    this.addBook(_arg0, _arg1, _arg2);
                    reply.writeNoException();
                    return true;
                }
                case TRANSACTION_getUser: {
                    data.enforceInterface(DESCRIPTOR);
                    java.util.List<yonyou.com.testbinderipc.User> _result = this.getUser();
                    reply.writeNoException();
                    reply.writeTypedList(_result);
                    return true;
                }
            }
            return super.onTransact(code, data, reply, flags);
        }

        private static class Proxy implements yonyou.com.testbinderipc.IMyAidlInterface {
            private android.os.IBinder mRemote;

            Proxy(android.os.IBinder remote) {
                mRemote = remote;
            }

            @Override
            public android.os.IBinder asBinder() {
                return mRemote;
            }

            public java.lang.String getInterfaceDescriptor() {
                return DESCRIPTOR;
            }

            @Override
            public void addBook(yonyou.com.testbinderipc.Book book, java.lang.String a, java.lang.String b) throws android.os.RemoteException {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    if ((book != null)) {
                        _data.writeInt(1);
                        book.writeToParcel(_data, 0);
                    } else {
                        _data.writeInt(0);
                    }
                    _data.writeString(a);
                    _data.writeString(b);
                    mRemote.transact(Stub.TRANSACTION_addBook, _data, _reply, 0);
                    _reply.readException();
                } finally {
                    _reply.recycle();
                    _data.recycle();
                }
            }

            @Override
            public java.util.List<yonyou.com.testbinderipc.User> getUser() throws android.os.RemoteException {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                java.util.List<yonyou.com.testbinderipc.User> _result;
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    mRemote.transact(Stub.TRANSACTION_getUser, _data, _reply, 0);
                    _reply.readException();
                    _result = _reply.createTypedArrayList(yonyou.com.testbinderipc.User.CREATOR);
                } finally {
                    _reply.recycle();
                    _data.recycle();
                }
                return _result;
            }
        }

        static final int TRANSACTION_addBook = (android.os.IBinder.FIRST_CALL_TRANSACTION + 0);
        static final int TRANSACTION_getUser = (android.os.IBinder.FIRST_CALL_TRANSACTION + 1);
    }

    public void addBook(yonyou.com.testbinderipc.Book book, java.lang.String a, java.lang.String b) throws android.os.RemoteException;

    public java.util.List<yonyou.com.testbinderipc.User> getUser() throws android.os.RemoteException;
}

  ```
  可以看到IMyAidlInterface表明是一个接口,同时内部有两个内部类sub和proxy,这两个对象就对应了上面个讲到的binder实体对象和binder代理对象.
  当client和server属于同一个进程时候,可以直接使用stub本地对象,如果不在同一进程中就只能使用proxy对象.这段逻辑对应这里的 *asInterface* 方法.

  写完service之后,我们需要在manifest中注册:
  ```
  <service android:name=".TestAidlService" android:exported="true" android:process=":remote" android:enabled="true">
              <intent-filter>
                  <action android:name="com.yonyou.test.aidl"/>
              </intent-filter>
      </service>
  ```

  到这里,server端的代码就算完成了.下面来看client端代码.

  client中也有aidl文件夹,里面的内容和server端是一模一样的,只需要把server中的aidl文件夹拷过来就可以了.还有两个java对象放入和server中包名一样的目录中.

  最终的目录结果应该是这样的:

![android aidl目录结构2  ](/images/2017/02/android aidl目录2.png)

  最后我们在client中的MainActivity来binder server中的service.
  ```java
  public class MainActivity extends AppCompatActivity {
    private IMyAidlInterface iMyAidlInterface;
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        Toolbar toolbar = (Toolbar) findViewById(R.id.toolbar);
        setSupportActionBar(toolbar);

        FloatingActionButton fab = (FloatingActionButton) findViewById(R.id.fab);
        fab.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View view) {
                Intent intent = new Intent("com.yonyou.test.aidl");
                intent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
                intent.setPackage("yonyou.com.testbinderipc");
                bindService(intent,sc, BIND_AUTO_CREATE);
            }
        });
    }
    private ServiceConnection sc = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            iMyAidlInterface = IMyAidlInterface.Stub.asInterface(service);
            try {
                iMyAidlInterface.addBook(new Book(1001,"如何阅读一本书"));
                List<User> user = iMyAidlInterface.getUser();
                Toast.makeText(MainActivity.this, "user is "+ user.get(0).name, Toast.LENGTH_SHORT).show();
            } catch (RemoteException e) {
                e.printStackTrace();
            }
        }

        @Override
        public void onServiceDisconnected(ComponentName name) {
            iMyAidlInterface = null;
        }
    };

    @Override
    protected void onDestroy() {
        super.onDestroy();
        unbindService(sc);
    }

  }
  ```
  点击FloatingActionButton,绑定server中的service,如果toast成功,并打印出日志表明调用成功.

  这里有个点需要注意下,在android5.0以后如果要startService或者bindService的时候一定要明确指定对方的包名,否者会抛出异常.这里intent的包名我们就写server的包名就可以了.

  在onServiceConnected中我们调用了IMyAidlInterface.Stub.asInterface,这里就完成了是否是统一进程的判断,如果不同进程就返回proxy对象,如果是统一进程就直接返回stub对象.


## 总结
  要明白的是,这里的讲的代码只是java层的binder实现,真正的binder是在底层通过c++来实现的,要明白真正的binder实现还是要去深入看源码.这里只要明白了binder的实现原理和机制就可以了.


## 参考资料

[Binder学习指南  ][9dbb3a7c]

  [9dbb3a7c]: http://weishu.me/2016/01/12/binder-index-for-newer/#comments "Binder学习指南"

[Android Binder机制(一) Binder的设计和框架][1267b68e]

  [1267b68e]: http://wangkuiwu.github.io/2014/09/01/Binder-Introduce/ "Android Binder机制(一) Binder的设计和框架"
