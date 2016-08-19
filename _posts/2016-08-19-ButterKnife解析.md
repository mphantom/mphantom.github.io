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

首先调用了findAndParseTargets获取到注解