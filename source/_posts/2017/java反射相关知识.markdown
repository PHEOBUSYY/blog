layout: "post"
title: "java反射相关知识"
date: "2017-01-19 15:16"
---

## java反射相关知识

### Class.getDeclaringClass 和 Class.getEnclosingClass的区别

getDeclaringClass表示这个内的声明类是那个,常用于获取在当前中声明的内部类
getEnclosingClass表示获取当前类的外部调用类,也可以用于在内部类中获取外部类.
二者的区别在于匿名内部类,如果这个类不是在当前类中声明,那么getDeclaringClass返回null,而getEnclosingClass返回外层类
```java
class TestClassReflection{

  public static void main(String[] args) throws NoSuchFieldException {
         new Thread(new Runnable() {
             @Override
             public void run() {
                 Class<? extends Runnable> aClass = this.getClass();
                 Class<?> declaringClass2 = aClass.getDeclaringClass();
                 Class<?> enclosingClass1 = aClass.getEnclosingClass();
                 System.out.println("declaringClass2 = "+declaringClass2);
                 System.out.println("enclosingClass1 = "+enclosingClass1);
             }
         }).start();
     }
}
```
```
declaringClass2 = null
enclosingClass1 = class com.justyan.reflection.TestClassReflection
```

### 打印出继承关系
```java
public static void printAncestor(Class aclass, ArrayList<Class> list) {
       if (aclass != null) {
           Class superclass = aclass.getSuperclass();
           if (superclass != null) {
               list.add(superclass);
               printAncestor(superclass, list);
           }
       }
   }
```
最后把所有的父类都在list中.

### Class.getGenericInterfaces方法和Class.getInterfaces方法的区别

getGenericInterfaces返回所有的实现接口,包含接口的泛型
getInterfaces返回所有的接口class对象,不包含泛型信息

### Class.getTypeParameters 方法
getTypeParameters返回类的泛型信息

### Class.getDeclaredMethods 和 Class.getMethods 区别
这类前面加了 *Declared* 的方法,只是获取当前类的成员,不包含父类的相关成员
而getMethods会获取所有的方法

### Class.getFields和Class.getDeclaredFields
getFields获取当前类所有的 *public* 属性
getDeclaredFields返回当前类所有的属性
下面的话摘自官方教程

>Tip: The Class.getField() and Class.getFields() methods return the public member field(s) of the class, enum, or interface represented by >the Class object. To retrieve all fields declared (but not inherited) in the Class, use the Class.getDeclaredFields() method.


### 反射中获取method参数的相关属性
method.getParameters用来获取该方法的所有参数.返回一个 *Parameter[]* 数组.
```java

    private static void printParam(Method declaredMethod) {
        Parameter[] parameters = declaredMethod.getParameters();
        for (Parameter parameter : parameters) {
            System.out.println(parameter.getType());
            System.out.println(parameter.getName());
            System.out.println(parameter.getModifiers());
            System.out.println(parameter.isImplicit());
            System.out.println(parameter.isNamePresent());
            System.out.println(parameter.isSynthetic());
        }
        System.out.println("-------------------------");
    }
```
分别来解释下这几个方法的用法:
1. *getType* 获取参数类型,比如int,java.lang.String等
2. *getName* 获取参数名称,这里有个问题要注意,默认情况下获取到的参数都是 *arg0* ,*arg1* 等这样的系统生成的参数名称,是获取不到原始的命名的,这样做的目的是为了节约class文件的空间.如果想要获取到原始参数名称,需要在编译的 *javac* 后面加上 *-parameters* 参数.
3. *getModifiers* 是一个int值,是把下面要3种情况相加得出的.
![modifiers属性说明](/images/2017/01/method的modifiers属性说明.png)

4. *isImplicit* 和 *isSynthetic* 下面重点来讲一下
5. *isNamePresent* 对应上面的 *getName* 的情况,如果是从显示的原始的class中参数命名就是true,如果是编译器生成的 *arg0* 这种方式就是 false.

####  *isImplicit* 和 *isSynthetic*
先来看 *isImplicit* 通过字面理解就是 *含蓄的,隐式的* ,也就是说这个方法并没有在代码中直接声明,确实是存在的.对应的正常情景就是内部类对外部类的引用.我们都知道 *非静态* 内部类是对外部类有一个引用的.那么编译器在编译的时候生成的内部类的构造方法中是会有一个外部类的对象引用的.就想下面这样:
```java
public class MethodParameterExamples {
    public class InnerClass { }
}
```
```java
public class MethodParameterExamples {
    public class InnerClass {
        final MethodParameterExamples parent;
        InnerClass(final MethodParameterExamples this$0) {
            parent = this$0;
        }
    }
}
```
innerClass的默认构造函数其实是对外部的MethodParameterExamples有一个引用的,那么innerClass这个默认的构造方法的 *isImplicit* 属性就是true.

