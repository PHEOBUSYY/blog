layout: "post"
title: "android 自定义URI Router"
date: "2017-01-09 10:25"
---
## android 自定义URI Router
### 关于自定义Router学习之前的预备知识

1. 为什么要使用google的auto-service这个包
  如何才能获取一个接口的所有实现类?通过反射来遍历所有的类,然后判断该类是否是实现指定的接口,不过这样效率不太高,毕竟得对所有的类进行遍历;还有一种方法就是就是把实现了该接口的类的全路径名称放入一个配置文件中,这样在使用的时候,从配置文件中获取实现类就可以了.有点反着来的意思.那这里配置文件放在哪里呢?放在了jar包里面的META-INF这个文件夹中,这个文件夹中放的是jar包的一些签名和说明文件,通用的定义是把实现接口类的描述文件放在 META-INF/service/接口名称/中.注意,前面的这种方式是需要人工来完成的,每当你调整了实现接口的数量的话,就需要手动更新一下 META-INF文件,但是 auto-service可以自动完成这些工作,这个就是它的作用.
![auto-service生成的配置文件](images/2017/01/auto-service生成的配置文件.png)


2. 为什么要使用squareup:javapoet这个包
  用来通过封装的API来生成java源文件,非常的方便,这里还没有深入的学习

3. java的METE-INF的作用是说明?

  jar包中的签名,配置文件,清单文件,还有接口实现类的列表文件
4. java的serviceLoader类作用

  可以从 METE-INF 中取出指定接口的实现类名称列表,方便直接调用
  ```java
  public static void main(String[] args) {

       ServiceLoader<ITest> serviceLoader = ServiceLoader.load(ITest.class);
       for (ITest iTest : serviceLoader) {
           iTest.test();
       }
   }

  ```

5. 自定义的注解是怎么起作用的
  实现步骤如下:
  1. 顶部工程的gradle中配置:
  ```java
  dependencies {
      classpath 'com.android.tools.build:gradle:2.2.2'
      ....
      //引入apt插件
      classpath 'com.neenbedankt.gradle.plugins:android-apt:1.8'

  }
  ```
  2. 主工程的gradle中配置:
  ```java
  apply plugin: 'com.neenbedankt.android-apt'
  .....
  compile project (':annotationprocessor')
  apt project (':annotationprocessor')
  ```
  这里的 *annotationprocessor* 用来包含自定义的processor的java子工程

  3. 用来包含自定义processor的java工程中的gradle中配置:
  ```java
  dependencies {
    compile fileTree(dir: 'libs', include: ['*.jar'])
    compile project(':annotation')
    compile 'com.google.auto.service:auto-service:1.0-rc2'
    compile 'com.squareup:javapoet:1.7.0'
}

sourceCompatibility = "1.7"
targetCompatibility = "1.7"
  ```
这里的 *javapoet* 根据需求添加,默认可以不适用.
*annotation* 工程中存放的是自定义的 *annotation* 类文件

