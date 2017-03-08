layout: "post"
title: "android ButterKnife源码解析"
date: "2017-01-16 08:56"
---
## android ButterKnife源码解析

### ButterKnife基本原理

通过初次查看代码结构,发现原理大致和EventBus相同,通过定义不同的注解,然后通过注解分析器来生成新的源文件代码,在里面完成view的初始化和事件绑定,最后通过 *ButterKnife.bind* 方法来反射调用生成的新的源文件的中的初始化方法,来完成view的初始化和时间绑定.

可以这样理解,实际上以前是把findViewById等语句放在一个方法里面了,而这个方法在onCreate里面调用.现在是onCreate里面调用了 *ButterKnif.bind* 方法,里面还是findViewById那一套,只不过是通过注解分析器提前生成了源代码,里面是findViewById那些相关的声明语句.

注意,通过注解预编译这种方式可以达到在编译阶段来完成view的初始化和事件绑定,相对于反射是基本上没有性能消耗,所以是值得提倡的一种的技术方式.包括之前研究的Router也是通过这种方式来达到模块之间的相互调用的功能的.

### ButterKnife源代码分析

#### ButterKnife的Bind方法调用
先从ButterKnife的入口来分析,比如在Activity中我们调用 *ButterKnife.bind(this)* 方法,来看其内部实现:
```java
@NonNull @UiThread
 public static Unbinder bind(@NonNull Activity target) {
   View sourceView = target.getWindow().getDecorView();
   return createBinding(target, sourceView);
 }
```
这里获取到的是顶层DecorView,target是activity对象本身,然后交给 *createBinding* 方法.
```java
private static Unbinder createBinding(@NonNull Object target, @NonNull View source) {
   Class<?> targetClass = target.getClass();
   if (debug) Log.d(TAG, "Looking up binding for " + targetClass.getName());
   Constructor<? extends Unbinder> constructor = findBindingConstructorForClass(targetClass);

   if (constructor == null) {
     return Unbinder.EMPTY;
   }

   //noinspection TryWithIdenticalCatches Resolves to API 19+ only type.
   try {
     return constructor.newInstance(target, source);
   } catch (IllegalAccessException e) {
     throw new RuntimeException("Unable to invoke " + constructor, e);
   } catch (InstantiationException e) {
     throw new RuntimeException("Unable to invoke " + constructor, e);
   } catch (InvocationTargetException e) {
     Throwable cause = e.getCause();
     if (cause instanceof RuntimeException) {
       throw (RuntimeException) cause;
     }
     if (cause instanceof Error) {
       throw (Error) cause;
     }
     throw new RuntimeException("Unable to create binding instance.", cause);
   }
 }
```
这里可以看到 根据类名称来通过 *findBindingConstructorForClass* 查找该类具有两个参数的构造函数.
```java
@Nullable @CheckResult @UiThread
  private static Constructor<? extends Unbinder> findBindingConstructorForClass(Class<?> cls) {
    Constructor<? extends Unbinder> bindingCtor = BINDINGS.get(cls);
    if (bindingCtor != null) {
      if (debug) Log.d(TAG, "HIT: Cached in binding map.");
      return bindingCtor;
    }
    String clsName = cls.getName();
    if (clsName.startsWith("android.") || clsName.startsWith("java.")) {
      if (debug) Log.d(TAG, "MISS: Reached framework class. Abandoning search.");
      return null;
    }
    try {
      Class<?> bindingClass = Class.forName(clsName + "_ViewBinding");
      //noinspection unchecked
      bindingCtor = (Constructor<? extends Unbinder>) bindingClass.getConstructor(cls, View.class);
      if (debug) Log.d(TAG, "HIT: Loaded binding class and constructor.");
    } catch (ClassNotFoundException e) {
      if (debug) Log.d(TAG, "Not found. Trying superclass " + cls.getSuperclass().getName());
      bindingCtor = findBindingConstructorForClass(cls.getSuperclass());
    } catch (NoSuchMethodException e) {
      throw new RuntimeException("Unable to find binding constructor for " + clsName, e);
    }
    BINDINGS.put(cls, bindingCtor);
    return bindingCtor;
  }
```
```java
@VisibleForTesting
 static final Map<Class<?>, Constructor<? extends Unbinder>> BINDINGS = new LinkedHashMap<>();
```
这里 BINDINGS 是一个LinkedHashMap,存放的是类对象和它的构造函数对应关系.相当于一个缓存的作用,注意这里找的class对象是获取activity的类名称再加上 *_ViewBinding* 后缀.当使用ButterKnife编译完成之后,我们可以在我们工程中的build目录中找到 这些中间类.比如 你的activity名称叫做 "MainActivity",那么生成的
中间类就叫做 "MainActivity_ViewBinding".可以查看下里面的代码.就像这样:

