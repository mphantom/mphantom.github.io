## Rxjava解析第一弹  

现在项目中开发基本上都会用到Dagger2+rxjava+butterknife（合称三剑客）因为真的很好用啊。  

而且现在rxjava开始开发2.x版本了，本着使用一个东西就需要了解这个东西的原理的精神，希望在2.x版本发布之前先搞懂1.x版本的原理，而且在rxajava的github上说RxJava 2.0 has been completely rewritten from scratch on top of the Reactive-Streams specification.而且在2.x出来这一段时间内必然会因为语言不同和文档缺乏而导致使用不开，所以了解1.x版本的原理很重要。  

我们使用rxjava一般都是按照这种方式来使用：  


```java
 Observable.create(new Observable.OnSubscribe<Integer>() {
    @Override
    public void call(Subscriber<? super Integer> subscriber) {
        subscriber.onNext(1);
    }
}).subscribe(new Subscriber<Integer>() {
    @Override
    public void onCompleted() {

    }

    @Override
    public void onError(Throwable e) {

    }

    @Override
    public void onNext(Integer integer) {

    }
});
```


那么在Observable.create（）中做了什么操作呢？  

```java
public static <T> Observable<T> create(OnSubscribe<T> f) {
        return new Observable<T>(hook.onCreate(f));
    }
```  

代码出乎意料的简单，就是直接new了一个Observable对象。。 我们再看到Observable的构造器。。  


```java
    final OnSubscribe<T> onSubscribe;
    protected Observable(OnSubscribe<T> f) {
        this.onSubscribe = f;
    }
```  

就是将传入的值赋值给成员变量。。 
```java
static final RxJavaObservableExecutionHook hook = RxJavaPlugins.getInstance().getObservableExecutionHook();
```  


而这个传入的值是通过RxJavaObservableExecutionHook的onCreate方法产生的。  
```java
public abstract class RxJavaObservableExecutionHook {
    public <T> OnSubscribe<T> onCreate(OnSubscribe<T> f) {
        return f;
    }
}
```  

而RxJavaObservableExecutionHook的onCreate方法更简单就是将传入的值再返回回去。这就相当于我们在调用Observable.create()时就是将传入的OnSubscribe对象赋值给Observable的成员变量onSubscribe而已。。  

接下来我们再来研究Observable在subscribe的时候做了什么操作。  

```java
public final Subscription subscribe(Subscriber<? super T> subscriber) {
        return Observable.subscribe(subscriber, this);
    }
    
    static <T> Subscription subscribe(Subscriber<? super T> subscriber, Observable<T> observable) {
     // validate and proceed
        if (subscriber == null) {
            throw new IllegalArgumentException("subscriber can not be null");
        }
        if (observable.onSubscribe == null) {
            throw new IllegalStateException("onSubscribe function can not be null.");
            /*
             * the subscribe function can also be overridden but generally that's not the appropriate approach
             * so I won't mention that in the exception
             */
        }
        
        // new Subscriber so onStart it
        subscriber.onStart();
        
        /*
         * See https://github.com/ReactiveX/RxJava/issues/216 for discussion on "Guideline 6.4: Protect calls
         * to user code from within an Observer"
         */
        // if not already wrapped
        if (!(subscriber instanceof SafeSubscriber)) {
            // assign to `observer` so we return the protected version
            subscriber = new SafeSubscriber<T>(subscriber);
        }

        // The code below is exactly the same an unsafeSubscribe but not used because it would 
        // add a significant depth to already huge call stacks.
        try {
            // allow the hook to intercept and/or decorate
            hook.onSubscribeStart(observable, observable.onSubscribe).call(subscriber);
            return hook.onSubscribeReturn(subscriber);
        } catch (Throwable e) {
            // special handling for certain Throwable/Error/Exception types
            Exceptions.throwIfFatal(e);
            // in case the subscriber can't listen to exceptions anymore
            if (subscriber.isUnsubscribed()) {
                RxJavaPluginUtils.handleException(hook.onSubscribeError(e));
            } else {
                // if an unhandled error occurs executing the onSubscribe we will propagate it
                try {
                    subscriber.onError(hook.onSubscribeError(e));
                } catch (Throwable e2) {
                    Exceptions.throwIfFatal(e2);
                    // if this happens it means the onError itself failed (perhaps an invalid function implementation)
                    // so we are unable to propagate the error correctly and will just throw
                    RuntimeException r = new OnErrorFailedException("Error occurred attempting to subscribe [" + e.getMessage() + "] and then again while trying to pass to onError.", e2);
                    // TODO could the hook be the cause of the error in the on error handling.
                    hook.onSubscribeError(r);
                    // TODO why aren't we throwing the hook's return value.
                    throw r;
                }
            }
            return Subscriptions.unsubscribed();
        }
    }
```  

真正起作用的只有这么一句。。  

```java
hook.onSubscribeStart(observable, observable.onSubscribe).call(subscriber);
```  

而onSubscribeStart这个方法也是直接将传入的onSubscribe直接返回。  

```java
public <T> OnSubscribe<T> onSubscribeStart(Observable<? extends T> observableInstance, final OnSubscribe<T> onSubscribe) {
        // pass through by default
        return onSubscribe;
    }
```

也就相当于直接调用Observable的成员变量onSubscribe的call方法，并将subscriber参数传入。这就是我们一开时自己实现的方法。
而且通过我们分析可以了解到rxjava中程序真正执行的时候时在调用subscribe方法的时候，而且还有一点没讲到的就是在调用OnSubscribe的call方法之前还会执行Subscriber的onStart方法


