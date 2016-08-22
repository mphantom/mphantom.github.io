---
layout: post
title: ButterKnife解析
---
## ButterKnife解析
ButterKnife是什么想来就不用我多说废话了，android开发者必备神器，那么为什么要解析它呢纯粹是我在凑博客文章数量，又不想随便水一篇，就拿它开刀吧。。  

案例先给出butterKnife的使用方法（虽然我觉得有点多次一举，但是还是写出来吧）  

```groovy
dependencies {
  compile 'com.jakewharton:butterknife:8.2.1'
  annotationProcessor 'com.jakewharton:butterknife-compiler:8.2.1'
}
```  

```java
class ExampleActivity extends Activity {
  @BindView(R.id.title) TextView title;
  @BindView(R.id.subtitle) TextView subtitle;
  @BindView(R.id.footer) TextView footer;

  @Override public void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.simple_activity);
    ButterKnife.bind(this);
    // TODO Use fields...
  }
}
```  

可以看到我们平时使用的findViewById都被bindView这个注解取代了。  
那么这种实现方式的原理是什么呢。。听我慢慢道来。。  
在我们引入的annotationProcessor 'com.jakewharton:butterknife-compiler:8.2.1'这个其实起到的一个编译时注解的解析功能，而解析的目标就是我们取代了findViewById的bindView这个注解。而起作用的就是ButterKnifeProcessor这个类。  

```java
@AutoService(Processor.class)
public final class ButterKnifeProcessor extends AbstractProcessor {
……
	@Override public boolean process(Set<? extends TypeElement> elements, RoundEnvironment env) {
    Map<TypeElement, BindingClass> targetClassMap = findAndParseTargets(env);

    for (Map.Entry<TypeElement, BindingClass> entry : targetClassMap.entrySet()) {
      TypeElement typeElement = entry.getKey();
      BindingClass bindingClass = entry.getValue();

      for (JavaFile javaFile : bindingClass.brewJava()) {
        try {
          javaFile.writeTo(filer);
        } catch (IOException e) {
          error(typeElement, "Unable to write view binder for type %s: %s", typeElement,
              e.getMessage());
        }
      }
    }

    return true;
  }
……
}
```  

ButterKnifeProcessor集成了AbstractProcessor这个类（就是AbstractProcessor在编译中起到了作用）。实现了其中的process方法。。  

```java
@Override public boolean process(Set<? extends TypeElement> elements, RoundEnvironment env) {
    Map<TypeElement, BindingClass> targetClassMap = findAndParseTargets(env);

    for (Map.Entry<TypeElement, BindingClass> entry : targetClassMap.entrySet()) {
      TypeElement typeElement = entry.getKey();
      BindingClass bindingClass = entry.getValue();

      for (JavaFile javaFile : bindingClass.brewJava()) {
        try {
          javaFile.writeTo(filer);
        } catch (IOException e) {
          error(typeElement, "Unable to write view binder for type %s: %s", typeElement,
              e.getMessage());
        }
      }
    }

    return true;
  }
```  

首先调用了findAndParseTargets获取到注解。。以下是findAndroidParseTargets方法(省略部分代码)。。  

```java
private Map<TypeElement, BindingClass> findAndParseTargets(RoundEnvironment env) {
    Map<TypeElement, BindingClass> targetClassMap = new LinkedHashMap<>();
    Set<TypeElement> erasedTargetNames = new LinkedHashSet<>();

    scanForRClasses(env);

    ……

    // Process each @BindView element.
    for (Element element : env.getElementsAnnotatedWith(BindView.class)) {
      if (!SuperficialValidation.validateElement(element)) continue;
      try {
        parseBindView(element, targetClassMap, erasedTargetNames);
      } catch (Exception e) {
        logParsingError(element, BindView.class, e);
      }
    }
	……
    // Process each annotation that corresponds to a listener.
    for (Class<? extends Annotation> listener : LISTENERS) {
      findAndParseListener(env, listener, targetClassMap, erasedTargetNames);
    }

    // Try to find a parent binder for each.
    for (Map.Entry<TypeElement, BindingClass> entry : targetClassMap.entrySet()) {
      TypeElement parentType = findParentType(entry.getKey(), erasedTargetNames);
      if (parentType != null) {
        BindingClass bindingClass = entry.getValue();
        BindingClass parentBindingClass = targetClassMap.get(parentType);
        bindingClass.setParent(parentBindingClass);
      }
    }

    return targetClassMap;
  }
```  

