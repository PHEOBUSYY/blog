---
layout: "post"
title: "关于JAVA和ANDROID的注解（ANNOTATION）解析-终极教程(未完)"
date: "2016-02-19 16:00"
tags: [java, blog, github]
---

**前言:** 注解在java中是一个重要的特性，每一个开发者都应该知道使用他们.

  我们在Java Code Geeks提供了大量的教程,比如 [Creating Your Own Java Annotation](http://www.javacodegeeks.com/2014/07/creating-your-own-java-annotations.html),[Java Annotation Tutorial with Custom Annotation](http://www.javacodegeeks.com/2014/07/creating-your-own-java-annotations.html) and [Java Annotation;Explored & Explained](http://www.javacodegeeks.com/2012/08/java-annotations-explored-explained.html).

  我们也精选了在各种框架中使用注解的文章,比如 [Make your Spring Security @Secured annotations more DRY](http://www.javacodegeeks.com/2012/06/make-your-spring-security-secured.html) and [Java Annotation & A Real World Spring Example](http://www.javacodegeeks.com/2012/01/java-annotations-real-world-spring.html).

  现在,为了让你愉快的学习,我们把这些关于注解信息整合到这个帖子中，好好享受吧！
<!--more-->
	目录
	1. 为什么注解
	2. 介绍
	3. 使用
	4. 语法和元素
	5. 使用范围
	6. 使用示例
	7. 创建注解
	8. java 8 和注解
	9. 自定义注解
	10. 可检索的注解
	11. 继承注解
	12. 已知库的注解
	13. 总结
	14. 下载
	15. 资源

在这篇文章中，我们讲解什么是java注解，它如何工作并且我们如何在java中使用.

我们将看到如何在java中开箱即用注解，这种称为元注解或者构	造注解和在java8中有关注解的新特性.

最后我们将提供一个自定义注解和注解解析程序在java中通过反射来使用

我们将提供一些基于注解的知名类库，比如Junit，JAXB，Spring and Hibernate.

文章的最后我们提供教程中使用到的示例代码，在代码中我们基于以下软件版本的构建环境：
- Eclipse Luna 4.4
- JRE Update 8.20
- Junit 4
- Hibernate 4.3.6
- FindBugs 3.0.0

### 1. 为什么注解?
  注解在j2se 5引入，目的是为了提供一种在代码中写入元数据的机制给编程人员.
	在注解出现之前，程序员描述代码的方式没有统一标准，并且每个人有自己的方法：