4. 最后就是java代码的书写了,首先是一个自定义的java注解类
```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.CLASS)
public @interface YYAnnotation {
    String value();
}

```
然后是自定义的解析processor
```java
@AutoService(Processor.class)
public class YYAnnotationProcessor extends AbstractProcessor {

    private Filer filer;
    private Messager messager;

    @Override
    public synchronized void init(ProcessingEnvironment processingEnvironment) {
        super.init(processingEnvironment);
        filer = processingEnvironment.getFiler();
        messager = processingEnvironment.getMessager();
    }


    @Override
    public SourceVersion getSupportedSourceVersion() {
        return SourceVersion.latestSupported();
    }
    @Override
    public Set<String> getSupportedAnnotationTypes() {
        Set<String> set = new HashSet<>();
        //这里是你要处理注解的集合
        set.add(YYAnnotation.class.getCanonicalName());
        return set;
    }

    @Override
    public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {
        try {
            Set<? extends Element> mainAppElement = roundEnv.getElementsAnnotatedWith(YYAnnotation.class);
            if (!mainAppElement.isEmpty()) {
                //这里实现自己的需求
                process(mainAppElement);
                return true;
            }
        } catch (Exception e) {
            messager.printMessage(Diagnostic.Kind.ERROR, e.getMessage());
        }

        return true;
    }
    private void process(Set<? extends Element> elementsAnnotatedWith) throws IOException {
      TypeElement typeElement = (TypeElement) elementsAnnotatedWith.iterator().next();
      //这里的Config.FILE_NAME是你要生成的java源文件的名称
      JavaFileObject javaFileObject = filer.createSourceFile(Config.FILE_NAME, typeElement);
      //通过printWriter来生成java源文件
      PrintWriter writer = new PrintWriter(javaFileObject.openWriter());
      //这里的Config.PACKAGE_NAME你要生成的java源文件的包名
      writer.println("package " + Config.PACKAGE_NAME + ";");
      writer.println("public class TestProcessor {");
      writer.println("public static void test () {");

      YYAnnotation componentsAnnotation = typeElement.getAnnotation(YYAnnotation.class);
      String components = componentsAnnotation.value();
      writer.println("System.out.println(\""+components+"\");");

      writer.println("}");
      writer.println("}");

      writer.flush();
      writer.close();
    }

}

```
是不是非常的神奇,你可以再编译之前做一些事情,比如生成一个java文件,或者对一些java源文件做处理等等,给了我们很大的灵活性,在这里我们是通过apt生成了可以打印信息的java类,非常的简单,后期可以根据需求调整成更加复杂的逻辑.



6. 如何某个接口的所有实现类或者某个父类的所有子类
反射遍历所有的java类文件,找到实现该接口的类名称.

### 别人写的Router框架解析
首先要明白,Router的真正的作用是用在模块之间的解耦上,我们都知道android可以分为主工程+多个library,多个library可以作为单独module的体现,这里Router的作用就是可以让多个library互相之间调用各自的页面,service或者广播,而不用互相知道对方的具体类名称等信息,达到了内部解耦的功能.
先看工程目录结构:
![router工程目录结构](/images/2017/01/router工程目录结构.png)

这里主要的功能实现就在 annotation,router,routerhelper中实现的.其他的两个shoplib和bbslib代表了两个独立模块.主工程app,uitlslib和reslib内部定义了一些工具类,这里不过多的介绍了.

#### annotation工程

首先来看annotation,这个工程主要放置的是关于Router定义的各种注解.是一个纯java工程.
![](/images/2017/01/annotation项目类定义.png)

其中, *AutoRouter* , *StaticRouter* 是router注解的两种模式,分别对应自动根据绑定的class类型来给URI添加对应的前缀,比如如果绑定的是activity则前缀为 *activity://* ,而后者则是完全按照注解的value值来作为绑定的URI.这个后续在代码中就明白怎么用了.
*Components* 对应所有模块的注解,主要用在application中声明都有那几个需要Router的模块.
*Component* 对应每个独立的模块
下面来看代码定义,非常的简单,就是普通的java注解声明:
```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.CLASS)
public @interface AutoRouter {
}
```

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.CLASS)
public @interface StaticRouter {
    String value();
}
```

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.CLASS)
public @interface Component {
    String value();
}
```

