---
layout: post
title: android知识点
date: '2016-03-1 22:00'
tags:
  - android
  - blog
  - github
---

android知识点汇总

## ActivityLifecycleCallbacks的使用
  ActivityLifecycleCallbacks用来在application中统一监听activity的生命周期回调

## 为什么在theme设置layout_margin等以layout_开头的属性无效
  这是因为以layout_开头的属性是用来告诉父控件如何放置该控件位置的，必须明确了外部viewGroup的类型之后才可以生效，   所以在theme中设置无效，需要在具体调用的xml中设置对应的style才可以生效

  [  To use layout_marginLeft in a button style applied as a theme?][3da2c9b8]

## 如何修改全局的view属性，比如改变当前activity或者application的button背景
  方法一.theme中引用style，然后设置为当前activity的theme

```xml
  <?xml version="1.0" encoding="utf-8"?>
<resources>
  <style name="MyRedTheme" parent="android:Theme.Light">
    <item name="android:textAppearance">@style/MyRedTextAppearance</item>
  </style>
  <style name="MyRedTextAppearance" parent="@android:style/TextAppearance">
    <item name="android:textColor">#F00</item>
    <item name="android:textStyle">bold</item>
  </style>
</resources>
```

  方法二.自定义一个view

```java
  public class RedTextView extends TextView{
    public RedTextView(Context context, AttributeSet attrs) {
        super(context, attrs);
        setTextColor(Color.RED);
    }
  }
```

  [  Setting global styles for Views in Android][08902e5d]
 <!--more-->
## 动态设置textView的drawableLeft
  方法一.
  ```java
  Drawable img = getContext().getResources().getDrawable( R.drawable.smiley );
  img.setBounds( 0, 0, 60, 60 );
  txtVw.setCompoundDrawables( img, null, null, null );
  ```
  方法二.
  ```java
  Drawable img = getContext().getResources().getDrawable( R.drawable.smiley );
  txtVw.setCompoundDrawablesWithIntrinsicBounds( img, null, null, null);
  ```
  方法三.
  ```java
  txtVw.setCompoundDrawablesWithIntrinsicBounds( R.drawable.smiley, 0, 0, 0);
  ```
  [  How to programatically set drawableLeft on Android button?][668016e2]

## 代码中设置TextView或者RadioButton的文字selector
  常用的做法是在res/color文件夹下放置对应的selector，然后在xml中引用 android:textColor="@color/selector",本以为在java代码中调用
  setTextColor(context.getResources().getColor(R.color.selector))好使的,结果没有起作用。。。。
  后来在stackoverflow中找到了解决方法，调用setTextColor(context.getResources().getColorStateList(R.color.selector));解决了问题

  [  Using selector to change TextView text color][7f26391d]

  [7f26391d]: http://stackoverflow.com/questions/9982182/using-selector-to-change-textview-text-color "Using selector to change TextView text color"

## popupWindow无法嵌入fragment
  原因是fragment需要放入归属于activity中的某个容器，popupWindow不归属于ativity所以无法嵌入fragment

  [Properly creating a fragment in a PopupWindow][ebd60c09]

## java代码中设置editText的输入类型 TYPE_NUMBER_FLAG_DECIMAL 无效的问题
  需要设置为
  ```java
  editText.setInputType(InputType.TYPE_NUMBER_FLAG_DECIMAL|InputType.TYPE_CLASS_NUMBER);
  ```
  才能生效。并且如果在xml中设置了
  ```xml
  android:inputType="numberDecimal"
  ```
  代码中判断
  ```java
  editText.getInputType() == InputType.TYPE_NUMBER_FLAG_DECIMAL
  ```
  返回false，道理是一样的，应该判断为
  ```java
  editText.getInputType() == (InputType.TYPE_NUMBER_FLAG_DECIMAL | InputType.TYPE_CLASS_NUMBER)
  ```

  [android:inputType=“numberDecimal” vs. InputType.TYPE_NUMBER_FLAG_DECIMAL][84d7fa5f]

## android 中的LayoutInflater.inflate和View.inflate方法的区别
  实际上View.inflate内部调用了LayoutInflater.inflate(resource, root, root != null)方法
  ```java
  public View inflate(@LayoutRes int resource, @Nullable ViewGroup root, boolean attachToRoot)
  ```
  在上面的inflate方法中，第一个是布局id，第二个是父容器对象，第三个是是否把view添加到前面的父容器中。
  默认情况下如果第二个父容器传入null，那么view的layoutParam属性会被清空，也就是说view在父容器中的位置信息（lyaout_param）会丢失.所以在以后的使用的时候
  应该把父容器对象传入，但是第三个参数设置为false不把view添加到父容器中。

  [LayoutInflater.inflate方法解析][cc462667]

## android创建自定义属性的注意事项
  在主工程中使用自定义属性的时候命名空间使用的当前程序的包名，如果是在子工程中定义的属性，命名空间需要使用 “res-auto”
  ```
  xmlns:custom="http://schemas.android.com/apk/res-auto"
  ```


[cc462667]: http://bxbxbai.github.io/2014/11/19/make-sense-of-layoutinflater/ "LayoutInflater.inflate方法解析"

[84d7fa5f]: http://stackoverflow.com/questions/31914388/androidinputtype-numberdecimal-vs-inputtype-type-number-flag-decimal "android:inputType=“numberDecimal” vs. InputType.TYPE_NUMBER_FLAG_DECIMAL"

[668016e2]: http://stackoverflow.com/questions/4502605/how-to-programatically-set-drawableleft-on-android-button "How to programatically set drawableLeft on Android button?"

[08902e5d]: http://stackoverflow.com/questions/3078081/setting-global-styles-for-views-in-android "Setting global styles for Views in Android"

[3da2c9b8]: http://stackoverflow.com/questions/3775587/to-use-layout-marginleft-in-a-button-style-applied-as-a-theme "To use layout_marginLeft in a button style applied as a theme?"

[ebd60c09]: http://stackoverflow.com/questions/8044593/properly-creating-a-fragment-in-a-popupwindow "Properly creating a fragment in a PopupWindow"