其中scanForRClasses(env）这个方法，老是说我也没有看得很懂。先暂时略过，直接往下看吧。  

```java
for (Element element : env.getElementsAnnotatedWith(BindView.class)) {
      if (!SuperficialValidation.validateElement(element)) continue;
      try {
        parseBindView(element, targetClassMap, erasedTargetNames);
      } catch (Exception e) {
        logParsingError(element, BindView.class, e);
      }
    }
```  

这里遍历了所有使用了Bindview注解的对象并进行校验，如果通过就调用parseBindView()方法。我们再进到这个方法中去。。  

```java
  private void parseBindView(Element element, Map<TypeElement, BindingClass> targetClassMap,
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
      error(element, "@%s fields must extend from View or be an interface. (%s.%s)",
          BindView.class.getSimpleName(), enclosingElement.getQualifiedName(),
          element.getSimpleName());
      hasError = true;
    }

    if (hasError) {
      return;
    }

    // Assemble information on the field.
    int id = element.getAnnotation(BindView.class).value();

    BindingClass bindingClass = targetClassMap.get(enclosingElement);
    if (bindingClass != null) {
      ViewBindings viewBindings = bindingClass.getViewBinding(getId(id));
      if (viewBindings != null && viewBindings.getFieldBinding() != null) {
        FieldViewBinding existingBinding = viewBindings.getFieldBinding();
        error(element, "Attempt to use @%s for an already bound ID %d on '%s'. (%s.%s)",
            BindView.class.getSimpleName(), id, existingBinding.getName(),
            enclosingElement.getQualifiedName(), element.getSimpleName());
        return;
      }
    } else {
      bindingClass = getOrCreateTargetClass(targetClassMap, enclosingElement);
    }

    String name = element.getSimpleName().toString();
    TypeName type = TypeName.get(elementType);
    boolean required = isFieldRequired(element);

    FieldViewBinding binding = new FieldViewBinding(name, type, required);
    bindingClass.addField(getId(id), binding);

    // Add the type-erased version to the valid binding targets set.
    erasedTargetNames.add(enclosingElement);
  }
```  

前半部分是做一些检验类型的操作，后面才进行解析操作。。  

首先获取到注解中包含的资源Id然后判断targetCallMap中是否曾先保存过BindingClass，如果有就直接使用，没有就调用getOrCreateTargetClass创建一个。。  

getOrCreateTargetClass方法如下:  

```java  
private BindingClass getOrCreateTargetClass(Map<TypeElement, BindingClass> targetClassMap,
      TypeElement enclosingElement) {
    BindingClass bindingClass = targetClassMap.get(enclosingElement);
    if (bindingClass == null) {
      TypeName targetType = TypeName.get(enclosingElement.asType());
      if (targetType instanceof ParameterizedTypeName) {
        targetType = ((ParameterizedTypeName) targetType).rawType;
      }

      String packageName = getPackageName(enclosingElement);
      String className = getClassName(enclosingElement, packageName);
      ClassName binderClassName = ClassName.get(packageName, className + "_ViewBinder");
      ClassName unbinderClassName = ClassName.get(packageName, className + "_ViewBinding");

      boolean isFinal = enclosingElement.getModifiers().contains(Modifier.FINAL);

      bindingClass = new BindingClass(targetType, binderClassName, unbinderClassName, isFinal);
      targetClassMap.put(enclosingElement, bindingClass);
    }
    return bindingClass;
  }
```  

这里所做的操作是创建一个BindingClass对象，并将我们注解所在类的雷名加上_ViewBinde作为参数用来传入。。
接着又回到parseBindView中将所获的的注解值和名称类型用以创建FieldViewBinding对象，然后将其传入到bindingclass对象中。。  

那么如此一来所有的注解信息都被包含到BindingClass对象中了。现在我们再重新回到一开始的程序入口process方法。  

```java
@Override public boolean process(Set<? extends TypeElement> elements, RoundEnvironment env) {
    Map<TypeElement, BindingClass> targetClassMap = findAndParseTargets(env);

    for (Map.Entry<TypeElement, BindingClass> entry : targetClassMap.entrySet()) {
      TypeElement typeElement = entry.getKey();
      BindingClass bindingClass = entry.getValue();

      for (JavaFile javaFile : bindingClass.brewJava()) {
        try {
          javaFile.writeTo(filer);
        } catch (IOException e) {
          error(typeElement, "Unable to write view binder for type %s: %s", typeElement,
              e.getMessage());
        }
      }
    }

    return true;
  }
```  

现在我们知道targetClassMap中就是包含了所有注解对象的BindClass对象。。  
然后接着的操作就是遍历出BindClass对象然后将其写入到java文件中。。  

这样一来在你构建项目的时候就会在/build/generated/source/apt/debug/包名/下看到带有"$$ViewBinder"结尾的类。。
里面的内容大致如下:


```java
public class MainActivity$$ViewBinder<T extends MainActivity> implements ViewBinder<T> {
  @Override
  public Unbinder bind(Finder finder, T target, Object source) {
    return new InnerUnbinder<>(target, finder, source);
  }

  protected static class InnerUnbinder<T extends MainActivity> implements Unbinder {
    protected T target;

    protected InnerUnbinder(T target, Finder finder, Object source) {
      this.target = target;

      target.textView = finder.findRequiredViewAsType(source, 2131755137, "field 'rbtnMain'", TextView.class);
    }

    @Override
    public void unbind() {
      T target = this.target;
      if (target == null) throw new IllegalStateException("Bindings already cleared.");

      target.textView = null;
      this.target = null;
    }
  }
}
```  

当然到此为止我们的程序还未调用到这个类。那究竟是怎么调用到这个类的呢。。  
就在这句话里：  

```java
ButterKnife.bind(this);
```  

而就在这个方法中对资源进行了解析。我们往里面解析。。  

```java
  public static Unbinder bind(@NonNull Activity target) {
    return getViewBinder(target).bind(Finder.ACTIVITY, target, target);
  }
  static ViewBinder<Object> getViewBinder(@NonNull Object target) {
    Class<?> targetClass = target.getClass();
    if (debug) Log.d(TAG, "Looking up view binder for " + targetClass.getName());
    return findViewBinderForClass(targetClass);
  }
  private static ViewBinder<Object> findViewBinderForClass(Class<?> cls) {
    ViewBinder<Object> viewBinder = BINDERS.get(cls);
    if (viewBinder != null) {
      if (debug) Log.d(TAG, "HIT: Cached in view binder map.");
      return viewBinder;
    }
    String clsName = cls.getName();
    if (clsName.startsWith("android.") || clsName.startsWith("java.")) {
      if (debug) Log.d(TAG, "MISS: Reached framework class. Abandoning search.");
      return NOP_VIEW_BINDER;
    }
    //noinspection TryWithIdenticalCatches Resolves to API 19+ only type.
    try {
      Class<?> viewBindingClass = Class.forName(clsName + "$$ViewBinder");
      //noinspection unchecked
      viewBinder = (ViewBinder<Object>) viewBindingClass.newInstance();
      if (debug) Log.d(TAG, "HIT: Loaded view binder class.");
    } catch (ClassNotFoundException e) {
      if (debug) Log.d(TAG, "Not found. Trying superclass " + cls.getSuperclass().getName());
      viewBinder = findViewBinderForClass(cls.getSuperclass());
    } catch (InstantiationException e) {
      throw new RuntimeException("Unable to create view binder for " + clsName, e);
    } catch (IllegalAccessException e) {
      throw new RuntimeException("Unable to create view binder for " + clsName, e);
    }
    BINDERS.put(cls, viewBinder);
    return viewBinder;
  }
```

可以看出这里是通过反射获取到我们生成的Class类的对象。也就是我们上文提到的，然后在调用其的bind方法。。  

到这里我们的ButterKnife流程就走完了。。