```java
@Retention(RetentionPolicy.CLASS)
@Target(ElementType.TYPE)
public @interface Components {
    String[] value();
}
```
#### routerhelper工程
上面再 *annotation* 项目中完成了注解的定义,那势必会有一个 *processor* 来完成注解的解析和逻辑处理,这里如果对注解的处理流程不是很熟悉的话可以看下相关的java注解处理的资源.这里就不重点介绍了.直接进入 *RouterHelper* 这个解析注解类中.
```java
@AutoService(Processor.class)
public class RouterProcessor extends AbstractProcessor {

    private Filer mFiler;
    private Messager mMessager;

    private Map<String, String> mStaticRouterMap = new HashMap<>();
    private List<String> mAutoRouterList = new ArrayList<>();

    @Override
    public synchronized void init(ProcessingEnvironment processingEnvironment) {
        super.init(processingEnvironment);
        mFiler = processingEnvironment.getFiler();
        mMessager = processingEnvironment.getMessager();
    }

    @Override
    public SourceVersion getSupportedSourceVersion() {
        return SourceVersion.latestSupported();
    }

    @Override
    public Set<String> getSupportedAnnotationTypes() {
        Set<String> set = new HashSet<>();
        set.add(AutoRouter.class.getCanonicalName());
        set.add(StaticRouter.class.getCanonicalName());
        set.add(Component.class.getCanonicalName());
        set.add(Components.class.getCanonicalName());
        return set;
    }

    @Override
    public boolean process(Set<? extends TypeElement> set, RoundEnvironment roundEnvironment) {
        mStaticRouterMap.clear();
        mAutoRouterList.clear();

        try {
            Set<? extends Element> mainAppElement = roundEnvironment.getElementsAnnotatedWith(Components.class);
            if (!mainAppElement.isEmpty()) {
                processInstaller(mainAppElement);
                return true;
            }
            processComponent(roundEnvironment);
        } catch (Exception e) {
            mMessager.printMessage(Diagnostic.Kind.ERROR, e.getMessage());
        }

        return true;
    }

    private void processInstaller(Set<? extends Element> mainAppElement) throws IOException {
        TypeElement typeElement = (TypeElement) mainAppElement.iterator().next();
        JavaFileObject javaFileObject = mFiler.createSourceFile(Config.ROUTER_MANAGER, typeElement);
        PrintWriter writer = new PrintWriter(javaFileObject.openWriter());

        writer.println("package " + Config.PACKAGE_NAME + ";");
        writer.println("public class " + Config.ROUTER_MANAGER + " {");
        writer.println("public static void " + Config.ROUTER_MANAGER_METHOD + "() {");

        Components componentsAnnotation = typeElement.getAnnotation(Components.class);
        String[] components = componentsAnnotation.value();
        for (String item : components) {
            writer.println(Config.FILE_PREFIX + item + ".router();");
        }

        writer.println("}");
        writer.println("}");

        writer.flush();
        writer.close();
    }

    private void processComponent(RoundEnvironment roundEnvironment) throws Exception {
        Set<? extends Element> compElements = roundEnvironment.getElementsAnnotatedWith(Component.class);
        if (compElements.isEmpty()) { return;}

        Element item = compElements.iterator().next();
        String componentName = item.getAnnotation(Component.class).value();

        Set<? extends Element> routerElements = roundEnvironment.getElementsAnnotatedWith(StaticRouter.class);
        for (Element e : routerElements) {
            if (! (e instanceof TypeElement)) { continue;}
            TypeElement typeElement = (TypeElement) e;
            String pattern = typeElement.getAnnotation(StaticRouter.class).value();
            mStaticRouterMap.put(pattern, typeElement.getQualifiedName().toString());
        }

        Set<? extends Element> autoRouterElements = roundEnvironment.getElementsAnnotatedWith(AutoRouter.class);
        for (Element e : autoRouterElements) {
            if (!(e instanceof TypeElement)) { continue;}
            TypeElement typeElement = (TypeElement) e;
            mAutoRouterList.add(typeElement.getQualifiedName().toString());
        }

        writeComponentFile(componentName);
    }

    private void writeComponentFile(String componentName) throws Exception {
        String className = Config.FILE_PREFIX + componentName;
        JavaFileObject javaFileObject = mFiler.createSourceFile(className);
//        javaFileObject.delete();

        PrintWriter printWriter = new PrintWriter(javaFileObject.openWriter());

        printWriter.println("package " + Config.PACKAGE_NAME + ";");

        printWriter.println("import android.app.Activity;");
        printWriter.println("import android.app.Service;");
        printWriter.println("import android.content.BroadcastReceiver;");

        printWriter.println("public class " + className + " {");
        printWriter.println("public static void router() {");

        // // Router.router(ActivityRule.ACTIVITY_SCHEME + "shop.main", ShopActivity.class);
        for(Map.Entry<String, String> entry : mStaticRouterMap.entrySet()) {
            printWriter.println("org.loader.router.Router.router(\"" + entry.getKey()
                    +"\", "+entry.getValue()+".class);");
        }

        for (String klass : mAutoRouterList) {
            printWriter.println("if (Activity.class.isAssignableFrom(" + klass + ".class)) {");
            printWriter.println("org.loader.router.Router.router(org.loader.router.rule.ActivityRule.ACTIVITY_SCHEME + \""
                    +klass+"\", " + klass + ".class);");
            printWriter.println("}");

            printWriter.println("else if (Service.class.isAssignableFrom(" + klass + ".class)) {");
            printWriter.println("org.loader.router.Router.router(org.loader.router.rule.ServiceRule.SERVICE_SCHEME + \""
                    +klass+"\", " + klass + ".class);");
            printWriter.println("}");

            printWriter.println("else if (BroadcastReceiver.class.isAssignableFrom(" + klass + ".class)) {");
            printWriter.println("org.loader.router.Router.router(org.loader.router.rule.ReceiverRule.RECEIVER_SCHEME + \""
                    +klass+"\", "+klass+".class);");
            printWriter.println("}");
        }

        printWriter.println("}");
        printWriter.println("}");
        printWriter.flush();
        printWriter.close();
    }
}
```
首先在顶部声明了 *AutoService* 对应本文最上面提到的它的作用用来更新METE-INF中接口的列表文件的.这样可以快速找到工程都有哪些 *processor* .

