---
layout: "post"
title: "JAVA中的保留两位小数"
date: "2016-02-19 16:00"
tags: "java blog github"
---



在java中，保留两位小数有很多种方法，其中主要思路有痛过NumberFormat等格式化数字或者通过BigDecimal来设置精度处理，下面列出了常用的4种方法。

<!--more-->
## 一. JAVA方法实现
### 方法一
  通过设置BigDecimal的精度来处理

```java
  public static String parseDoubleDecimal(double d) {
    BigDecimal bigDecimal = new BigDecimal(d);
    return bigDecimal.setScale(2, BigDecimal.ROUND_HALF_UP).doubleValue()+"";
  }
```

### 方法二
  通过DecimalFormat的setMaximumFractionDigits设置最长小数位数

```java
  public static String parseDoubleDecimal2(double d) {
    NumberFormat decimalFormat = DecimalFormat.getInstance();
    decimalFormat.setMaximumFractionDigits(2);
    return decimalFormat.format(d);
  }
```

### 方法三
  通过DecimalFormat的format方法来格式化

```java
  public static String parseDoubleDecimal3(double d) {
    NumberFormat decimalFormat = new  DecimalFormat("##.00");
    return decimalFormat.format(d);
  }
```

### 方法四
  通过String的format方法来格式化

```java
  public static String parseDoubleDecimal4(double d) {
    return String.format("%.2f",d);
  }
```

### 方法五
  通过Formatter的format方法来格式化，和方法四类似

```java
  public static String parseDoubleDecimal5(double d) {
    return new Formatter().format("%.2f", d);
  }
```

## 补充
  什么情况下可以避免科学计数法，和NumberFormat的format方法中pattern详解

### 如何避免科学计数法
  在java中，如果浮点数大于999999.0的时候，会通过科学计数法莱显示浮点数，所以在编程的时候要注意   如何避免科学计数法，可以通过下面提供的方法来避免科学计数法

```java
public static void main(String[] args) {
      BigDecimal bigDecimal = new BigDecimal("123456789.123456789");  
      String result = bigDecimal.toString();  
      System.out.println(result);
}
```

```java
public static void main(String[] args) {
    Double double1 = 123456789.123456789;  
    DecimalFormat decimalFormat = new DecimalFormat("#,##0.00");//格式化设置  
    System.out.println(decimalFormat.format(double1));  
    System.out.println(double1);
}
```
### 关于format的pattern详解
  pattern转义符表格

|符号  | 位置  |含义  |
|:-:|:----------:|:--:|
 |0|  数字     |  数字  |
 |#|  数字     |  数字，如果是0就不显示|
 |.|  数字     |  小数和整数分隔符|
 |-|  数字     |  负数|
 |,|  数字     |  整数分隔符|
 |E|  数字     |  科学计数法分隔符|
 |;|  分隔符    | 多个pattern之间分隔符|
 |%|  前缀后缀  | 百分位|
 |'|  前缀后缀|转义符号|
|\u2030|  前缀后缀|千分位|
|\u00A4|  前缀后缀|货币符号|


```java
public class TestDecimalFormatPattern {
    public static double d = 100000.0009;

    public static void main(String[] args) {
        testPattern("##.##");
        testPattern("00.##");
        testPattern("##.00");
        testPattern(",###.00");
        testPattern(",##.00");
        testPattern("0.00E0");
        testPattern("##%");
        testPattern("###,##0.00");
        testPattern("###\u2030");
        testPattern("###\u00A4");
        testPattern("'#'#.##");
        testPattern("''##.##");
        testPattern("AAAA##.##");

    }

    public static void testPattern(String pattern) {
        DecimalFormat decimalFormat = new DecimalFormat(pattern);
        System.out.println("pattern=" + pattern + " ----->" + decimalFormat.format(d));
    }
}
```
    输出:
    pattern=00.## ----->100000
    pattern=##.00 ----->100000.00
    pattern=,###.00 ----->100,000.00
    pattern=,##.00 ----->10,00,00.00
    pattern=0.00E0 ----->1.00E5
    pattern=##% ----->10000000%
    pattern=###,##0.00 ----->100,000.00
    pattern=###‰ ----->100000001‰
    pattern=###¤ ----->100000￥
    pattern='#'#.## ----->#100000
    pattern=''##.## ----->'100000
    pattern=AAAA##.## ----->AAAA100000

### 参考资料
1. [java保留两位小数][bd956b56]
2. [DecimalFormat api][bf1cdc4e]

  [bf1cdc4e]: https://docs.oracle.com/javase/7/docs/api/java/text/DecimalFormat.html "DecimalFormat"
  [bd956b56]: http://blog.csdn.net/yuhua3272004/article/details/3075436 "java保留两位小数"
