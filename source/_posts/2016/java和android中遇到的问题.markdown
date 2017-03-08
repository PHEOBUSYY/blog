layout: "post"
title: "java和android中遇到的问题"
date: "2016-11-16 14:32"
---

## java获取当前的方法名和类名
原来真的可以，以前太想当然了，居然光是想着得整体反射才能获取到，没想到在方法内部就可以，遇到问题不能凭感觉，要求证之后再下结论，永远保持怀疑的态度

```java
Thread.currentThread().getStackTrace()[1].getClassName();
Thread.currentThread().getStackTrace()[1].getMethodName();
```


## gson parse转换带有非法字符的字符串时报错
   今天遇到一个问题，服务器返回的HashKey中是通过base64生成的字符串，里面有个“=”，这个时候通过gson parse的时候就会报MalformedJsonException异常
   ，后来通过gson的toJson方法来解决，也就是用gson再转一次，这样就会把里面的“=”转化为16进制的字符串了。

```java
public static String toJson(Map<String, String> paramMap) {
        //所有的参数最终都要拼接成一个JsonObject对象，然后通过一个叫param的参数发送到服务器
        JsonObject jsonObject = new JsonObject();
        if (paramMap != null && !paramMap.isEmpty()) {
            for (String key : paramMap.keySet()) {
                String value = paramMap.get(key);
                if (TextUtils.isEmpty(key) || TextUtils.isEmpty(value)) {
                    continue;
                }
                //通过jsonParser可以解决java的\“转义的问题
                //如果文本中带空格的，请使用整个对象转json的方式
                JsonParser parser = new JsonParser();
                try {
                    JsonElement jsonElement = parser.parse(value);
                    jsonObject.add(key, jsonElement);
                } catch (JsonSyntaxException e) {
                    e.printStackTrace();
                    //如果字符串不符合格式，gson会报MalformedJsonException异常，比如字符串中包含等号，<,>等特殊符号，不能直接转换
                    //目前在简报的服务器返回HashKey中发现有等号的清空下无法解析，故在这里增加了处理
                    try {
                        value = GsonUtils.ObjectToJson(value);
                        JsonElement jsonElement = parser.parse(value);
                        jsonObject.add(key, jsonElement);
                    } catch (JsonSyntaxException e1) {
                        e1.printStackTrace();
                        //如果再次转换还是失败的话，使用JsonReader的宽容模式
                        try {
                            JsonReader jsonReader = new JsonReader(new StringReader(value));
                            jsonReader.setLenient(true);
                            JsonElement jsonElement = parser.parse(jsonReader);
                            jsonObject.add(key, jsonElement);
                        } catch (Exception e2) {
                            e2.printStackTrace();
                            NetLogger.log("HttpUtil toJson meet error param ,the param is " + value + ",please check the value is correct!!!");
                        }
                    }
                }

            }
        }
        NetLogger.log("toJson =" + jsonObject.toString());
        return jsonObject.toString();
      }

```

## 关于java中BigInteger的原理分析
  通过内部的数组来分段存放数字集合，最后再拼成一个整体返回
  [BigInteger原理][aeb50140]



  [aeb50140]: http://www.hollischuang.com/archives/176 "BigInteger原理"

## 为什么在android的api中有些方法明明是public的,但是在外部调用的时候就是编译错误,显示为不可以使用的状态?
  原因是在方法或者field的注释中有一个 *@hide* 的注解,这个注解的作用就是告诉javaDoc,凡是带了这个注解的方法或者field都会把javaDoc忽略,也就是说对外是看不到这个方法.但是在内部是可以使用的.有点像android的internal包,只能手机本身调用,外部是无法直接调用的.这样做的目的是比如有些方法还不是很稳定,就先把它标记为对外暂时不可用,等后期稳定了之后再打开.

  不过通过java的发射是可以调用这些带有 *@hide* 的方法和filed的.

[What does @hide mean in the Android source code?][54bc024f]

  [54bc024f]: http://stackoverflow.com/questions/17035271/what-does-hide-mean-in-the-android-source-code "What does @hide mean in the Android source code?"

## EventBus在子线程中post事件遇到的问题
  之前遇到一个问题,就是EventBus在post事件的时候如果是在子线程中,同时回到事件更新UI了,就会导致网络请求的数据没有了,不知道为什么.今天看了源码明白了其中的缘由.
  源码,在EventBus中,当你没有指定事件回调的ThreadMode的话,回调事件就会在post所在线程中执行了,那如果你在子线程中post事件,在主线程中的回调事件没有修改ThreadMode话回调事件就会在子线程中运行,这个时候如果更新UI,就会报异常,但是由于EventBus把异常给catch到了,所以页面没有什么异常反应.看着就好像没啥变化一样.
  然后再说为啥请求就没有数据了呢,是因为有个同事把post事件放在网络请求的回调来发送的,但是网络请求的回调默认不是在主线程中处理的,所以就会导致post的事件是在子线程中发送的,然后就会报异常就是不能在子线程中更新UI.这个时候,activity的UI是就不会响应UI更新了.