再来看,如果一个方法既不是显式的也不是隐式的而是通过编译器生成的,那么这个 *isSynthetic* 属性就为true.经典的用在枚举类型中
```java
public class MethodParameterExamples {
    enum Colors {
        RED, WHITE;
    }
}
```
如果答应这里Colors类对象的话,会发现其内部是这样的:
```java
final class Colors extends java.lang.Enum<Colors> {
    public final static Colors RED = new Colors("RED", 0);
    public final static Colors BLUE = new Colors("WHITE", 1);

    private final static values = new Colors[]{ RED, BLUE };

    private Colors(String name, int ordinal) {
        super(name, ordinal);
    }

    public static Colors[] values(){
        return values;
    }

    public static Colors valueOf(String name){
        return (Colors)java.lang.Enum.valueOf(Colors.class, name);
    }
}
```
这里面的构造函数 *Colors(String name , ind ordinal)* 是隐式声明的,其中的参数是编译器给生成的,那么参数的 *isSynthetic* 属性就是true.其中的 *valueOf* 方法中的参数是 *isImplicit* 的.
[implicit and synthetic](https://docs.oracle.com/javase/tutorial/reflect/member/methodparameterreflection.html)

###  Method.isVarArgs()
 Method.isVarArgs() 表明方法的参数是不是隐式数组这种格式,比如main方法中的 *String... args*

### Method的isSynthetic和isBridge
```java
public class TestMethodModifiers {
    public static void main(String[] args) {
        Method[] methods = Integer.class.getMethods();
        for (Method method : methods) {
            if (method.getName().equals("compareTo")) {
                System.out.println(method.toGenericString());
                System.out.println(method.isSynthetic());
                System.out.println(method.isBridge());
            }
        }
    }
}

```
```
public int java.lang.Integer.compareTo(java.lang.Integer)
false
false
public int java.lang.Integer.compareTo(java.lang.Object)
true
true
```
编译器会帮助生成一个 *compareTo(Object object)* 的中间方法,这个方法就是一个中间方法,用来解决 *泛型擦除?* 这里我也不是很明白.

### method反射调用注意事项
1. 如果要反射的方法需要传入的参数是一个数组,就像这样的:
  ```java
  void test(String... arr) {
         System.out.println("Iam test!!!!------" +arr);
     }
  ```
  如果你直接这样调用是不行的:
  ```java
  String[] mainArgs = new String[]{"a", "b"};
  method.invoke(TestMethodBean2.class.newInstance(), mainArgs);
  ```
  虽然你传入的确实是一个数组,但是invoke方法会认为你要调用的方法是参数个数为数组容量的方法,也就是说这里认为你想调用的是包含两个参数的test方法:
  ```java
  void test(String a,String b) {
         System.out.println("Iam test!!!!------" +arr);
     }
  ```
  如果要正确调用,应该把数组对象转换为Object对象,这样就可以了:
  ```java
  String[] mainArgs = new String[]{"a", "b"};
  method.invoke(TestMethodBean2.class.newInstance(), (Object) mainArgs);
  ```
2. 如果要反射一个带有泛型的方法,就像这样:
  ```java
  class TestMethodBean2<T>{
  void test(T t) {
      System.out.println("it is generic");
   }
  }
  ```
  通过代码:
  ```java
  new TestMethodBean2<Integer>().getClass().getDeclaredMethod("test",Integer.class);
  ```
  是不行的,会找不到这个方法,因为泛型在编译之后是会被擦除的,这个时候应该找的是最顶上的对象object才可以
  ```java
  new TestMethodBean2<Integer>().getClass().getDeclaredMethod("test",Object.class);
  ```
3. 对private方法的反射调用
  要在获取到方法对象之后设置 *method.setAccessible(true);* 就可以了

4. 如果要反射的方法没有参数,那么调用invoke方法的时候一定要注意
  ```java
  public class MethodTrouble {
    public static void main(String[] args) throws NoSuchMethodException, InvocationTargetException, IllegalAccessException {
        MethodTroubleBean bean = new MethodTroubleBean();
        Method test = bean.getClass().getDeclaredMethod("test");
        test.invoke(bean);  //success
        test.invoke(bean, null); //success ,但是有警告
        test.invoke(bean, new Object[0]); //success
        Object o = new Object[0];
        test.invoke(bean, o);  //wrong IllegalArgumentException
        Object n = null;
        test.invoke(bean, n);  //wrong IllegalArgumentException
    }
}

class MethodTroubleBean {
    void test() {
        System.out.println("test");
    }
  }
```  
一定要注意对参数的传入.

### Class.newInstance 和Construction.newInstance的区别
Class.newInstance只能调用默认的构造函数(没有参数的那个),而且必须是可见的
Construction.newInstance可以调用所有的构造函数,不管是否是可见的,也不管有多少个参数
推荐使用后面这个

### 反射创建数组
```java
Object o = Array.newInstance(int.class, 3);
        Array.set(o,0,1);
        Array.set(o,1,2);
        Array.set(o,2,3);
```

### 参考资料
[java reflect tutorial][5142240c]

  [5142240c]: https://docs.oracle.com/javase/tutorial/reflect/TOC.html "java reflect tutorial"