```java
@Override
public Set<String> getSupportedAnnotationTypes() {
    Set<String> set = new HashSet<>();
    set.add(AutoRouter.class.getCanonicalName());
    set.add(StaticRouter.class.getCanonicalName());
    set.add(Component.class.getCanonicalName());
    set.add(Components.class.getCanonicalName());
    return set;
}
```
用来声明这个processor可以支持处理哪些注解.是一个set集合
mFiler可以用来生成一些自定义的源文件,class或其他文件.
mMessager 可以把错误信息打印出来
重点解析注解的逻辑就在 *process* 方法中.
```java
Set<? extends Element> mainAppElement = roundEnvironment.getElementsAnnotatedWith(Components.class);
if (!mainAppElement.isEmpty()) {
    processInstaller(mainAppElement);
    return true;
}
```
来看第一个关键点,先是找到所有声明 *Components* 的类对象,这里注意由于 *Components* 声明的是 *ElementType.TYPE* 表明只能在类顶部定义,那么这里找到的element都是TypeElement表示类元素.
前面说到 *Components* 的作用是用来表示都有哪些模块需要Router的.比如这里在Application顶部声明了:
```java

@Components({"shop", "bbs"})
public class App extends MultiDexApplication {
  ....
}
```
说明了有两个工程工程需要Router功能,对应上面说的 shop和bbs两个模块.
回到 *process* 方法中,找到了 *Components* 的声明类并且不为空的话就进入 *processInstaller* 方法
```java
private void processInstaller(Set<? extends Element> mainAppElement) throws IOException {
       TypeElement typeElement = (TypeElement) mainAppElement.iterator().next();
       JavaFileObject javaFileObject = mFiler.createSourceFile(Config.ROUTER_MANAGER, typeElement);
       PrintWriter writer = new PrintWriter(javaFileObject.openWriter());

       writer.println("package " + Config.PACKAGE_NAME + ";");
       writer.println("public class " + Config.ROUTER_MANAGER + " {");
       writer.println("public static void " + Config.ROUTER_MANAGER_METHOD + "() {");

       Components componentsAnnotation = typeElement.getAnnotation(Components.class);
       String[] components = componentsAnnotation.value();
       for (String item : components) {
           writer.println(Config.FILE_PREFIX + item + ".router();");
       }

       writer.println("}");
       writer.println("}");

       writer.flush();
       writer.close();
   }
```
可以看到这里,通过mFiler对象创建了新的java源文件,位于 *Config.PACKAGE_NAME* 中的叫做 *Config.ROUTER_MANAGER* 的java文件,其内部定义了一个 *Config.ROUTER_MANAGER_METHOD* 方法,我们可以在编译完成后,在我们的工程中对应的目录中找到这个生成的源文件.
![processor生成的java源码](/images/2017/01/processor生成的java源码.png)

