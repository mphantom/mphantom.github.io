
---
layout:post
title:virtualAPP解析第一弹
---

VirtualCore.java 
```java
patchManager.injectAll();
patchManager.checkEnv();
```

injectAll中调用injectInternal（）方法
```java
private void injectInternal() throws Throwable {
		addPatch(new ActivityManagerPatch());
		addPatch(new PackageManagerPatch());

		if (VirtualCore.getCore().isVAppProcess()) {
			addPatch(HCallbackHook.getDefault());
		....略
		}
	}
```
在这个方法中将需要注入的类都加入到injectableMap中。  
所有注入的对象都实现了Injectable这个接口。  
然后在checkEnv()实现注入
```java  
public void checkEnv() throws Throwable {
		for (Injectable injectable : injectableMap.values()) {
			if (injectable.isEnvBad()) {
				injectable.inject();
			}
		}
	}
```
现在就以ActivityManagerPatch为例，看看内部做了什么操作。
然后在跟进到ActivityManagerPatch去看看。  
```java
@Patch({Hook_StartActivities.class, Hook_StartActivity.class, Hook_StartActivityAsCaller.class,
		Hook_StartActivityAsUser.class, Hook_GetIntentSender.class, Hook_RegisterReceiver.class,
		Hook_GetContentProvider.class, Hook_GetContentProviderExternal.class,
略.....})
public class ActivityManagerPatch extends PatchObject<IActivityManager, HookObject<IActivityManager>>
```
ActivityManagerPatch继承自PatchObject，通过一个注解传出了好多参数，那这个注解的作用是什么呢？？？  
在这个类里没有讲到，我们先进到PatchObject去看看。  
```java  
public abstract class PatchObject<T, H extends IHookObject<T>> implements Injectable {
	....略
	public PatchObject() {
		this.hookObject = initHookObject();
		applyHooks();
		afterHookApply(hookObject);
	}
```
在构造函数中调用了applyHooks这个方法在这个方法中完成了对@Path这个注解的解析  
```java
protected void applyHooks() {
		if (hookObject != null) {
			Class<? extends PatchObject> clazz = getClass();
			Patch patch = clazz.getAnnotation(Patch.class);
			int version = Build.VERSION.SDK_INT;
			if (patch != null) {
				Class<? extends Hook>[] hookTypes = patch.value();
				for (Class<? extends Hook> hookType : hookTypes) {
					ApiLimit apiLimit = hookType.getAnnotation(ApiLimit.class);
					boolean needToAddHook = true;
					if (apiLimit != null) {
						int apiStart = apiLimit.start();
						int apiEnd = apiLimit.end();
						boolean highThanStart = apiStart == -1 || version > apiStart;
						boolean lowThanEnd = apiEnd == -1 || version < apiEnd;
						if (!highThanStart || !lowThanEnd) {
							needToAddHook = false;
						}
					}
					if (needToAddHook) {
						addHook(hookType);
					}
				}

			}
		}
	}
```
在这里主要就是解析出这个注解的值，并调用addHook方法，我们再进到addHook这个方法中去  
```java
protected void addHook(Class<? extends Hook> hookType) {
		try {
			Constructor<?> constructor = hookType.getDeclaredConstructors()[0];
			if (!constructor.isAccessible()) {
				constructor.setAccessible(true);
			}
			Hook hook = (Hook) constructor.newInstance(this);
			hookObject.addHook(hook);
		} catch (Throwable e) {
			throw new RuntimeException("Unable to instance Hook : " + hookType + " :" + e.getMessage());
		}
	}
```
这里做的就是实例化出对象，然后调用hookObject.addHook()方法并传值。  
那hookObject又是什么呢，它是一个由子类指定的泛型。是一个实现了IHookObject接口的类型，但是我们是从ActivityManagerPatch开始解析的，那么就直接从实现了这个接口的HookObject开始看吧。。
```java
public void addHook(Hook hook) {
		if (hook != null && !TextUtils.isEmpty(hook.getName())) {
			if (internalHookMapping.containsKey(hook.getName())) {
				XLog.w(TAG, "Hook(%s) from class(%s) have been added, can't add again.", hook.getName(),
						hook.getClass().getName());
			}
			internalHookMapping.put(hook.getName(), hook);
		}
	}
```
这里做的就是将hook对象加入到internalHookMapping中。  


