---
layout: "post"
title: "IOS知识点集合"
date: "2017-04-13 13:50"
---
这里主要列出一些在IOS学习工程中用到的知识点，包括他们的使用场景和注意事项等等。
### NSCharacterSet的用法
  首先要明白 *NSCharacterSet* 与 *NSSet* 二者之间没有关系，不要以为前者是后者的子集。
  *NSCharacterSet* 主要是用来预置一下char在里面作为一个集合，方便做过滤与对比。它的内部预置了很多默认的集合类型，比如：

  >controlCharacterSet
  >whitespaceCharacterSet
  >whitespaceAndNewlineCharacterSet
  >decimalDigitCharacterSet
  >letterCharacterSet
  >lowercaseLetterCharacterSet
  >uppercaseLetterCharacterSet
  >nonBaseCharacterSet
  >alphanumericCharacterSet
  >decomposableCharacterSet
  >illegalCharacterSe;
  >punctuationCharacterSet
  >capitalizedLetterCharacterSet
  >symbolCharacterSet

  通过字面很容易理解这些类型都是有什么作用。我们可以在对字符串做trim处理的时候通过它来完成一些操作。下面是常用的示例代码：
  1. 去掉字符串两边的空格
  ```Objective-c
  NSString * testStr = @" this is a test string ";
  testStr = [testStr stringByTrimmingCharactersInSet:[NSCharacterSet whitespaceCharacterSet]];
  ```
  注意：上面的处理方法只会去掉两边的空格，中间是没有处理，如果需要处理中间的空格，可以通过把字符串拆分成数组，通过NSPredicate处理完数组之后再合并成字符串来完成。
  2. 去掉字符串所有的空格
  ```Objective-c
  NSString * testStr = @" this is a test string ";
  NSArray *array = [testStr componentsSeparatedByCharactersInSet:[NSCharacterSet whitespaceCharacterSet]];
  [array filteredArrayUsingPredicate:[NSPredicate predicateWithFormat:@"self <> ''"]];
  testStr = [array componentsJoinedByString:@""];
  NSLog(@"the test string is %@",testStr);
  ```


### NSPredicate的用法  

  *NSPredicate* 直译过来就是 *谓词* 的意思，常用对集合做各种符合条件的过滤，在一点上感觉OC比java方便很多，在java中你还得自己去实现数组过滤器也就是谓词，OC是预置好了的，使用起来非常方便。

  常用的方法主要有两个：
  1. NSPredicate predicateWithFormat
  2. NSPredicate predicateWithBlock

  第一个后面需要输入格式化字符串，后面那个是直接通过一个block来完成的。后面那个有点像java8提供的predicate接口方法。
  如果要熟练使用第一种格式化方式，需要了解一下类似sql的语法规则：
  1. self代指集合中遍历的单个item
  2. < ,> ,<=,>= 表示了小于，大于这些比较
  3. <> 和 != 是一样的，都表示不等于
  4. 字符串相关：BEGINSWITH、ENDSWITH、CONTAINS,表示字符串的比较关系，下面的示例会有体现
  5. 通配符LIKE
  6. 正则 MATCHS

  ```Objective-c
   NSArray *testArr = @[@"lucy",@"jack",@"david",@"justyan"];

   NSPredicate *predicate = [NSPredicate predicateWithFormat:@" self like %@",@"*y*"];
   NSPredicate *predicate2 = [NSPredicate predicateWithFormat:@" self <> %@",@"lucy"];

   NSArray *newArr = [testArr filteredArrayUsingPredicate:predicate];
   NSLog(@"the newArr is %@",newArr);
  ```
  在上面的示例中第一个predicate表示只保留数组中包含该字母 “y” 的item。
  第二个predicate表示保留不等于 “lucy” 的item。
  可以看到用法还是很容易理解的。

### NSScanner的用法
  NSScanner是用来对字符串做各种匹配赋值操作的类。有点类似javaScript中的 *eval* 方法，比如当你输入一个字符串 “1+1” 的时候，如果通过eval函数处理它会直接输出 ”2”，相当于智能的理解了字符串的含义。

  NSScanner常用来获取字符串中想要的值，比如int，string，等等。常用的主要有下面几个方法：
  1. - (BOOL)scanString:string intoString:result;
  2. - (BOOL)scanUpToString:string intoString:result;
  3. - (BOOL)scanInt:result;
  4. - (BOOL)scanUpToCharactersFromSet:set intoString:result;

  首先来看第一个方法 *scanString* 表示把第一个参数识别的字符串放入后面的result中。如果后面的result传入null表示跳过前面一个参数的字符串。

  第二个方法 *scanUpToString* 表示截取字符串到第一个参数。比如“ the test string” 如果传入第一个参数是 “test” 的话表示截取到 “test” 为止，那么第二个参数的值就是 " the "

  第三个方法 *scanInt* 表示截取字符串里面的int值，并把int值放入第二个参数中。比如 ”123 test” 会返回123

  第四个方法 *scanUpToCharactersFromSet* 与第二个方法类似，只是截取的节点用的传入的CharcterSet中预置的字符

  下面通过一个示例来完成展现一个上面方法的用法：
  ```Objective-c
     NSString *str = @"product: the apple mac ;price: 20000 ;number:5 ";
     NSString *str2 = @"product: the apple iphone ;price: 6999 ;number:10 ";
     NSString *str3 = @"product: the apple ipad ;price: 4000 ;number:20 ";

     str = [str stringByAppendingString:str2];
     str = [str stringByAppendingString:str3];
    NSScanner *scanner = [NSScanner scannerWithString:str];
       NSString *productName;
       int price ;
       int count;
       NSCharacterSet *set = [NSCharacterSet characterSetWithCharactersInString:@";"];
       while (![scanner isAtEnd]) {
           [scanner scanString:@"product:" intoString:nil];
           [scanner scanUpToCharactersFromSet:set intoString:&productName];
//         [scanner scanUpToString:@";" intoString:&productName];
           [scanner scanString:@";price:" intoString:nil];
           [scanner scanInt:&price];
           [scanner scanUpToString:@";number:" intoString:nil];
           [scanner scanString:@";number:" intoString:nil];
           [scanner scanInt:&count];

           NSLog(@"the product is %@,the price is %d ,the number is %d",productName,price,count);
       }
  ```
  输出：
  ```
     the product is the apple mac ,the price is 20000 ,the number is 5
     the product is the apple iphone ,the price is 6999 ,the number is 10
     the product is the apple ipad ,the price is 4000 ,the number is 20
  ```