```java
public class SimpleActivity_ViewBinding<T extends SimpleActivity> implements Unbinder {
  protected T target;

  private View view2130968578;

  private View view2130968579;

  @UiThread
  public SimpleActivity_ViewBinding(final T target, View source) {
    this.target = target;

    View view;
    target.title = Utils.findRequiredViewAsType(source, R.id.title, "field 'title'", TextView.class);
    target.subtitle = Utils.findRequiredViewAsType(source, R.id.subtitle, "field 'subtitle'", TextView.class);
    view = Utils.findRequiredView(source, R.id.hello, "field 'hello', method 'sayHello', and method 'sayGetOffMe'");
    target.hello = Utils.castView(view, R.id.hello, "field 'hello'", Button.class);
    view2130968578 = view;
    view.setOnClickListener(new DebouncingOnClickListener() {
      @Override
      public void doClick(View p0) {
        target.sayHello();
      }
    });
    view.setOnLongClickListener(new View.OnLongClickListener() {
      @Override
      public boolean onLongClick(View p0) {
        return target.sayGetOffMe();
      }
    });
    view = Utils.findRequiredView(source, R.id.list_of_things, "field 'listOfThings' and method 'onItemClick'");
    target.listOfThings = Utils.castView(view, R.id.list_of_things, "field 'listOfThings'", ListView.class);
    view2130968579 = view;
    ((AdapterView<?>) view).setOnItemClickListener(new AdapterView.OnItemClickListener() {
      @Override
      public void onItemClick(AdapterView<?> p0, View p1, int p2, long p3) {
        target.onItemClick(p2);
      }
    });
    target.footer = Utils.findRequiredViewAsType(source, R.id.footer, "field 'footer'", TextView.class);
    target.headerViews = Utils.listOf(
        Utils.findRequiredView(source, R.id.title, "field 'headerViews'"),
        Utils.findRequiredView(source, R.id.subtitle, "field 'headerViews'"),
        Utils.findRequiredView(source, R.id.hello, "field 'headerViews'"));
  }

  @Override
  @CallSuper
  public void unbind() {
    T target = this.target;
    if (target == null) throw new IllegalStateException("Bindings already cleared.");

    target.title = null;
    target.subtitle = null;
    target.hello = null;
    target.listOfThings = null;
    target.footer = null;
    target.headerViews = null;

    view2130968578.setOnClickListener(null);
    view2130968578.setOnLongClickListener(null);
    view2130968578 = null;
    ((AdapterView<?>) view2130968579).setOnItemClickListener(null);
    view2130968579 = null;

    this.target = null;
  }
}

```
这个java类是通过注解解析器完成通过代码来生成的.可以看到里面就是一些view的初始化和事件绑定.唯一要注意的一点是,这里的中间类都是实现了unBinder接口,实现了unBind方法,里面是对view和各种事件的回收.主要是用在 *fragment* 中的使用的,看下ButterKnife的说明会看到,在 fragment使用完成之后必须调用 *ViewBinder.unBinder* 方法来完成view的回收.
下面的代码来着ButterKnife的说明:
```java
BINDING RESET

Fragments have a different view lifecycle than activities. When binding a fragment in onCreateView, set the views to null in onDestroyView. Butter Knife returns an Unbinder instance when you call bind to do this for you. Call its unbind method in the appropriate lifecycle callback.

public class FancyFragment extends Fragment {
  @BindView(R.id.button1) Button button1;
  @BindView(R.id.button2) Button button2;
  private Unbinder unbinder;

  @Override public View onCreateView(LayoutInflater inflater, ViewGroup container, Bundle savedInstanceState) {
    View view = inflater.inflate(R.layout.fancy_fragment, container, false);
    unbinder = ButterKnife.bind(this, view);
    // TODO Use fields...
    return view;
  }

  @Override public void onDestroyView() {
    super.onDestroyView();
    unbinder.unbind();
  }
}

```
#### 注解分析器AnnotationProcessor的逻辑分析