看到这里就发现我们居然可以自己生成java源文件,简直是打开了新世界的大门啊,用java来写java源文件,这本身就有种递归的感觉在里边,通过想象,我们可以做很多事情了,这个后续我们可以研究一下更多的用法.
生成的java类中的方法调用了每个模块的Router类的Router方法.可以看到这个Router是红色的,但是搜索一下发现,这个类居然也是生成的.那我们可以发现这个类肯定是通过 *processor* 方法中剩下那句代码来完成, 进入 *processComponent(roundEnvironment);*
```java
private void processComponent(RoundEnvironment roundEnvironment) throws Exception {
       Set<? extends Element> compElements = roundEnvironment.getElementsAnnotatedWith(Component.class);
       if (compElements.isEmpty()) { return;}

       Element item = compElements.iterator().next();
       String componentName = item.getAnnotation(Component.class).value();

       Set<? extends Element> routerElements = roundEnvironment.getElementsAnnotatedWith(StaticRouter.class);
       for (Element e : routerElements) {
           if (! (e instanceof TypeElement)) { continue;}
           TypeElement typeElement = (TypeElement) e;
           String pattern = typeElement.getAnnotation(StaticRouter.class).value();
           mStaticRouterMap.put(pattern, typeElement.getQualifiedName().toString());
       }

       Set<? extends Element> autoRouterElements = roundEnvironment.getElementsAnnotatedWith(AutoRouter.class);
       for (Element e : autoRouterElements) {
           if (!(e instanceof TypeElement)) { continue;}
           TypeElement typeElement = (TypeElement) e;
           mAutoRouterList.add(typeElement.getQualifiedName().toString());
       }

       writeComponentFile(componentName);
   }
```
可以看到,这里也很好理解,把所有的router声明类和URI放入集合中.通过后面的 *writeComponentFile* 方法来完成每个工程自己的router类生成.
之前读到这里这里的时候有个问题一直不明白:就是很明显关于 *Component* 注解应该有两个啊,但是这里却只是通过iterator的next方法获取注解集合中的一个然就去生成router类,那不就少了另外一个 *Component* 吗?,也就是说一个shop,一个bbs,都是注册了 *Component* 注解的,但是这里确只是从集合中取了一个,那后面那个就没人管了,岂不是少了一个.后来想明白了:
每个工程都是单独的调用 *processor* 来完成注解解析的,也就是说 shop工程中会调用一次这个 *processor* ,bbs工程中也会调用一次这个 *processor* ,这样关于 *component* 注解的处理就会执行两次,同时为每个模块各自生成一个 Router类.
```java
private void writeComponentFile(String componentName) throws Exception {
      String className = Config.FILE_PREFIX + componentName;
      JavaFileObject javaFileObject = mFiler.createSourceFile(className);
//        javaFileObject.delete();

      PrintWriter printWriter = new PrintWriter(javaFileObject.openWriter());

      printWriter.println("package " + Config.PACKAGE_NAME + ";");

      printWriter.println("import android.app.Activity;");
      printWriter.println("import android.app.Service;");
      printWriter.println("import android.content.BroadcastReceiver;");

      printWriter.println("public class " + className + " {");
      printWriter.println("public static void router() {");

      // // Router.router(ActivityRule.ACTIVITY_SCHEME + "shop.main", ShopActivity.class);
      for(Map.Entry<String, String> entry : mStaticRouterMap.entrySet()) {
          printWriter.println("org.loader.router.Router.router(\"" + entry.getKey()
                  +"\", "+entry.getValue()+".class);");
      }

      for (String klass : mAutoRouterList) {
          printWriter.println("if (Activity.class.isAssignableFrom(" + klass + ".class)) {");
          printWriter.println("org.loader.router.Router.router(org.loader.router.rule.ActivityRule.ACTIVITY_SCHEME + \""
                  +klass+"\", " + klass + ".class);");
          printWriter.println("}");

          printWriter.println("else if (Service.class.isAssignableFrom(" + klass + ".class)) {");
          printWriter.println("org.loader.router.Router.router(org.loader.router.rule.ServiceRule.SERVICE_SCHEME + \""
                  +klass+"\", " + klass + ".class);");
          printWriter.println("}");

          printWriter.println("else if (BroadcastReceiver.class.isAssignableFrom(" + klass + ".class)) {");
          printWriter.println("org.loader.router.Router.router(org.loader.router.rule.ReceiverRule.RECEIVER_SCHEME + \""
                  +klass+"\", "+klass+".class);");
          printWriter.println("}");
      }

      printWriter.println("}");
      printWriter.println("}");
      printWriter.flush();
      printWriter.close();
  }
```
这个 *writeComponentFile* 方法就是用来完成Router注册的,可以看到很直白的java代码,把java代码作为输入交给pritWriter来写入指定的文件中.
来看下两个生成的Router类的代码:
```java
package org.loader.router;
import android.app.Activity;
import android.app.Service;
import android.content.BroadcastReceiver;
public class Router_bbs {
public static void router() {
if (Activity.class.isAssignableFrom(org.loader.bbslib.BBSActivity.class)) {
org.loader.router.Router.router(org.loader.router.rule.ActivityRule.ACTIVITY_SCHEME + "org.loader.bbslib.BBSActivity", org.loader.bbslib.BBSActivity.class);
}
else if (Service.class.isAssignableFrom(org.loader.bbslib.BBSActivity.class)) {
org.loader.router.Router.router(org.loader.router.rule.ServiceRule.SERVICE_SCHEME + "org.loader.bbslib.BBSActivity", org.loader.bbslib.BBSActivity.class);
}
else if (BroadcastReceiver.class.isAssignableFrom(org.loader.bbslib.BBSActivity.class)) {
org.loader.router.Router.router(org.loader.router.rule.ReceiverRule.RECEIVER_SCHEME + "org.loader.bbslib.BBSActivity", org.loader.bbslib.BBSActivity.class);
}
}
}
```
```java
package org.loader.router;
import android.app.Activity;
import android.app.Service;
import android.content.BroadcastReceiver;
public class Router_shop {
public static void router() {
org.loader.router.Router.router("activity://shop.main", org.loader.shoplib.ShopActivity.class);
}
}
```
可以看到这两个通过代码生成源文件最后都是进入了Router类的router方法.
到这里关于注解的解析就完成,下面开始进入的Router的核心实现.

