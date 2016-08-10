---
layout: post
title: VirtualApp解析第二弹
---

## VirtualApp解析第二弹

这一节用来解析BinderProvider这个类的创建流程。  
BinderProvider是继承自ContentProvider，是从Application启动时就创建的。  
在它的onCreate方法中做了如下操作：  
```java
public boolean onCreate() {
        Context context = getContext();
        KeepService.startup(context);
        if (!VirtualCore.getCore().isStartup()) {
            return true;
        }
        AppFileSystem.getDefault();
        VAppService.getService().systemReady();
        addService(ServiceManagerNative.APP_MANAGER, VAppService.getService());
        addService(ServiceManagerNative.PROCESS_MANAGER, VProcessService.getService());
        VPackageService.getService().onCreate(context);
        addService(ServiceManagerNative.PACKAGE_MANAGER, VPackageService.getService());
        VActivityService.systemReady(context);
        addService(ServiceManagerNative.ACTIVITY_MANAGER, VActivityService.getService());
        addService(ServiceManagerNative.SERVICE_MANAGER, VServiceService.getService());
        VContentService.systemReady(context);
        addService(ServiceManagerNative.CONTENT_MANAGER, VContentService.getService());
        VAccountService.systemReady(context);
        addService(ServiceManagerNative.ACCOUNT_MANAGER, VAccountService.getSingleton());
        return true;
    }
```


这里主要是初始化一些Service并调用addService（） 


```java  
private void addService(String name, IBinder service) {
	ServiceCache.addService(name, service);
}

```

```java
public class ServiceCache {

    private static final Map<String, IBinder> sCache = new HashMap<String, IBinder>(5);

    public static void addService(String name, IBinder service) {
        sCache.put(name, service);
    }

    public static IBinder removeService(String name) {
        return sCache.remove(name);
    }

    public static IBinder getService(String name) {
        return sCache.get(name);
    }


}
```


addService是将传入的变量存储在sCache这个Map中。  
在BinderProvider中还定义了一个名叫ServiceFetcher的内部类。用来操作ServiceCache这个类。  


```java
    private class ServiceFetcher extends IServiceFetcher.Stub {
        @Override
        public IBinder getService(String name) throws RemoteException {
            if (name != null) {
                return ServiceCache.getService(name);
            }
            return null;
        }

        @Override
        public void addService(String name, IBinder service) throws RemoteException {
            if (name != null && service != null) {
                ServiceCache.addService(name, service);
            }
        }

        @Override
        public void removeService(String name) throws RemoteException {
            if (name != null) {
                ServiceCache.removeService(name);
            }
        }
    }
```


这个类的作用无非就是提供外部对ServiceChache的操作。但是是怎么将这个开放到外部的呢？？  
在BinderProvider的call中给出了答案。。  


```java
    public Bundle call(String method, String arg, Bundle extras) {
        if (method.equals(MethodConstants.INIT_SERVICE)) {
            // 确保ServiceContentProvider所在进程创建，因为一切插件服务都依赖这个桥梁。
            return null;
        } else {
            Bundle bundle = new Bundle();
            BundleCompat.putBinder(bundle, ExtraConstants.EXTRA_BINDER, mServiceFetcher);
            return bundle;
        }
    }
```


那么是在哪里调用并获取到的这个对象呢？？  
在ServiceManagerNative我们可以看到。。  


```java
	private static final String SERVICE_CP_AUTH = "virtual.service.BinderProvider";

	public static IServiceFetcher getServiceFetcher() {
		Context context = VirtualCore.getCore().getContext();
		Bundle response = new ProviderCaller.Builder(context, SERVICE_CP_AUTH).methodName("@").call();
		if (response != null) {
			IBinder binder = BundleCompat.getBinder(response, ExtraConstants.EXTRA_BINDER);
			linkBinderDied(binder);
			return IServiceFetcher.Stub.asInterface(binder);
		}
		return null;
	}
```


ProviderCaller这个类是对ContentResolver的call操作做了封装。也就是在这里调用了BinderProvider中的call方法。并且在ServiceManagerNative中使用getService（）方法获取到之前在BinderProvider初始化并保存的service对象。  
那么这样在其他类中就可以通过ServiceManagerNative的getService（）方法获取到保存的对象。
