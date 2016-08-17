---
layout: post
title: android studio 常见注解使用
---

注解在anroid中的应用范围很广，而且android源码中你可以看到很多的注解。。  
那么哪些注解有什么作用呢？  
接下来就来解释一下  
先拿一个最简单的@StringRes来说，在TextView的setText中就使用了该注解。  

```java
public final void setText(@StringRes int resid) {
        setText(getContext().getResources().getText(resid));
    }
```

而我们一般使用这个函数时都是这样用的。  

```java
tv.setText(R.string.helloworld);
```  

这样是不是一眼就明白了@StringRes的作用，没错这是用来标志一个String资源的Id，必须时int型而且必须是String类型的资源不然编译器就会提示你出错了。  
除了这个之外还有好多@ColorRes,@AnimatorRes等等Res结尾的注解，基本上都是用来指定某一类型的资源文件ID。  

在比如我们在开发中有时需要传入的值在指定范围之内，这时候可能会使用到枚举类型，而我们都知道枚举在android中会占据大量资源所以这个也可以通过注解来实现。。  

```java
	@IntDef({A,B})
    @Retention(RetentionPolicy.SOURCE)
    public @interface AB {
    }

    public final static int A = 1;
    public final static int B = 2;
```  

而在需要指定传入值在A和B两个之间的函数时可以这样写：  

```java
void setAB(@AB int value){}
```  
这样编辑器就会限制你传入的值必须满足A或B才行，否则依旧会报红。。
  