### router工程
到这里,才算真正开始Router核心实现,要知道两个模块互相调用对方的元素,而又不直接调用,势必会有一个中间的东西来完成Router功能,首先是每个模块把自己的对外开放的元素注册到中间人,然后在需要调用别的模块的元素的时候,通过URI从中间人中来匹配找到对方的元素.这就是一个完成的Router流程.
很明显,需要一个map结构,key是URI,value是对应的class对象或者className.
上面每个模块的Router类都是最终调用的这个工程的Router类的router方法:
```java
public class Router {

    /**
     * 添加自定义路由规则
     * @param scheme 路由scheme
     * @param rule 路由规则
     * @return {@code RouterInternal} Router真实调用类
     */
    public static RouterInternal addRule(String scheme, Rule rule) {
        RouterInternal router = RouterInternal.get();
        router.addRule(scheme, rule);
        return router;
    }

    /**
     * 添加路由
     * @param pattern 路由uri
     * @param klass 路由class
     * @return {@code RouterInternal} Router真实调用类
     */
    public static <T> RouterInternal router(String pattern, Class<T> klass) {
        return RouterInternal.get().router(pattern, klass);
    }

    /**
     * 路由调用
     * @param ctx Context
     * @param pattern 路由uri
     * @return {@code V} 返回对应的返回值
     */
    public static <V> V invoke(Context ctx, String pattern) {
        return RouterInternal.get().invoke(ctx, pattern);
    }

    /**
     * 是否存在该路由
     * @param pattern
     * @return
     */
    public static boolean resolveRouter(String pattern) {
        return RouterInternal.get().resolveRouter(pattern);
    }
}

```
这个router方法就是用来注册URI的.
可以看到所有的逻辑都是在 *RouterInternal* 中实现的.
```java
public class RouterInternal {

    private static RouterInternal sInstance;

    /** scheme->路由规则 */
    private HashMap<String, Rule> mRules;

    private RouterInternal() {
        mRules = new HashMap<>();
        initDefaultRouter();
    }

    /**
     * 添加默认的Activity，Service，Receiver路由
     */
    private void initDefaultRouter() {
        addRule(ActivityRule.ACTIVITY_SCHEME, new ActivityRule());
        addRule(ServiceRule.SERVICE_SCHEME, new ServiceRule());
        addRule(ReceiverRule.RECEIVER_SCHEME, new ReceiverRule());
    }

    /*package */ static RouterInternal get() {
        if (sInstance == null) {
            synchronized (RouterInternal.class) {
                if (sInstance == null) {
                    sInstance = new RouterInternal();
                }
            }
        }

        return sInstance;
    }

    /**
     * 添加自定义路由规则
     * @param scheme 路由scheme
     * @param rule 路由规则
     * @return {@code RouterInternal} Router真实调用类
     */
    public final RouterInternal addRule(String scheme, Rule rule) {
        mRules.put(scheme, rule);
        return this;
    }

    private <T, V> Rule<T, V> getRule(String pattern) {
        HashMap<String, Rule> rules = mRules;
        Set<String> keySet = rules.keySet();
        Rule<T, V> rule = null;
        for (String scheme : keySet) {
            if (pattern.startsWith(scheme)) {
                rule = rules.get(scheme);
                break;
            }
        }

        return rule;
    }

    /**
     * 添加路由
     * @param pattern 路由uri
     * @param klass 路由class
     * @return {@code RouterInternal} Router真实调用类
     */
    public final <T> RouterInternal router(String pattern, Class<T> klass) {
        Rule<T, ?> rule = getRule(pattern);
        if (rule == null) {
            throw new NotRouteException("unknown", pattern);
        }

        rule.router(pattern, klass);
        return this;
    }

    /**
     * 路由调用
     * @param ctx Context
     * @param pattern 路由uri
     * @return {@code V} 返回对应的返回值
     */
    /*package*/ final <V> V invoke(Context ctx, String pattern) {
        Rule<?, V> rule = getRule(pattern);
        if (rule == null) {
            throw new NotRouteException("unknown", pattern);
        }

        return rule.invoke(ctx, pattern);
    }

    /**
     * 是否存在该路由
     * @param pattern
     * @return
     */
    final boolean resolveRouter(String pattern) {
        Rule<?, ?> rule = getRule(pattern);
        return rule != null && rule.resolveRule(pattern);
    }
}
```
这里 *RouterInternal* 对象通过一个两层map结构来完成路由功能,第一层就是pattern--->rule对象,第二次是rule对象内部的pattern----->class对象.这样有什么好处呢?可以看到通过 *initDefaultRouter* 预置了3个默认的rule对象在 *RouterInternal* 内部的map中,分别对应activity,service和广播.然后每次先通过pattern来找到这3个内置rule中的一个,然后才把pattern和class对象放入rule对象中.
这样的好处是便于管理,所有的activity都放入一个rule对象的,所有的service都放入同一个rule中,结构非常的清晰,同时也节约了内存.