核心点就在 *ButterKnifeProcessor* 里面.
```java
@Override public Set<String> getSupportedAnnotationTypes() {
    Set<String> types = new LinkedHashSet<>();
    for (Class<? extends Annotation> annotation : getSupportedAnnotations()) {
      types.add(annotation.getCanonicalName());
    }
    return types;
  }

  private Set<Class<? extends Annotation>> getSupportedAnnotations() {
    Set<Class<? extends Annotation>> annotations = new LinkedHashSet<>();

    annotations.add(BindArray.class);
    annotations.add(BindBitmap.class);
    annotations.add(BindBool.class);
    annotations.add(BindColor.class);
    annotations.add(BindDimen.class);
    annotations.add(BindDrawable.class);
    annotations.add(BindFloat.class);
    annotations.add(BindInt.class);
    annotations.add(BindString.class);
    annotations.add(BindView.class);
    annotations.add(BindViews.class);
    annotations.addAll(LISTENERS);

    return annotations;
  }
  private static final List<Class<? extends Annotation>> LISTENERS = Arrays.asList(//
       OnCheckedChanged.class, //
       OnClick.class, //
       OnEditorAction.class, //
       OnFocusChange.class, //
       OnItemClick.class, //
       OnItemLongClick.class, //
       OnItemSelected.class, //
       OnLongClick.class, //
       OnPageChange.class, //
       OnTextChanged.class, //
       OnTouch.class //
   );
```
可以通过 *getSupportedAnnotationTypes* 方法看到都支持哪些注解的处理.
重点的处理方法就在 *process* 方法中:
```java
@Override public boolean process(Set<? extends TypeElement> elements, RoundEnvironment env) {
   Map<TypeElement, BindingSet> bindingMap = findAndParseTargets(env);

   for (Map.Entry<TypeElement, BindingSet> entry : bindingMap.entrySet()) {
     TypeElement typeElement = entry.getKey();
     BindingSet binding = entry.getValue();

     JavaFile javaFile = binding.brewJava(sdk);
     try {
       javaFile.writeTo(filer);
     } catch (IOException e) {
       error(typeElement, "Unable to write binding for type %s: %s", typeElement, e.getMessage());
     }
   }

   return false;
 }
```
可以看到第一行就是通过遍历扫描来找到对应类的所有 *ButterKnife* 注解,然后一次性来生成每个类的新的源文件类.

下面来看 *findAndParseTargets* 方法.