## android的子线程中真的不能更新UI吗?
  如果在Activity的onCreate方法中直接new子线程然后在里面更新UI是不会报错的.原因是检查线程是否是子线程的方法在onCreate的时候还没有调用呢.
[  Android中子线程真的不能更新UI吗？][1f5066c8]

  [1f5066c8]: http://www.cnblogs.com/xuyinhuan/p/5930287.html "Android中子线程真的不能更新UI吗？"

## 在生成android AIDL的过程中老是提示失败
  通过网上的教程尝试通过AIDL传递复杂对象,比如一个序列化对象Book,然后把Book.java,Book.aidl,TestAidlInterface.aidl 这三个文件放在同一个包中,老是无法通过编译,没有生成响应的java文件.后来鼓捣了半天,发现把 Book.java 文件放在其他的java目录中就可以了.也就是说aidl文件放在专用的aidl包中,java类放在普通的java包中.猜测可能是新版的android studio做了调整.

## 在android 5.0及以上的版本中直接通过intent调用startService,bindService,stopService等方法的时候报错 *java.lang.IllegalArgumentException: Service Intent must be explicit: Intent { act=XXXXXXXX flg=0x10000000 }*  
  这是因为在5.0的android源码中增加了对intent的package的限制.下面来看bindService的直接调用方 *ContextImpl* 中的 *bindService* 方法:
  ```java
  @Override
   public boolean bindService(Intent service, ServiceConnection conn,
           int flags) {
       warnIfCallingFromSystemProcess();
       return bindServiceCommon(service, conn, flags, mMainThread.getHandler(),
               Process.myUserHandle());
   }
  ```
  ```java
  private boolean bindServiceCommon(Intent service, ServiceConnection conn, int flags, Handler
             handler, UserHandle user) {
         IServiceConnection sd;
         if (conn == null) {
             throw new IllegalArgumentException("connection is null");
         }
         if (mPackageInfo != null) {
             sd = mPackageInfo.getServiceDispatcher(conn, getOuterContext(), handler, flags);
         } else {
             throw new RuntimeException("Not supported in system context");
         }
         validateServiceIntent(service);
         try {
             IBinder token = getActivityToken();
             if (token == null && (flags&BIND_AUTO_CREATE) == 0 && mPackageInfo != null
                     && mPackageInfo.getApplicationInfo().targetSdkVersion
                     < android.os.Build.VERSION_CODES.ICE_CREAM_SANDWICH) {
                 flags |= BIND_WAIVE_PRIORITY;
             }
             service.prepareToLeaveProcess(this);
             int res = ActivityManagerNative.getDefault().bindService(
                 mMainThread.getApplicationThread(), getActivityToken(), service,
                 service.resolveTypeIfNeeded(getContentResolver()),
                 sd, flags, getOpPackageName(), user.getIdentifier());
             if (res < 0) {
                 throw new SecurityException(
                         "Not allowed to bind to service " + service);
             }
             return res != 0;
         } catch (RemoteException e) {
             throw e.rethrowFromSystemServer();
         }
     }
  ```
  注意上面方法中的 *validateServiceIntent* 方法,这个就是上面报错的原因所在了.
  ```java
  private void validateServiceIntent(Intent service) {
       if (service.getComponent() == null && service.getPackage() == null) {
           if (getApplicationInfo().targetSdkVersion >= Build.VERSION_CODES.LOLLIPOP) {
               IllegalArgumentException ex = new IllegalArgumentException(
                       "Service Intent must be explicit: " + service);
               throw ex;
           } else {
               Log.w(TAG, "Implicit intents with startService are not safe: " + service
                       + " " + Debug.getCallers(2, 3));
           }
       }
   }
  ```
  可以看到,这里对intent内部的package属性做了判断,如果大于APIlevel21并且package为空的话就会抛出上面提到的异常.
  解决方法很简单,在intent初始化的时候设置一下package就可以了.
  ```java
  Intent intent = new Intent(action);
  intent.setPackage(getPackageName());
  bindService(intent);
  ```
  > 注意:如果在使用AIDL时,这里传入的package应该是服务端对应的包名,也就是你想让谁来处理这个intent,这里的package就传谁的.
  > 注意:这里getPackageName这个方法返回的是application的包名,也就是apk的包名,不要和java反射的获取包名方法混淆了,java的获取包名是通过class.getPackage().getName() 来获取的.