```java
public interface Rule<T, V> {
    /**
     * 添加路由
     * @param pattern 路由uri
     * @param klass 路由class
     */
    void router(String pattern, Class<T> klass);

    /**
     * 路由调用
     * @param ctx Context
     * @param pattern 路由uri
     * @return {@code V} 返回对应的返回值
     */
    V invoke(Context ctx, String pattern);

    /**
     * 查看是否存在pattern对应的路由
     * @param pattern
     * @return
     */
    boolean resolveRule(String pattern);
}
```
rule是一个map结构的对象,用来URI匹配的细分,具体实现在 *BaseInternRule* 中
```java
public abstract class BaseIntentRule<T> implements Rule<T, Intent> {

    private HashMap<String, Class<T>> mIntentRules;

    public BaseIntentRule() {
        mIntentRules = new HashMap<>();
    }

    /**
     * {@inheritDoc}
     */
    @Override
    public void router(String pattern, Class<T> klass) {
        mIntentRules.put(pattern, klass);
    }

    /**
     * {@inheritDoc}
     */
    @Override
    public Intent invoke(Context ctx, String pattern) {
        Class<T> klass = mIntentRules.get(pattern);
        if (klass == null) { throwException(pattern);}
        return new Intent(ctx, klass);
    }

    @Override
    public boolean resolveRule(String pattern) {
        return mIntentRules.get(pattern) != null;
    }

    /**
     * 当找不到路由规则时抛出异常
     * @param pattern 路由pattern
     */
    public abstract void throwException(String pattern);

}
```
可以看到这里就是一个简单的map结构
剩下的 *ActivityRule* ,*ReceiverRule* ,*ServiceRule* 都是继承自 *BaseIntentRule* 中,内部定义了各自的前缀.

