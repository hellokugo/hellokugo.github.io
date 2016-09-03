title: 简析DroidPlugin之Hook
date: 2016/6/14 17：55

categories:
- Mobile
tags:
- android
---

# 前言 #
被誉为“安卓黑科技”的开源框架DroidPlugin，是目前较为流行的安卓动态加载技术的典范。免安装和免修改运行第三方sdk，实现多团队协助开发，是360推出这个牛逼able框架的主打亮点。那么，它是如何一步步去实现这个神奇的框架的呢？又和我们常见的动态加载技术有何区别去值得我们学习的？由于目前国内对这个框架没太多的研究，很多地方只能做到点到即止。本文主要分享其hook的过程。<!-- more -->

## 了解入口架构 ##

首先，看到application的attachBaseContext方法里面做了这样一步：

	# PluginApplication.java

    PluginHelper.getInstance().applicationAttachBaseContext(base);

这一步并没有做太多操作，只是注册了crash之后的log发送之类的信息。这里不展开细说。

在onCreate方法里有：

	# PluginApplication.java

    PluginHelper.getInstance().applicationOnCreate(getBaseContext());

这一步是核心操作，进行了一系列的初始化，本文也是从这里开始展开。跟代码找到了initPlugin方法：

	# PluginHelper.java

    private void initPlugin(Context baseContext) {
        long b = System.currentTimeMillis();
        try {
            try {
                fixMiUiLbeSecurity();
            } catch (Throwable e) {
                Log.e(TAG, "fixMiUiLbeSecurity has error", e);
            }
 
            try {
                PluginProcessManager.installHook(baseContext);
            } catch (Throwable e) {
                Log.e(TAG, "installHook has error", e);
            }
 		......
    }

看到了 PluginProcessManager.installHook(baseContext); 这是安装钩子函数（hook）相关的入口。接着往下看：

	# HookFactory.java

    public final void installHook(Context context, ClassLoader classLoader) throws Throwable {
        installHook(new IClipboardBinderHook(context), classLoader);
        //for ISearchManager
        installHook(new ISearchManagerBinderHook(context), classLoader);
        //for INotificationManager
        installHook(new INotificationManagerBinderHook(context), classLoader);
 		..........
 
        installHook(new IPackageManagerHook(context), classLoader);
        installHook(new IActivityManagerHook(context), classLoader);
        installHook(new PluginCallbackHook(context), classLoader);
		.......
    }

很明显，这里做了一系列系统组件的hook操作，这里以

	# HookFactory.java

    installHook(new IActivityManagerHook(context), classLoader);

为例，实例化 IActivityManagerHook ，这个类继承谁呢？查看该类hook的体系结构：

    Hook
    --| ProxyHook
    ----| IActivityManagerHook

基类Hook.java构造方法做了两步操作，一是初始化了hostContext，二是创建hookHandle：

	# Hook.java
	
    protected Hook(Context hostContext) {
        mHostContext = hostContext;
        mHookHandles = createHookHandle();
    }

跟着代码走到了IActivityManagerHookHandle的init()方法。咦，代码看上去貌似很长，其实就是用一个Map去存放方法名和对应操作的键值对，这里先有个概念，后面会用到。

IActivityManagerHook实例化完之后，就是installHook的方法，先贴上代码：

	# HookFactory.java 

    public void installHook(Hook hook, ClassLoader cl) {
        try {
            hook.onInstall(cl);
            synchronized (mHookList) {
                mHookList.add(hook);
            }
        } catch (Throwable throwable) {
            Log.e(TAG, "installHook %s error", throwable, hook);
        }
    }

首先，这里对刚才实例化的hook调用了onInstall方法，并传入一个classLoder，这里为空；然后把hook添加到一个List集合中去。我们看一下IActivityManagerHook的onInstall方法的前半部分：

	# IActivityManagerHook.java

    public void onInstall(ClassLoader classLoader) throws Throwable {
    	Class cls = ActivityManagerNativeCompat.Class();
    	Object obj = FieldUtils.readStaticField(cls, "gDefault");
   	 	if (obj == null) {
    		ActivityManagerNativeCompat.getDefault();
    		obj = FieldUtils.readStaticField(cls, "gDefault");
   		 }
    
    	.............
    }

整个onInstall方法做了两件事情：</br>
1.找到系统的IActivityManager;</br>
2.判断该IActivityManager是否为单例，做不同的处理

## 找到Hook对象 ##

我们看一下在IAcitivityManager里面"gDefault"这个变量是干嘛的，贴源码：

	# ActivityManagerNative.java

    private static final Singleton<IActivityManager> gDefault = new Singleton<IActivityManager>() {
        protected IActivityManager create() {
            IBinder b = ServiceManager.getService("activity");
            if (false) {
                Log.v("ActivityManager", "default service binder = " + b);
            }
            IActivityManager am = asInterface(b);
            if (false) {
                Log.v("ActivityManager", "default service = " + am);
            }
            return am;
        }
    };

我们知道，Android中ActivityManager是与系统所有运行着的Activity进行交互，对系统所有运行的Activity进行管理和维护，但是这些信息真正维护并不是由ActivityManager来负责。读起来有点绕，RTFSC是王道，截取其中一段获取运行服务的方法：

	# ActivityManager.java

    public List<RunningServiceInfo> getRunningServices(int maxNum)
            throws SecurityException {
        try {
            return ActivityManagerNative.getDefault()
                    .getServices(maxNum, 0);
        } catch (RemoteException e) {
            // System dead, we will be dead too soon!
            return null;
        }
    }

通过阅读ActivityManager的源码发现，所有的信息都是通过ActivityManagerNative.getDefault()来操作获取。哎呦，这里面到底是干了啥？我们先来看一幅Activity Manager相关类继承层次关系图：

![](http://i.imgur.com/ToILsf2.jpg)

ActivityManagerProxy，见名思意，是一个代理类，同时也是ActivityManagerNative的内部类；ActivityManagerNative是一个抽象类，实现它的是ActivityManagerService，这是一个系统Service组件。
从图中可以看出，使用ActivityManagerProxy，来代理ActivityManagerNative，实质就是代理它的实现类ActivityManagerService；ActivityManagerService是系统统一的Service，运行在独立的进程中，通过系统ServiceManger来获取。


那么，问题来了，ActivityManager运行在一个进程里面，ActivityManagerService运行在另一个进程内，这里面涉及到跨进程的对象访问，毫无疑问，采用的是Binder机制实现。

我们从刚刚截取的ActivityManagerNative.java中定义gDefault的方法可以看到，首先获取“activity”的IBinder

	IBinder b = ServiceManager.getService("activity");

这个可以从以下源码得知，其中sCache存放的是名字和对应IBinder的键值对Map

	# ServiceManager.java

    /**
     * Returns a reference to a service with the given name.
     * 
     * @param name the name of the service to get
     * @return a reference to the service, or <code>null</code> if the service doesn't exist
     */
    public static IBinder getService(String name) {
        try {
            IBinder service = sCache.get(name);
            if (service != null) {
                return service;
            } else {
                return getIServiceManager().getService(name);
            }
        } catch (RemoteException e) {
            Log.e(TAG, "error in getService", e);
        }
        return null;
    }

然后通过 asInterface(b) 就获取了ActivityManagerService实例的本地代理对象ActivityManagerProxy实例：


	# ActivityManagerNative.java

    /**
     * Cast a Binder object into an activity manager interface, generating
     * a proxy if needed.
     */
    static public IActivityManager asInterface(IBinder obj) {
        if (obj == null) {
            return null;
        }
        IActivityManager in =
            (IActivityManager)obj.queryLocalInterface(descriptor);
        if (in != null) {
            return in;
        }

        return new ActivityManagerProxy(obj);
    }

贴上一图，让你瞬间有豁然开朗的感觉有木有：

![](http://i.imgur.com/Au935XN.jpg)

ok，回到我们的最原点，找到系统的IActivityManager要干嘛？当然就是我们的劫持hook了，在说hook之前，要说说动态代理机制。

## 动态代理机制 ##

动态代理，是一种设计模式，可以为其他对象提供一种代理以控制对这个对象的访问。举一个肤浅的栗子：


   	public Object invoke( Object proxy, Method method, Object[] args ) throws Throwable   
      {   
    //1.在转调具体目标对象之前，可以执行一些功能处理

    //2.转调具体目标对象的方法
    return method.invoke( proxied, args);  
    
    //3.在转调具体目标对象之后，可以执行一些功能处理

      }
    } 

所谓代理，就是首先要获得被代理者的控制权，被代理者在正常调用的时候，代理可以在被代理者的所有方法前或后添加想要的操作。从上图简单地理解，就是在2方法（被代理者）添加1和3方法。可以看一下代理结构图：

![](http://i.imgur.com/s0zRfSt.gif)

在动态代理机制中，有两个重要的类，分别是InvocationHandler和Proxy。我们看一下API帮助文档是怎么对两个类做描述的：

InvocationHandler：

    InvocationHandler is the interface implemented by the invocation handler of a proxy instance. 
    
    Each proxy instance has an associated invocation handler. When a method is invoked on a proxy instance, the method invocation is encoded and dispatched to the invoke method of its invocation handler.

每一个动态代理类必须要实现InvocationHandler这个接口，而InvocationHandler这个接口只有唯一一个invoke方法：
    
    Object invoke(Object proxy, Method method, Object[] args) throws Throwable

    proxy:　　指代我们所代理的那个真实对象
    method:　 指代的是我们所要调用真实对象的某个方法的Method对象
    args:　　 指代的是调用真实对象某个方法时接受的参数

Proxy：

    Proxy provides static methods for creating dynamic proxy classes and instances, and it is also the superclass of all dynamic proxy classes created by those methods. 

Proxy这个类的作用是创建一个代理对象的类，它提供了很多方法，在这里我们只需要了解到newProxyInstance 即可：

    public static Object newProxyInstance(ClassLoader loader, Class<?>[] interfaces, InvocationHandler h) throws IllegalArgumentException
    
    loader:　　  一个ClassLoader对象，定义了由哪个ClassLoader对象来对生成的代理对象进行加载   
    interfaces:  一个Interface对象的数组，表示的是我将要给我需要代理的对象提供一组什么接口，如果我提供了一组接口给它，那么这个代理对象就宣称实现了该接口(多态)，这样我就能调用这组接口中的方法了
   	h:　　        一个InvocationHandler对象，表示的是当我这个动态代理对象在调用方法的时候，会关联到哪一个InvocationHandler对象上

ok，有了点概念，我们来继续举个具体点的栗子：

定义一个people的接口：

    public interface People{
    	public void countMoney();
    }

boss类去实现这个接口：

    public class Boss implements people{
		@Override
		public void countMoney(){
			system.out.println("I have too money to burn.");
		}
    } 

定义一个代理类workers，前面说到了，每一个代理类必须实现InvocationHandler 这个接口：

	public class Workers implements InvocationHandler{
		//这个是我们要代理的真实对象
		private Object people;

		//构造方法，给我们要代理的真实对象赋值
		public workers(Object people){
			this.people = people;
		}

		@Override
		public Object invoke(Object object, Method method, Object[] args){
			
			//在代理真实对象之前加上一些操作
			system.out.println("I must word hard because I have no money.");

			//调用真实对象的相关方法
			method.invoke(subject, args);

			//在代理真实对象之后加上一些操作
			system.out.println("Whether you have got money or not,it's wise for you to go back home for the Spring Festival.");
		}
	} 

编写测试类：

	public class test{
		....
		Boss boss = new Boss;
		InvocationHandle handle = new Workers(boss);
		Workers workers = (Workers) Proxy.newProxyInstance(handle.getClass().getClassLoader(),boss.getClass().getInterfaces(),handle);
		
		workers.countMoney();
		
	}

看一下输出结果：

    I must word hard because I have no money.
    I have too money to burn.
    Whether you have got money or not,it's wise for you to go back home for the Spring Festival.

结果很明显了，如果想了解其中的跳转原理，请查看源码，这里就不多作介绍了。

## 开始Hook ##
我们来看看onInstall方法的后半部分：

	# IActivityManagerHook.java

	public void onInstall(ClassLoader classLoader) throws Throwable {
		.....

        if (IActivityManagerCompat.isIActivityManager(obj)) {
            setOldObj(obj);
            Class<?> objClass = mOldObj.getClass();
            List<Class<?>> interfaces = Utils.getAllInterfaces(objClass);
            Class[] ifs = interfaces != null && interfaces.size() > 0 ? interfaces.toArray(new Class[interfaces.size()]) : new Class[0];
            Object proxiedActivityManager = MyProxy.newProxyInstance(objClass.getClassLoader(), ifs, this);
            FieldUtils.writeStaticField(cls, "gDefault", proxiedActivityManager);
            Log.i(TAG, "Install ActivityManager Hook 1 old=%s,new=%s", mOldObj, proxiedActivityManager);
        } 

			.......
        }

首先，判断上面获取找到系统的IActivityManager是否正确，然后setOldObj(obj)保存当前系统的IActivityManager对象，留意IActivityManagerHook类是继承ProxyHook，该类实现InvocationHandler接口，就是一个代理类的实体啊。ok，你应该留意到接下来的操作就是介绍如何一步步实现代理，并把代理对象写到ActivityManagerNative的gDefault变量中，这样就实现hook了：

	Object proxiedActivityManager = MyProxy.newProxyInstance(objClass.getClassLoader(), ifs, this);
    FieldUtils.writeStaticField(cls, "gDefault", proxiedActivityManager);

classLoader和获取所有接口数组都是沿用mOldObj对象。

hook之后就开始干坏事了，把目光转到ProxyHook类中的invoke方法：

	# ProxyHook.java

    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {

        try {
            if (!isEnable()) {
                return method.invoke(mOldObj, args);
            }
            HookedMethodHandler hookedMethodHandler = mHookHandles.getHookedMethodHandler(method);
            if (hookedMethodHandler != null) {
                return hookedMethodHandler.doHookInner(mOldObj, method, args);
            }
            return method.invoke(mOldObj, args);
        } catch (InvocationTargetException e) {
            Throwable cause = e.getTargetException();
         .......
    }	

首先，isEnable()先判断一下这个函数是否可以hook，不可以的话直接返回mOldObj（系统）的原本方法；可以的话继续往下走：

	# ProxyHook.java

	HookedMethodHandler hookedMethodHandler = mHookHandles.getHookedMethodHandler(method);
    if (hookedMethodHandler != null) {
        return hookedMethodHandler.doHookInner(mOldObj, method, args);
    }
	return method.invoke(mOldObj, args);

根据方法名字获取一个hookedMethodHandler，如果不为空，则执行hookedMethodHandler的doHookInner方法；如果为空，依旧返回系统的原本方法。

hookedMethodHandler是什么鬼？还记得上面提到的IActivityManagerHookHandle的init()方法？是的，这里就要用到，先看一下之前的init()代码：

	# IActivityManagerHookHandle.java

    protected void init() {
        sHookedMethodHandlers.put("startActivity", new startActivity(mHostContext));
        sHookedMethodHandlers.put("startActivityAsUser", new startActivityAsUser(mHostContext));
        sHookedMethodHandlers.put("startActivityAsCaller", new startActivityAsCaller(mHostContext));
        sHookedMethodHandlers.put("startActivityAndWait", new startActivityAndWait(mHostContext));
        sHookedMethodHandlers.put("startActivityWithConfig", new startActivityWithConfig(mHostContext));
        sHookedMethodHandlers.put("startActivityIntentSender", new startActivityIntentSender(mHostContext));
        sHookedMethodHandlers.put("startVoiceActivity", new startVoiceActivity(mHostContext));
        sHookedMethodHandlers.put("startNextMatchingActivity", new startNextMatchingActivity(mHostContext));
        sHookedMethodHandlers.put("startActivityFromRecents", new startActivityFromRecents(mHostContext));
      ...........
    }

初始化把所有方法和对应的HookedMethodHandler处理类放在一起。

我们假如传入一个方法名为startActivity，hookedMethodHandler存在，并进入doHookInner方法：

	# HookedMethodHandler.java

    public synchronized Object doHookInner(Object receiver, Method method, Object[] args) throws Throwable {
        long b = System.currentTimeMillis();
        try {
            mUseFakedResult = false;
            mFakedResult = null;
            boolean suc = beforeInvoke(receiver, method, args);
            Object invokeResult = null;
            if (!suc) {
                invokeResult = method.invoke(receiver, args);
            }
            afterInvoke(receiver, method, args, invokeResult);
            if (mUseFakedResult) {
                return mFakedResult;
            } else {
                return invokeResult;
            }
        } 
	.........
    }

从上面看到熟悉的动态代理，先执行beforeInvoke方法，如果结果为true，拦截了该事件；如果为false，还是会继续执行系统原本方法，最后调用afterInvoke方法。在HookedMethodHandler里面beforeInvoke方法默认返回false，afterInvoke方法并没有实现：

	# HookedMethodHandler.java

    /**
     * 在某个方法被调用之前执行，如果返回true，则不执行原始的方法，否则执行原始方法
     */
    protected boolean beforeInvoke(Object receiver, Method method, Object[] args) throws Throwable {
        return false;
    }

    protected void afterInvoke(Object receiver, Method method, Object[] args, Object invokeResult) throws Throwable {
    }

如果需要实现肯定是在子类了，在IActivityManagerHookHandle的init()方法里面，以startActivity为例：

    private static class startActivity extends HookedMethodHandler {

        public startActivity(Context hostContext) {
            super(hostContext);
        }
		......
        @Override
        protected boolean beforeInvoke(Object receiver, Method method, Object[] args) throws Throwable {
            if (Build.VERSION.SDK_INT < Build.VERSION_CODES.JELLY_BEAN_MR2) {
                //2.3
        /*public int startActivity(IApplicationThread caller,
            Intent intent, String resolvedType, Uri[] grantedUriPermissions,
            int grantedMode, IBinder resultTo, String resultWho, int requestCode,
            boolean onlyIfNeeded, boolean debug) throws RemoteException;*/

                //api 15
        /*public int startActivity(IApplicationThread caller,
            Intent intent, String resolvedType, Uri[] grantedUriPermissions,
            int grantedMode, IBinder resultTo, String resultWho, int requestCode,
            boolean onlyIfNeeded, boolean debug, String profileFile,
            ParcelFileDescriptor profileFd, boolean autoStopProfiler) throws RemoteException;*/

                //api 16,17
        /*  public int startActivity(IApplicationThread caller,
            Intent intent, String resolvedType, IBinder resultTo, String resultWho,
            int requestCode, int flags, String profileFile,
            ParcelFileDescriptor profileFd, Bundle options) throws RemoteException;*/
                doReplaceIntentForStartActivityAPILow(args);
            } else {
                //api 18,19
         /*  public int startActivity(IApplicationThread caller, String callingPackage,
            Intent intent, String resolvedType, IBinder resultTo, String resultWho,
            int requestCode, int flags, String profileFile,
            ParcelFileDescriptor profileFd, Bundle options) throws RemoteException;*/

                //api 21
        /*   public int startActivity(IApplicationThread caller, String callingPackage,
            Intent intent, String resolvedType, IBinder resultTo, String resultWho,
            int requestCode, int flags, ProfilerInfo profilerInfo,
            Bundle options) throws RemoteException;*/
                doReplaceIntentForStartActivityAPIHigh(args);
            }
            return super.beforeInvoke(receiver, method, args);
        }
    }

至此，hook的过程应该已经介绍完毕，其中涉及比较高深的问题，小弟完美绕过。






