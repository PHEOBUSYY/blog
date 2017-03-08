layout: "post"
title: "android 开源Logger库"
date: "2017-01-11 09:05"
---
## android 开源Logger库

### 需要明白的问题

1. 如何打印当前线程的名称的
直接Thread.currentThread.getName调用的

2. 如何打印调用方法的名称和行数的
通过StackTraceElement来获取到引用的数组

3. StackTraceElement的概念
StackTraceElement是java1.5新增的用来输出方法调用栈信息的对象,通过它我们可以打印出当前线程中方法的调用栈,从里面可以可以获取到方法名称,行数,类名称等我们想要的信息

4. 如何增加分割线的和边框的
为了把打印日志用方框圈起来,这里把整个日志分成了两部分,顶部叫header+剩下的我们真正需要的信息.
header又包含了当前顶部分割线,线程名称,方法调用栈信息
最后再把底部分割线加上,正好组合成一个方框,这个方框是没有右边的,因为分割线都是顶满整行的

5. 如何保证打印日志没有超过最大字符个数限制
由于原生api的单次打印最大字符限制是4M,所以判断如果要打印的字符串超过4M就把字符串拆开成多段来分别打印

6. 如何处理JSON数据
JSONObject原生api居然提供了一个带缩进的toString方法....
同时Arrays居然也提供了一个deepToString的方法.....

7. 如何处理XML
通过java提供的Transformer API来打印XML

### 主要代码分析

其他的方法都非常简单,核心方法就是下面这个log方法:
```java
@Override public synchronized void log(int priority, String tag, String message, Throwable throwable) {
   if (settings.getLogLevel() == LogLevel.NONE) {
     return;
   }
   if (throwable != null && message != null) {
     message += " : " + Helper.getStackTraceString(throwable);
   }
   if (throwable != null && message == null) {
     message = Helper.getStackTraceString(throwable);
   }
   if (message == null) {
     message = "No message/exception is set";
   }
   int methodCount = getMethodCount();
   if (Helper.isEmpty(message)) {
     message = "Empty/NULL log message";
   }

   logTopBorder(priority, tag);
   logHeaderContent(priority, tag, methodCount);

   //get bytes of message with system's default charset (which is UTF-8 for Android)
   byte[] bytes = message.getBytes();
   int length = bytes.length;
   if (length <= CHUNK_SIZE) {
     if (methodCount > 0) {
       logDivider(priority, tag);
     }
     logContent(priority, tag, message);
     logBottomBorder(priority, tag);
     return;
   }
   if (methodCount > 0) {
     logDivider(priority, tag);
   }
   for (int i = 0; i < length; i += CHUNK_SIZE) {
     int count = Math.min(length - i, CHUNK_SIZE);
     //create a new String with system's default charset (which is UTF-8 for Android)
     logContent(priority, tag, new String(bytes, i, count));
   }
   logBottomBorder(priority, tag);
 }

```
注意,这里加锁来保证数据顺序的正确性
通过 *logHeaderContent(priority, tag, methodCount);* 来打印线程信息和方法调用栈信息.
```java
private void logHeaderContent(int logType, String tag, int methodCount) {
   StackTraceElement[] trace = Thread.currentThread().getStackTrace();
   if (settings.isShowThreadInfo()) {
     logChunk(logType, tag, HORIZONTAL_DOUBLE_LINE + " Thread: " + Thread.currentThread().getName());
     logDivider(logType, tag);
   }
   String level = "";

   int stackOffset = getStackOffset(trace) + settings.getMethodOffset();

   //corresponding method count with the current stack may exceeds the stack trace. Trims the count
   if (methodCount + stackOffset > trace.length) {
     methodCount = trace.length - stackOffset - 1;
   }

   for (int i = methodCount; i > 0; i--) {
     int stackIndex = i + stackOffset;
     if (stackIndex >= trace.length) {
       continue;
     }
     StringBuilder builder = new StringBuilder();
     builder.append("║ ")
         .append(level)
         .append(getSimpleClassName(trace[stackIndex].getClassName()))
         .append(".")
         .append(trace[stackIndex].getMethodName())
         .append(" ")
         .append(" (")
         .append(trace[stackIndex].getFileName())
         .append(":")
         .append(trace[stackIndex].getLineNumber())
         .append(")");
     level += "   ";
     logChunk(logType, tag, builder.toString());
   }
 }
```
获取到调用栈信息,从第三个开始获取,因为前两个都是系统的调用信息,忽略不计,从第三个开始真正的方法调用的.再加上设置里面要求打印的方法个数,组成了要打印的方法的最终个数.
```java
private int getStackOffset(StackTraceElement[] trace) {
   for (int i = MIN_STACK_OFFSET; i < trace.length; i++) {
     StackTraceElement e = trace[i];
     String name = e.getClassName();
     if (!name.equals(LoggerPrinter.class.getName()) && !name.equals(Logger.class.getName())) {
       return --i;
     }
   }
   return -1;
 }
```
剩下的打印主要信息的时候做个容量判断,如果没超过原声API的最大字符串限制直接打印,如果超了就拆成几段分别打印.
```java
byte[] bytes = message.getBytes();
   int length = bytes.length;
   if (length <= CHUNK_SIZE) {
     if (methodCount > 0) {
       logDivider(priority, tag);
     }
     logContent(priority, tag, message);
     logBottomBorder(priority, tag);
     return;
   }
   if (methodCount > 0) {
     logDivider(priority, tag);
   }
   for (int i = 0; i < length; i += CHUNK_SIZE) {
     int count = Math.min(length - i, CHUNK_SIZE);
     //create a new String with system's default charset (which is UTF-8 for Android)
     logContent(priority, tag, new String(bytes, i, count));
   }
   logBottomBorder(priority, tag)
```
非常的简单.

### 参考资料
[logger github][2af73b78]

  [2af73b78]: https://github.com/orhanobut/logger "logger github"