```java
private Map<TypeElement, BindingSet> findAndParseTargets(RoundEnvironment env) {
    Map<TypeElement, BindingSet.Builder> builderMap = new LinkedHashMap<>();
    Set<TypeElement> erasedTargetNames = new LinkedHashSet<>();

    scanForRClasses(env);

    // Process each @BindArray element.
    for (Element element : env.getElementsAnnotatedWith(BindArray.class)) {
      if (!SuperficialValidation.validateElement(element)) continue;
      try {
        parseResourceArray(element, builderMap, erasedTargetNames);
      } catch (Exception e) {
        logParsingError(element, BindArray.class, e);
      }
    }

    // Process each @BindBitmap element.
    for (Element element : env.getElementsAnnotatedWith(BindBitmap.class)) {
      if (!SuperficialValidation.validateElement(element)) continue;
      try {
        parseResourceBitmap(element, builderMap, erasedTargetNames);
      } catch (Exception e) {
        logParsingError(element, BindBitmap.class, e);
      }
    }

    // Process each @BindBool element.
    for (Element element : env.getElementsAnnotatedWith(BindBool.class)) {
      if (!SuperficialValidation.validateElement(element)) continue;
      try {
        parseResourceBool(element, builderMap, erasedTargetNames);
      } catch (Exception e) {
        logParsingError(element, BindBool.class, e);
      }
    }

    // Process each @BindColor element.
    for (Element element : env.getElementsAnnotatedWith(BindColor.class)) {
      if (!SuperficialValidation.validateElement(element)) continue;
      try {
        parseResourceColor(element, builderMap, erasedTargetNames);
      } catch (Exception e) {
        logParsingError(element, BindColor.class, e);
      }
    }

    // Process each @BindDimen element.
    for (Element element : env.getElementsAnnotatedWith(BindDimen.class)) {
      if (!SuperficialValidation.validateElement(element)) continue;
      try {
        parseResourceDimen(element, builderMap, erasedTargetNames);
      } catch (Exception e) {
        logParsingError(element, BindDimen.class, e);
      }
    }

    // Process each @BindDrawable element.
    for (Element element : env.getElementsAnnotatedWith(BindDrawable.class)) {
      if (!SuperficialValidation.validateElement(element)) continue;
      try {
        parseResourceDrawable(element, builderMap, erasedTargetNames);
      } catch (Exception e) {
        logParsingError(element, BindDrawable.class, e);
      }
    }

    // Process each @BindFloat element.
    for (Element element : env.getElementsAnnotatedWith(BindFloat.class)) {
      if (!SuperficialValidation.validateElement(element)) continue;
      try {
        parseResourceFloat(element, builderMap, erasedTargetNames);
      } catch (Exception e) {
        logParsingError(element, BindFloat.class, e);
      }
    }

    // Process each @BindInt element.
    for (Element element : env.getElementsAnnotatedWith(BindInt.class)) {
      if (!SuperficialValidation.validateElement(element)) continue;
      try {
        parseResourceInt(element, builderMap, erasedTargetNames);
      } catch (Exception e) {
        logParsingError(element, BindInt.class, e);
      }
    }

    // Process each @BindString element.
    for (Element element : env.getElementsAnnotatedWith(BindString.class)) {
      if (!SuperficialValidation.validateElement(element)) continue;
      try {
        parseResourceString(element, builderMap, erasedTargetNames);
      } catch (Exception e) {
        logParsingError(element, BindString.class, e);
      }
    }

    // Process each @BindView element.
    for (Element element : env.getElementsAnnotatedWith(BindView.class)) {
      // we don't SuperficialValidation.validateElement(element)
      // so that an unresolved View type can be generated by later processing rounds
      try {
        parseBindView(element, builderMap, erasedTargetNames);
      } catch (Exception e) {
        logParsingError(element, BindView.class, e);
      }
    }

    // Process each @BindViews element.
    for (Element element : env.getElementsAnnotatedWith(BindViews.class)) {
      // we don't SuperficialValidation.validateElement(element)
      // so that an unresolved View type can be generated by later processing rounds
      try {
        parseBindViews(element, builderMap, erasedTargetNames);
      } catch (Exception e) {
        logParsingError(element, BindViews.class, e);
      }
    }

    // Process each annotation that corresponds to a listener.
    for (Class<? extends Annotation> listener : LISTENERS) {
      findAndParseListener(env, listener, builderMap, erasedTargetNames);
    }

    // Associate superclass binders with their subclass binders. This is a queue-based tree walk
    // which starts at the roots (superclasses) and walks to the leafs (subclasses).
    Deque<Map.Entry<TypeElement, BindingSet.Builder>> entries =
        new ArrayDeque<>(builderMap.entrySet());
    Map<TypeElement, BindingSet> bindingMap = new LinkedHashMap<>();
    while (!entries.isEmpty()) {
      Map.Entry<TypeElement, BindingSet.Builder> entry = entries.removeFirst();

      TypeElement type = entry.getKey();
      BindingSet.Builder builder = entry.getValue();

      TypeElement parentType = findParentType(type, erasedTargetNames);
      if (parentType == null) {
        bindingMap.put(type, builder.build());
      } else {
        BindingSet parentBinding = bindingMap.get(parentType);
        if (parentBinding != null) {
          builder.setParent(parentBinding);
          bindingMap.put(type, builder.build());
        } else {
          // Has a superclass binding but we haven't built it yet. Re-enqueue for later.
          entries.addLast(entry);
        }
      }
    }

    return bindingMap;
  }
```
这个 *findAndParseTargets* 方法看着很长,实际上分成了3部分,顶部是用来生成id和view的对应关系,中间是各种注解的处理后放入builderMap中,第三部分是遍历builderMap来生成BindingMap的.
先看如何生成ID的对应关系:
```java
  scanForRClasses(env);
```
```java
private void scanForRClasses(RoundEnvironment env) {
   if (trees == null) return;

   RClassScanner scanner = new RClassScanner();

   for (Class<? extends Annotation> annotation : getSupportedAnnotations()) {
     for (Element element : env.getElementsAnnotatedWith(annotation)) {
       JCTree tree = (JCTree) trees.getTree(element, getMirror(element, annotation));
       if (tree != null) { // tree can be null if the references are compiled types and not source
         tree.accept(scanner);
       }
     }
   }

   for (String rClass : scanner.getRClasses()) {
     parseRClass(rClass);
   }
 }

 private void parseRClass(String rClass) {
   Element element;

   try {
     element = elementUtils.getTypeElement(rClass);
   } catch (MirroredTypeException mte) {
     element = typeUtils.asElement(mte.getTypeMirror());
   }

   JCTree tree = (JCTree) trees.getTree(element);
   if (tree != null) { // tree can be null if the references are compiled types and not source
     IdScanner idScanner =
         new IdScanner(symbols, elementUtils.getPackageOf(element).getQualifiedName().toString());
     tree.accept(idScanner);
   } else {
     parseCompiledR((TypeElement) element);
   }
 }

 private void parseCompiledR(TypeElement rClass) {
   for (Element element : rClass.getEnclosedElements()) {
     String innerClassName = element.getSimpleName().toString();
     if (SUPPORTED_TYPES.contains(innerClassName)) {
       for (Element enclosedElement : element.getEnclosedElements()) {
         if (enclosedElement instanceof VariableElement) {
           VariableElement variableElement = (VariableElement) enclosedElement;
           Object value = variableElement.getConstantValue();

           if (value instanceof Integer) {
             int id = (Integer) value;
             ClassName rClassName =
                 ClassName.get(elementUtils.getPackageOf(variableElement).toString(), "R",
                     innerClassName);
             String resourceName = variableElement.getSimpleName().toString();
             symbols.put(id, new Id(id, rClassName, resourceName));
           }
         }
       }
     }
   }
 }

 private static class RClassScanner extends TreeScanner {
   private final Set<String> rClasses = new LinkedHashSet<>();

   @Override public void visitSelect(JCTree.JCFieldAccess jcFieldAccess) {
     Symbol symbol = jcFieldAccess.sym;
     if (symbol != null
         && symbol.getEnclosingElement() != null
         && symbol.getEnclosingElement().getEnclosingElement() != null
         && symbol.getEnclosingElement().getEnclosingElement().enclClass() != null) {
       rClasses.add(symbol.getEnclosingElement().getEnclosingElement().enclClass().className());
     }
   }

   Set<String> getRClasses() {
     return rClasses;
   }
 }

 private static class IdScanner extends TreeScanner {
   private final Map<Integer, Id> ids;
   private final String packageName;

   IdScanner(Map<Integer, Id> ids, String packageName) {
     this.ids = ids;
     this.packageName = packageName;
   }

   @Override public void visitClassDef(JCTree.JCClassDecl jcClassDecl) {
     for (JCTree tree : jcClassDecl.defs) {
       if (tree instanceof ClassTree) {
         ClassTree classTree = (ClassTree) tree;
         String className = classTree.getSimpleName().toString();
         if (SUPPORTED_TYPES.contains(className)) {
           ClassName rClassName = ClassName.get(packageName, "R", className);
           VarScanner scanner = new VarScanner(ids, rClassName);
           ((JCTree) classTree).accept(scanner);
         }
       }
     }
   }
 }

 private static class VarScanner extends TreeScanner {
   private final Map<Integer, Id> ids;
   private final ClassName className;

   private VarScanner(Map<Integer, Id> ids, ClassName className) {
     this.ids = ids;
     this.className = className;
   }

   @Override public void visitVarDef(JCTree.JCVariableDecl jcVariableDecl) {
     if ("int".equals(jcVariableDecl.getType().toString())) {
       int id = Integer.valueOf(jcVariableDecl.getInitializer().toString());
       String resourceName = jcVariableDecl.getName().toString();
       ids.put(id, new Id(id, className, resourceName));
     }
   }
 }
```
这里,如何关于如何生成这个对应关系的里面用到一些api我不是很了解,网上的资料也很少,后续等到研究明白了再补充吧,只是通过字面理解应该是通过遍历java的语法分析树找到R文件中对应的filed的ID,组合成一个 *Id* .
中间的部分就是用来做生成builderMap的,我们挑其中的一个方法来看,大部分的方法都是类似的.
```java
private void parseBindView(Element element, Map<TypeElement, BindingSet.Builder> builderMap,
                              Set<TypeElement> erasedTargetNames) {
       TypeElement enclosingElement = (TypeElement) element.getEnclosingElement();

       // Start by verifying common generated code restrictions.
       boolean hasError = isInaccessibleViaGeneratedCode(BindView.class, "fields", element)
               || isBindingInWrongPackage(BindView.class, element);

       // Verify that the target type extends from View.
       TypeMirror elementType = element.asType();
       if (elementType.getKind() == TypeKind.TYPEVAR) {
           TypeVariable typeVariable = (TypeVariable) elementType;
           elementType = typeVariable.getUpperBound();
       }
       if (!isSubtypeOfType(elementType, VIEW_TYPE) && !isInterface(elementType)) {
           if (elementType.getKind() == TypeKind.ERROR) {
               note(element, "@%s field with unresolved type (%s) "
                               + "must elsewhere be generated as a View or interface. (%s.%s)",
                       BindView.class.getSimpleName(), elementType, enclosingElement.getQualifiedName(),
                       element.getSimpleName());
           } else {
               error(element, "@%s fields must extend from View or be an interface. (%s.%s)",
                       BindView.class.getSimpleName(), enclosingElement.getQualifiedName(),
                       element.getSimpleName());
               hasError = true;
           }
       }

       if (hasError) {
           return;
       }

       // Assemble information on the field.
       int id = element.getAnnotation(BindView.class).value();

       BindingSet.Builder builder = builderMap.get(enclosingElement);
       if (builder != null) {
           String existingBindingName = builder.findExistingBindingName(getId(id));
           if (existingBindingName != null) {
               error(element, "Attempt to use @%s for an already bound ID %d on '%s'. (%s.%s)",
                       BindView.class.getSimpleName(), id, existingBindingName,
                       enclosingElement.getQualifiedName(), element.getSimpleName());
               return;
           }
       } else {
           builder = getOrCreateBindingBuilder(builderMap, enclosingElement);
       }

       String name = element.getSimpleName().toString();
       TypeName type = TypeName.get(elementType);
       boolean required = isFieldRequired(element);

       builder.addField(getId(id), new FieldViewBinding(name, type, required));

       // Add the type-erased version to the valid binding targets set.
       erasedTargetNames.add(enclosingElement);
   }
```
这个方法前半部分用来做ID校验,防止有重复的Id定义出现错误.
后半部分生成一个 *BindingSet.Builder* 对象,这个对象就是用来后续通过 *javapoet* 来生成制定的java源文件的.builderMap的key是声明注解的类,通常是我们的Activity或者viewHolder.
然后把 *FieldViewBinding* 放入builder中,其中 filed的key就是view本身的ID.