到这里router的核心逻辑就完成了,下面来看怎么用的

### app工程的router调用
```java

@Components({"shop", "bbs"})
public class App extends MultiDexApplication {

    @Override
    public void onCreate() {
        super.onCreate();
        setupRouter();
        Logger.dump("TAG", "application: " + getClass().getName());
    }

    private void setupRouter() {
        RouterHelper.install();

//        Router.router(ActivityRule.ACTIVITY_SCHEME + "shop.main", ShopActivity.class);
//        Router.router(ActivityRule.ACTIVITY_SCHEME + "bbs.main", BBSActivity.class);
    }
}
```
在工程的总入口application对象中完成了模块的注册,同是调用了 *RouterHelper.install();*
```java
public class RouterHelper {

    public static void install() {
        try {
            Class<?> klass = Class.forName(Config.PACKAGE_NAME + "." + Config.ROUTER_MANAGER);
            Method method = klass.getDeclaredMethod(Config.ROUTER_MANAGER_METHOD);
            method.invoke(null);
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        } catch (NoSuchMethodException e) {
            e.printStackTrace();
        } catch (InvocationTargetException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        }

    }
}
```
这里就是通过反射来找到上面那个 *processor* 生成的java源文件,调用里面的唯一方法.完成router的注册的.

再看两个子模块是怎么互相调用对方的


### shop工程
```java
@Component("shop")
public class Shop {
}
```
```java
@StaticRouter(ActivityRule.ACTIVITY_SCHEME + "shop.main")
public class ShopActivity extends AppCompatActivity {

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        TextView tv = new TextView(this);
        tv.setTextSize(50);
        tv.setText("SHOP!!!");
        setContentView(tv);

        Logger.dump("TAG", "Hei! I am shop!!!");
        UseContext.use(Application.get());

        tv.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                Toast.makeText(ShopActivity.this, getResources().getString(R.string.click_notice), Toast.LENGTH_SHORT).show();
                if (Router.resolveRouter(ActivityRule.ACTIVITY_SCHEME + "org.loader.bbslib.BBSActivity")) {
                    Intent it = org.loader.router.Router.invoke(ShopActivity.this, ActivityRule.ACTIVITY_SCHEME + "org.loader.bbslib.BBSActivity");
                    startActivity(it);
                }
            }
        });
    }
}
```
可以看到这里通过Shop对象完成模块的注册,最后在textView的跳转是调用 *Router的invoke* 方法来找到bbs的制定页面.
这里的注册用是 *StaticRouter* 所以要写完整的URI路径.
### bbs工程
```java
@Component("bbs")
public class BBS {
}
```
```java

@AutoRouter
public class BBSActivity extends AppCompatActivity {

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        TextView tv = new TextView(this);
        tv.setTextSize(50);
        tv.setText("BBS!!!");
        setContentView(tv);

        Logger.dump("TAG", "Hei! I am bbs!!!");
        UseContext.use(Application.get());

        tv.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                Toast.makeText(BBSActivity.this, getResources().getString(R.string.click_notice), Toast.LENGTH_SHORT).show();
                if (Router.resolveRouter(ActivityRule.ACTIVITY_SCHEME + "shop.main")) {
                    Intent it = Router.invoke(BBSActivity.this, ActivityRule.ACTIVITY_SCHEME + "shop.main");
                    startActivity(it);
                }
            }
        });
    }
}
```
这里的注册用的 *AutoRouter* 所以不需要写URI,自动生成的URI是 类型+类的全路径

### 总结
可以看到 Router结构实际上就是通过一个URI找到对方的制定类.结构非常简单.之所以用了多点的时间主要是用来消化注解的相关处理逻辑,用着方便,但是要理解还是要话点时间的.
不过这种通过注解来生成中间类,最终达到解耦目的的思路非常值得尝试.开阔了视野.

### 参考资料
[Module2Module][93fa509a]

  [93fa509a]: https://github.com/qibin0506/Module2Module "Module2Module"

[Android路由实现][237ce709]

  [237ce709]: http://blog.csdn.net/qibin0506/article/details/53373412 "Android路由实现"
