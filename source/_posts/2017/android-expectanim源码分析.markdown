layout: "post"
title: "android ExpectAnim源码分析"
date: "2017-06-07 14:55"
---

  今天无意中发现了ExpectAnim这个android开源库，主要作用是对android属性动画的封装，希望通过这篇文章来介绍一下它的用法，并且通过源码来分析的一下实现原理。加深对android动画的理解和运用。

  <!--more-->

## 用法介绍

  通过该项目在github首页中的介绍可以看到它的一般用法如下：

  ```java
  new ExpectAnim()
                .expect(avatar)
                .toBe(
                    Expectation...
                )
                .toAnimation()
                .start();
  ```

  其中expect方法中传入是执行动画的view，toBe方法中放入的是 *Expectation* 集合，也就是动画动作的集合。这个后面会详细的讲解。
  如果是多个view同时执行动画，可以这样：
  ```java
  new ExpectAnim()
              .expect(avatar)
              .toBe(
                  Expectation...
              )
              .expect(name)
              .toBe(
                  Expectation...
              )
              .toAnimation()
              .start();
  ```

  再来看下android属性动画的用法：
  ```java
  ObjectAnimator animator = ObjectAnimator.ofFloat(textView,"translationX",100f);
  animator.setDuration(1000);
  animator.start();
  ```
  通过上面的示例代码对比可以发现该动画库的核心就是在 **toBe** 方法中的 **Expectation** 对象。最后通过代码把 **Expectation** 对象转换成android api。


### 核心类 ExpectAnim 和 ViewExpectation

  首先来看expect(View view)方法：
  ```java
  public ViewExpectation expect(View view) {
        this.anyView = view;
        final ViewExpectation viewExpectation = new ViewExpectation(this, view);
        expectationList.add(viewExpectation);
        return viewExpectation;
    }
  ```
  这里通过expect方法来生成了一个 **ViewExpectation** 对象。
  同时把这个对象放入 **expectationList** 集合中，等到最后调用对应方法来批量处理。
  注意：通过expect方法，成功的将 **ExpectAnim** 和 **ViewExpectation** 两个对象绑定到了一起。也就是说在 **ViewExpectation** 对象中是可以获取到 **ExpectAnim** 对象。

  然后来看 **ViewExpectation** 对象的 **toBe** 方法：
  ```java
  public ViewExpectation toBe(AnimExpectation... animExpectations) {
       this.animExpectations.addAll(Arrays.asList(animExpectations));
       return this;
   }
  ```
  这里是将动画操作放入它内部的 **animExpectations** 集合中。
  ```java
  public ExpectAnim toAnimation() {
        return expectAnim;
    }
  ```
  这里直接放回了绑定的 **ExpectAnim** 对象。最后来看 **ExpectAnim** 对象的 **start** 方法:
  ```java
  public ExpectAnim start() {
       executeAfterDraw(anyView, new Runnable() {
           @Override
           public void run() {
               calculate();
               animatorSet.start();
           }
       });
       return this;
   }
  ```
  ```java
  public void executeAfterDraw(final View view, final Runnable runnable) {
        view.postDelayed(runnable, 5);
    }
  ```
  通过上面的方法调用我们可以看到核心代码就是 **calculate** 方法了，就是通过它把我们的 **ViewExpectation** 转换成了android原生api的。

  下面来看 **calculate** 方法的具体实现：
  ```java
  private ExpectAnim calculate() {
        if (animatorSet == null) {
            animatorSet = new AnimatorSet();

            if (interpolator != null) {
                animatorSet.setInterpolator(interpolator);
            }

            animatorSet.setDuration(duration);

            final List<Animator> animatorList = new ArrayList<>();

            final List<ViewExpectation> expectationsToCalculate = new ArrayList<>();

            //"ViewDependencies" = récupérer toutes les vues des "Expectations"
            for (ViewExpectation viewExpectation : expectationList) {
                viewExpectation.calculateDependencies();
                viewToMove.add(viewExpectation.getViewToMove());
                expectationsToCalculate.add(viewExpectation);

                viewCalculator.setExpectationForView(viewExpectation.getViewToMove(), viewExpectation);
            }

            while (!expectationsToCalculate.isEmpty()) {
                //pour chaque expectation dans "Expectations"
                final Iterator<ViewExpectation> iterator = expectationsToCalculate.iterator();
                while (iterator.hasNext()) {
                    final ViewExpectation viewExpectation = iterator.next();

                    //regarder si une de ces dépendance est dans "ViewDependencies"
                    if (!hasDependency(viewExpectation)) {
                        //si non
                        viewExpectation.calculate(viewCalculator);
                        animatorList.addAll(viewExpectation.getAnimations());

                        final View view = viewExpectation.getViewToMove();
                        viewToMove.remove(view);
                        viewCalculator.wasCalculated(viewExpectation);

                        iterator.remove();
                    } else {
                        //si oui, attendre le prochain tour
                    }
                }
            }

            animatorSet.addListener(new AnimatorListenerAdapter() {

                @Override
                public void onAnimationEnd(Animator animation) {
                    super.onAnimationEnd(animation);
                    notifyListenerEnd();
                }

                @Override
                public void onAnimationStart(Animator animation) {
                    super.onAnimationStart(animation);
                    notifyListenerStart();
                }

            });

            animatorSet.playTogether(animatorList);
        }
        return this;
    }

  ```
  还记得上面调用 **expect** 方法把 **ViewExpectation** 对象放入一个 **expectationList** 集合中，这里先遍历这个集合：
  ```java
  for (ViewExpectation viewExpectation : expectationList) {
      viewExpectation.calculateDependencies();
      viewToMove.add(viewExpectation.getViewToMove());
      expectationsToCalculate.add(viewExpectation);

      viewCalculator.setExpectationForView(viewExpectation.getViewToMove(), viewExpectation);
  }
  ```
  先调用 **ViewExpectation** 对象的 **calculateDependencies** 方法：
  ```java
  List<View> calculateDependencies() {
      dependencies.clear();
      if (animExpectations != null) {
          for (AnimExpectation animExpectation : animExpectations) {
              dependencies.addAll(animExpectation.getViewsDependencies());
          }
      }
      return dependencies;
  }
  ```
  那在 **ViewExpectation** 中的 **dependencies** 集合到底有什么作用呢？
  ```java
   private final List<View> dependencies;
  ```
  可以看到在声明的时候其内部放入的是View对象，实际上ExpectAnim这个库中提供了很多根据其他View来调整自身view的属性的方法，比如sameWidthAs，sameScaleAs，sameAlphaAs等等，这些方法都是要依赖另一个view的属性来变换自身的属性的。如果依赖的这个view也是需要做动画的话，那么我们必须得在依赖的这个view之后做动画操作，这样才能保证动画执行的准确。这里的 **dependencies** 集合中放入的就是要依赖的view对象。当然大部分的变换是没有依赖对象的，只有像上面讲那些需要依赖其他view的时候这个集合中的对象才不为空。 通过 **calculateDependencies** 就完成了依赖对象集合的赋值。下面是把要变换的view和动作关联起来，这里通过 **viewCalculator** 中的map **expectationsToCalculate** 来完成。
  ```java
  private final Map<View, ViewExpectation> expectationForView;
  ```
