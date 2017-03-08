layout: "post"
title: "android自定义gradle插件"
date: "2017-01-23 13:31"
---
## android自定义gradle插件
  在android学习过程中,理解gradle的编译过程是非常重要的,我们可以通过自定义gradle插件来达到在编译打包的过程中人工参与其中的一部分工作,这样可以满足我们的各种需求,比如在编译之前要对所有的class做统一处理,比如打包完成之后要输出到指定的目录,这些都可以通过自定义的gradle插件来完成.

  下面讲解下如何在android studio中实现自定义的gradle插件

## 在当前工程的gradle文件中实现插件
  gradle插件理论上就是通过groovy语法实现的类,内部可以制定一些流程task来完成相应的工作.最简单的肯定是在当前工程的 *build.gradle* 文件中直接实现插件内容.因为实际上 *build.gralde* 就是一个groovy文件.

  在我们的android主工程目录中输入一下代码:

  ```groovy
  class GreetingPlugin implements Plugin<Project> {

    @Override
    void apply(Project project) {
        project.extensions.create("greeting", GreetingPluginExtension)
        project.task('hello') {
            doLast {
                println "${project.greeting.message} from ${project.greeting.greeter}"
            }
        }
    }
 }

 class GreetingPluginExtension {
    String message = 'Hello from GreetingPlugin'
    String greeter
 }
 ```
 这里首先是实现一个plugin接口,这个接口是所有的自定义插件的必须实现的.完成里面的 *apply* 方法,这里用到了一个 *extensions* 的概念,这个 *extensions* 是用来给工程project设置一些额外属性的对象.比如我们在android的 *build.gradle* 文件中经常看到的
 ```
 android{
   ....
 }
 ```
 这个 *android{}* 内部就是给工程中设置的一些参数.在咱们这里的设置的是 *extensions* 叫 *GreetingPluginExtension* ,内部定义了两个参数 *message* 和 *greeter* 这里前一个参数有默认值,后一个没有,其实groovy和java的语法基本上一样的,只不过groovy是动态的java.

接着看plugin中的 *apply* 方法,非常的简单就是定义一个名叫 *hello* 的 *task* ,然后在里面打印一下  *GreetingPluginExtension* 中的两个参数.那在 *build.gradle* 中如何设置对应的参数呢?看下面的代码:
```
greeting {
    message  'Hi from gradle'
    greeter  'Gradle'
}
```  
就是这么简单直白.定义完成之后,我们直接在主目录下终端运行一下命令:
```
./gradlew -q hello
```
可以看到在终端中输出:
```
Hi from gradle from Gradle
```

是不是非常的简单呢.在当前工程中定义的插件只能自己用,如果要给别人用的话就需要通过一个单独的工程来生成插件了.

### 在单独工程中创建自定义gradle插件

首先在当前的工程中随便创建一个android module ,然后src/main目录下所有文件夹全部删除掉.然后先创建一个groovy文件夹,里面用来存放我们的插件groovy代码,一个是 *GreetingPluginExtension* ,一个是 *MyGradlePlugin* 内部代码分别对应上面的两个class.

然后再创建一个 *resources* 文件夹,里面放入一个 *META-INFO* 文件夹,然后继续在 *META-INFO* 文件夹下创建 *gradle-pulgins* ,在 *gradle-pulgins* 下创建文件
*com.justyan.gradle.properties* 文件,文件内容为:
```
implementation-class = com.justyan.gradleplugin.MyGradlePlugin
```
这里的 *com.justyan.gradle.properties* 文件的 *com.justyan.gradle* 就是你以后要给其他工程引用的的 gradle ID.比如这样:
```
apply plugin: 'com.justyan.gradle'
```
这个名称你可以随便起,只要在引用的时候引用对了就ok.文件中内容的目的是指定了插件的类路径,这里就是groovy文件下的那两个groovy文件.

最后的文件目录应该是这样的:
![插件工程的目录结构](images/2017/01/android自定义gradle插件的目录结构.png)

这里的 *META-INFO* 和 *gradle-pulgins* 是android studio给合并显示了,实际上是一个层级关系.

这样插件代码就完成了,剩下的就是如果打包发布的问题了.

在创建的插件的module工程中的 *build.gradle* 文件中输入一下内容:
```
apply plugin: 'groovy'
apply plugin: 'maven'

version = '1.0.0'
group = 'com.justyan'
archivesBaseName = 'mygradleplugin'
repositories {
    mavenCentral()
}


dependencies {
    compile gradleApi()
    compile localGroovy()
}
// 一定要记得使用交叉编译选项，因为我们可能用很高的JDK版本编译，为了让安装了低版本的同学能用上我们写的插件，必须设定source和target
compileGroovy {
    sourceCompatibility = 1.7
    targetCompatibility = 1.7
    options.encoding = "UTF-8"
}
uploadArchives {
    repositories {
        mavenDeployer {
            repository(url: "file:////Users/justyan/Downloads/repo")
        }
    }
}

```

version用来表示maven生成的jar包版本号,group表示属于哪个组织, *archivesBaseName* 表示生成的java的名称.最后在 *uploadArchives* 中指定了生成的maven目录在哪里.这个目录你可以随意指定,只要后续引用的时候前后一致就可以.

完成这个之后,我们调用这个工程的 *uploadArchives* task:
```
./gradlew -q uploadArchives
```
然后就会在上面的指定目录下生成对应的maven文件了.

最后,来看下如何在其他工程中引用.

在要引用的工程的顶层目录的 *build.gralde* 文件中输入:
```
buildscript {
    repositories {
        jcenter()
        maven {
            url uri('file:////Users/justyan/Downloads/repo')
        }
    }
    dependencies {
        classpath 'com.android.tools.build:gradle:2.2.2'
        classpath group: 'com.justyan', name: 'mygradleplugin',
                version: '1.0.0'
        // NOTE: Do not place your application dependencies here; they belong
        // in the individual module build.gradle files
    }
}

```
首先指定maven文件目录,和上面的对应,然后在 *dependencies* 中输入组名,插件名,版本号.这样对插件的引用就完成了.最后在要使用的工程中的 *build.gralde* 文件中顶部输入:
```
apply plugin: 'com.justyan.gradle'
```
然后用android studio同步一下gradle就可以了.这个时候在旁边的gradle project中是可以看到 *hello* 这个task的,表明插件加载成功.
![task列表](images/2017/01/gradle project中的新增的task.png)

最后如果我们要使用这个task和上面的方法一样,在终端中输入:
```
./gradlew -q hello
```
可以看到在终端中输出:
```
Hi from gradle from Gradle
```
这样自定义插件就全部搞定了,后续可以根据需求来自己实现插件中的内容了.

### 参考资料
[cutom_plugin][62e8eb4d]

  [62e8eb4d]: https://docs.gradle.org/current/userguide/custom_plugins.html "cutom_plugin"

[在 Android Studio 中自定义 Gradle 插件][1127106a]

  [1127106a]: https://gold.xitu.io/entry/577bc26e165abd005530ead8 "在 Android Studio 中自定义 Gradle 插件"