那hookObject对象又是哪里来的呢？？我们回到PatchObject的构造方法。
而hookObject是通过调用initHookObject（）方法获取到的。  
在ActivityManagerPath中这个方法如下。
```java
@Override
	protected HookObject<IActivityManager> initHookObject() {
		return new HookObject<IActivityManager>(getAMN());
	}
```
就是直接构建了一个HookObject对象。。 
```java
public HookObject(ClassLoader cl, T baseObject, Class<?>... proxyInterfaces) {
		this.mBaseObject = baseObject;
		if (mBaseObject != null) {
			if (proxyInterfaces == null) {
				proxyInterfaces = baseObject.getClass().getInterfaces();
			}
			mProxyObject = (T) Proxy.newProxyInstance(cl, proxyInterfaces, new HookHandler());
		}
	}
```
在构造方法中做了动态代理，对于ActivtityManagerPath这个类来说也就是动态代理了IActivityManager这个接口。。
到这里为止ActivtityManagerPath的构造方法结束了。。  
在回到一开始的inject（）这个方法。。又回到原点了：）  
```java
public void inject() throws Throwable {

		Field f_gDefault = ActivityManagerNative.class.getDeclaredField("gDefault");
		if (!f_gDefault.isAccessible()) {
			f_gDefault.setAccessible(true);
		}
		if (f_gDefault.getType() == IActivityManager.class) {
			f_gDefault.set(null, getHookObject().getProxyObject());

		} else if (f_gDefault.getType() == Singleton.class) {

			Singleton gDefault = (Singleton) f_gDefault.get(null);
			Field f_mInstance = Singleton.class.getDeclaredField("mInstance");
			if (!f_mInstance.isAccessible()) {
				f_mInstance.setAccessible(true);
			}
			f_mInstance.set(gDefault, getHookObject().getProxyObject());
		} else {
			// 不会经过这里
			throw new UnsupportedOperationException("Singleton is not visible in AMN.");
		}

		HookBinder<IActivityManager> hookAMBinder = new HookBinder<IActivityManager>() {
			@Override
			protected IBinder queryBaseBinder() {
				return ServiceManager.getService(Context.ACTIVITY_SERVICE);
			}

			@Override
			protected IActivityManager createInterface(IBinder baseBinder) {
				return getHookObject().getProxyObject();
			}
		};
		hookAMBinder.injectService(Context.ACTIVITY_SERVICE);
	}
```
这里前半部分所做的事情就是将我们之前动态代理IActivityManager的返回值代理到IActivityManager。。  
这里让我们看看HookBinder这个类起的什么作用。。  
```java
private static Map<String, IBinder> sCache;

	static {
		try {
			Class.forName(ServiceManager.class.getName());
			Field f_sCache = ServiceManager.class.getDeclaredField("sCache");
			f_sCache.setAccessible(true);
			sCache = (Map<String, IBinder>) f_sCache.get(null);
		} catch (Throwable e) {
			// 不考虑
		}
	}
```
一开始就从获取了ServiceManager的sCache，然后赋值给本地的sCache。  
然后构造函数又做了什么呢？？？  
```java
	public HookBinder() {
		bind();
	}
    public final void bind() {
		this.baseBinder = queryBaseBinder();
		if (baseBinder != null) {
			this.mBaseObject = createInterface(baseBinder);
			mProxyObject = (Interface) Proxy.newProxyInstance(mBaseObject.getClass().getClassLoader(),
					mBaseObject.getClass().getInterfaces(), new HookHandler());

		}
	}
```
直接调用了bind（）函数，而bind（）函数中queryBaseBinder（）和createInterface（）又是在上面的那个匿名类中实现的。 这里所多的就是又一次代理