*findAndParseTargets* 的最后一部分是用来生成bindingMap的,key是声明类,value是BindingSet对象.
通过中间生成的 *builderMap* 来完成.主要是用来子类和父类如果同时声明的时把两者合并成一个BindingSet对象.

*BindingSet* 对象就是用来交给javaPoet来生成中间源文件的对象.

生成了BindingMap之后回到顶层的 *process* 方法:
```java
for (Map.Entry<TypeElement, BindingSet> entry : bindingMap.entrySet()) {
            TypeElement typeElement = entry.getKey();
            BindingSet binding = entry.getValue();

            JavaFile javaFile = binding.brewJava(sdk);
            try {
                javaFile.writeTo(filer);
            } catch (IOException e) {
                error(typeElement, "Unable to write binding for type %s: %s", typeElement, e.getMessage());
            }
        }
```
可以看到这里调用的是 *bindingSet* 对象的 *brewJava* 方法来完成java源文件的写入的:

```java

  JavaFile brewJava(int sdk) {
    return JavaFile.builder(bindingClassName.packageName(), createType(sdk))
        .addFileComment("Generated code from Butter Knife. Do not modify!")
        .build();
  }
```
然后调用 *createType* 方法:
```java
private TypeSpec createType(int sdk) {
    TypeSpec.Builder result = TypeSpec.classBuilder(bindingClassName.simpleName())
        .addModifiers(PUBLIC);
    if (isFinal) {
      result.addModifiers(FINAL);
    }

    if (parentBinding != null) {
      result.superclass(parentBinding.bindingClassName);
    } else {
      result.addSuperinterface(UNBINDER);
    }

    if (hasTargetField()) {
      result.addField(targetTypeName, "target", PRIVATE);
    }

    if (!constructorNeedsView()) {
      // Add a delegating constructor with a target type + view signature for reflective use.
      result.addMethod(createBindingViewDelegateConstructor(targetTypeName));
    }
    result.addMethod(createBindingConstructor(targetTypeName, sdk));

    if (hasViewBindings() || parentBinding == null) {
      result.addMethod(createBindingUnbindMethod(result, targetTypeName));
    }

    return result.build();
  }
```
*createType* 分为三部分,一部分是createBindingViewDelegateConstructor,一部分是createBindingConstructor,还有一部分是createBindingUnbindMethod
```java
private MethodSpec createBindingViewDelegateConstructor(TypeName targetType) {
  return MethodSpec.constructorBuilder()
      .addJavadoc("@deprecated Use {@link #$T($T, $T)} for direct creation.\n    "
              + "Only present for runtime invocation through {@code ButterKnife.bind()}.\n",
          bindingClassName, targetType, CONTEXT)
      .addAnnotation(Deprecated.class)
      .addAnnotation(UI_THREAD)
      .addModifiers(PUBLIC)
      .addParameter(targetType, "target")
      .addParameter(VIEW, "source")
      .addStatement(("this(target, source.getContext())"))
      .build();
}
```
这个构造函数的作用暂时没有弄明白,看说明好像是用来在运行的时候反射用的.
```java
private MethodSpec createBindingConstructor(TypeName targetType, int sdk) {
    MethodSpec.Builder constructor = MethodSpec.constructorBuilder()
        .addAnnotation(UI_THREAD)
        .addModifiers(PUBLIC);

    if (hasMethodBindings()) {
      constructor.addParameter(targetType, "target", FINAL);
    } else {
      constructor.addParameter(targetType, "target");
    }

    if (constructorNeedsView()) {
      constructor.addParameter(VIEW, "source");
    } else {
      constructor.addParameter(CONTEXT, "context");
    }

    if (hasUnqualifiedResourceBindings()) {
      // Aapt can change IDs out from underneath us, just suppress since all will work at runtime.
      constructor.addAnnotation(AnnotationSpec.builder(SuppressWarnings.class)
          .addMember("value", "$S", "ResourceType")
          .build());
    }

    if (hasOnTouchMethodBindings()) {
      constructor.addAnnotation(AnnotationSpec.builder(SUPPRESS_LINT)
          .addMember("value", "$S", "ClickableViewAccessibility")
          .build());
    }

    if (parentBinding != null) {
      if (parentBinding.constructorNeedsView()) {
        constructor.addStatement("super(target, source)");
      } else if (constructorNeedsView()) {
        constructor.addStatement("super(target, source.getContext())");
      } else {
        constructor.addStatement("super(target, context)");
      }
      constructor.addCode("\n");
    }
    if (hasTargetField()) {
      constructor.addStatement("this.target = target");
      constructor.addCode("\n");
    }

    if (hasViewBindings()) {
      if (hasViewLocal()) {
        // Local variable in which all views will be temporarily stored.
        constructor.addStatement("$T view", VIEW);
      }
      for (ViewBinding binding : viewBindings) {
        addViewBinding(constructor, binding);
      }
      for (FieldCollectionViewBinding binding : collectionBindings) {
        constructor.addStatement("$L", binding.render());
      }

      if (!resourceBindings.isEmpty()) {
        constructor.addCode("\n");
      }
    }

    if (!resourceBindings.isEmpty()) {
      if (constructorNeedsView()) {
        constructor.addStatement("$T context = source.getContext()", CONTEXT);
      }
      if (hasResourceBindingsNeedingResource(sdk)) {
        constructor.addStatement("$T res = context.getResources()", RESOURCES);
      }
      for (ResourceBinding binding : resourceBindings) {
        constructor.addStatement("$L", binding.render(sdk));
      }
    }

    return constructor.build();
  }
```
这个方法来生成中间类的构造函数,里面定义了view的findViewById和绑定事件,生成的结果在前面已经列出了,就是 *javaPoet* 的API调用,非常容易理解.

最后是生成filed和unbind方法的 *createBindingUnbindMethod*:
```java

  private MethodSpec createBindingUnbindMethod(TypeSpec.Builder bindingClass,
      TypeName targetType) {
    MethodSpec.Builder result = MethodSpec.methodBuilder("unbind")
        .addAnnotation(Override.class)
        .addModifiers(PUBLIC);
    if (!isFinal && parentBinding == null) {
      result.addAnnotation(CALL_SUPER);
    }

    if (hasTargetField()) {
      if (hasFieldBindings()) {
        result.addStatement("$T target = this.target", targetType);
      }
      result.addStatement("if (target == null) throw new $T($S)", IllegalStateException.class,
          "Bindings already cleared.");
      result.addStatement("$N = null", hasFieldBindings() ? "this.target" : "target");
      result.addCode("\n");
      for (ViewBinding binding : viewBindings) {
        if (binding.getFieldBinding() != null) {
          result.addStatement("target.$L = null", binding.getFieldBinding().getName());
        }
      }
      for (FieldCollectionViewBinding binding : collectionBindings) {
        result.addStatement("target.$L = null", binding.name);
      }
    }

    if (hasMethodBindings()) {
      result.addCode("\n");
      for (ViewBinding binding : viewBindings) {
        addFieldAndUnbindStatement(bindingClass, result, binding);
      }
    }

    if (parentBinding != null) {
      result.addCode("\n");
      result.addStatement("super.unbind()");
    }
    return result.build();
  }
```
这里unBind方法不做过多的解释,主要是来看
```java
for (ViewBinding binding : viewBindings) {
  addFieldAndUnbindStatement(bindingClass, result, binding);
}
```
```java

  private void addFieldAndUnbindStatement(TypeSpec.Builder result, MethodSpec.Builder unbindMethod,
      ViewBinding bindings) {
    // Only add fields to the binding if there are method bindings.
    Map<ListenerClass, Map<ListenerMethod, Set<MethodViewBinding>>> classMethodBindings =
        bindings.getMethodBindings();
    if (classMethodBindings.isEmpty()) {
      return;
    }

    String fieldName = bindings.isBoundToRoot() ? "viewSource" : "view" + bindings.getId().value;
    result.addField(VIEW, fieldName, PRIVATE);

    // We only need to emit the null check if there are zero required bindings.
    boolean needsNullChecked = bindings.getRequiredBindings().isEmpty();
    if (needsNullChecked) {
      unbindMethod.beginControlFlow("if ($N != null)", fieldName);
    }

    for (ListenerClass listenerClass : classMethodBindings.keySet()) {
      // We need to keep a reference to the listener
      // in case we need to unbind it via a remove method.
      boolean requiresRemoval = !"".equals(listenerClass.remover());
      String listenerField = "null";
      if (requiresRemoval) {
        TypeName listenerClassName = bestGuess(listenerClass.type());
        listenerField = fieldName + ((ClassName) listenerClassName).simpleName();
        result.addField(listenerClassName, listenerField, PRIVATE);
      }

      if (!VIEW_TYPE.equals(listenerClass.targetType())) {
        unbindMethod.addStatement("(($T) $N).$N($N)", bestGuess(listenerClass.targetType()),
            fieldName, removerOrSetter(listenerClass, requiresRemoval), listenerField);
      } else {
        unbindMethod.addStatement("$N.$N($N)", fieldName,
            removerOrSetter(listenerClass, requiresRemoval), listenerField);
      }

      if (requiresRemoval) {
        unbindMethod.addStatement("$N = null", listenerField);
      }
    }

    unbindMethod.addStatement("$N = null", fieldName);

    if (needsNullChecked) {
      unbindMethod.endControlFlow();
    }
  }
```
这个方法用来生成中间类的filed,也就是view和事件的声明.之前在前面找了半天如何生成filed的方法一直没有找到,后来这这里才发现的.
声明的view名称是View+ID这种格式.

到这里大概的流程就讲完了,可能还是有些细节没有讲到,不过没有关系,只要明白了大概的运行方式就可以了.可以看到作者的思路非常的新颖就是通过中间类来完成一些初始化工作,这个给我们后续的android的编程中提供了一套可借鉴的思路.

### 参考资料
[butterKnife document][5834d1fc]

  [5834d1fc]: http://jakewharton.github.io/butterknife/ "butterKnife document"
