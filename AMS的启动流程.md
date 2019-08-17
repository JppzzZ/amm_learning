# SystemServer的 **main** 中启动的 AMS

**frameworks/base/services/java/com/android/server/SystemServer.java**

```java
public static void main(String[] args) {
	new SystemServer().run();
}
```

接下来是 **run **方法

**frameworks/base/services/java/com/android/server/SystemServer.java**

```java
private void run() {
	...
	System.loadLibrary("android_servers");//加载了动态库libandroid_servers.so
	...
	// Initialize the system context.
	createSystemContext();//SystemServer在启动任何服务之前，就调用了createSystemContext
	//创建出的Context保存在mSystemContext中

	mSystemServiceManager = new SystemServiceManager(mSystemContext);//创建SystemServiceManager，它会对系统的服务进行创建、启动和生命周期管理
	LocalServices.addService(SystemServiceManager.class, mSystemServiceManager);
	...    
	try {
		Trace.traceBegin(Trace.TRACE_TAG_SYSTEM_SERVER, "StartServices");
		startBootstrapServices();//startBootstrapServices方法中用SystemServiceManager启动了ActivityManagerService、PowerManagerService、PackageManagerService等服务
		startCoreServices();//startCoreServices方法中则启动了BatteryService、UsageStatsService和WebViewUpdateService
		startOtherServices();//startOtherServices方法中启动了CameraService、AlarmManagerService、VrManagerService等服务
	} catch (Throwable ex) {
		Slog.e("System", "******************************************");
		Slog.e("System", "************ Failure starting system services", ex);
		throw ex;
	} finally {
		Trace.traceEnd(Trace.TRACE_TAG_SYSTEM_SERVER);
	}
	...
}
```

从startBootstrapServices、startCoreServices、startOtherServices方法可以看出，官方把系统服务分为了三种类型，分别是引导服务、核心服务和其他服务，这些服务的父类均为SystemService，其中其他服务是一些非紧要和一些不需要立即启动的服务。系统服务总共大约有80多个，AMS 即 ActivityManagerService 属于 BootstrapServices

```java
private void createSystemContext() {
  //调用ActivityThread的systemMain函数，其中会创建出系统对应的Context对象
	ActivityThread activityThread = ActivityThread.systemMain();
  //取出上面函数创建的Context对象，保存在mSystemContext中
	mSystemContext = activityThread.getSystemContext();
  //设置系统主题
	mSystemContext.setTheme(DEFAULT_SYSTEM_THEME);

	final Context systemUiContext = activityThread.getSystemUiContext();
	systemUiContext.setTheme(DEFAULT_SYSTEM_THEME);
}
```

# ActivityThread 的 **systemMain** 创建ActivityThread 并 attch

/frameworks/base/core/java/android/app/ActivityThread.java**

```java
public static ActivityThread systemMain() {
	// The system process on low-memory devices do not get to use hardware
	// accelerated drawing, since this can add too much overhead to the
	// process.
	if (!ActivityManager.isHighEndGfx()) {
    //虽然写着ActivityManager，但和AMS没有任何关系
    //就是利用系统属性和配置信息进行判断
    //关闭硬件渲染功能
		ThreadedRenderer.disable(true);
	} else {
		ThreadedRenderer.enableForegroundTrimming();
	}
  //新建 ActivityThread 并调用 attach 方法
	ActivityThread thread = new ActivityThread();
	thread.attach(true, 0);
	return thread;
}
```

**/frameworks/base/core/java/android/app/ActivityThread.java**

```java
...
//定义了AMS与应用通信的接口
final ApplicationThread mAppThread = new ApplicationThread();

//拥有自己的looper，说明ActivityThread确实可以代表事件处理线程
final Looper mLooper = Looper.myLooper();

//H继承Handler，ActivityThread中大量事件处理依赖此Handler
final H mH = new H();

//用于保存该进程的ActivityRecord
final ArrayMap<IBinder, ActivityClientRecord> mActivities = new ArrayMap<>()
...
//用于保存进程中的Service
final ArrayMap<IBinder, Service> mServices = new ArrayMap<>();
...
//用于保存进程中的Application
final ArrayList<Application> mAllApplications = new ArrayList<Application>();
...
// 构造函数
ActivityThread() {
	mResourcesManager = ResourcesManager.getInstance();
}
...
private void attach(boolean system, long startSeq) {
    sCurrentActivityThread = this;
    mSystemThread = system;
    if (!system) {
    	...
    } else {
        // Don't set application object here -- if the system crashes,
        // we can't display an alert, we just want to die die die.
        android.ddm.DdmHandleAppName.setAppName("system_process",
                UserHandle.myUserId());
        try {
          	//创建ActivityThread中的重要成员：Instrumentation、Application和Context
            mInstrumentation = new Instrumentation();
            mInstrumentation.basicInit(this);
            ContextImpl context = ContextImpl.createAppContext(
                    this, getSystemContext().mPackageInfo);
            mInitialApplication = context.mPackageInfo.makeApplication(true, null);
            mInitialApplication.onCreate();
        } catch (Exception e) {
            throw new RuntimeException(
                    "Unable to instantiate Application():" + e.toString(), e);
        }
    }
    ...
}
```

从上面的代码可以看出，对于系统进程而言，ActivityThread的attach函数最重要的工作就是创建了Instrumentation、Application和Context。

## **Instrumentation** 

Instrumentation是Android中的一个工具类，当该类被启用时，它将优先于应用中其它的类被初始化。 
此时，系统先创建它，再通过它创建其它组件。

此外，系统和应用组件之间的交互也将通过Instrumentation来传递。 
因此，Instrumentation就能监控系统和组件的交互情况了。

##  **Context**

Context是Android中的一个抽象类，用于维护应用运行环境的全局信息。 
通过Context可以访问应用的资源和类，甚至进行系统级的操作，例如启动Activity、发送广播等。

```java
//ContextImpl是Context的实现类
ContextImpl context = ContextImpl.createAppContext(this, getSystemContext().mPackageInfo);
```

先看看ActivityThread中getSystemContext方法：

```java
public ContextImpl getSystemContext() {
		synchronized (this) {
			if (mSystemContext == null) {
				//调用ContextImpl的静态函数createSystemContext
				mSystemContext = ContextImpl.createSystemContext(this);
			}
			return mSystemContext;
		}
}
```

ContextImpl 的 createSystemContext 方法：

```java
static ContextImpl createSystemContext(ActivityThread mainThread) {
    //创建LoadedApk类，代表一个加载到系统中的APK
    //注意此时的LoadedApk只是一个空壳
    //PKMS还没有启动，估无法得到有效的ApplicationInfo
    LoadedApk packageInfo = new LoadedApk(mainThread);
  
    //调用ContextImpl的构造函数
    ContextImpl context = new ContextImpl(null, mainThread, packageInfo, null, null, null, 0,
            null);
    context.setResources(packageInfo.getResources());
  
    //初始化资源信息
    context.mResources.updateConfiguration(context.mResourcesManager.getConfiguration(),
            context.mResourcesManager.getDisplayMetrics());
    return context;
}
```

LoadedApk的构造方法 ，指定packageName 为 “android”，也就是framwork-res.apk。由于该APK仅供SystemServer进程使用，因此创建的Context被定义为System Context：

```java
LoadedApk(ActivityThread activityThread) {
    mActivityThread = activityThread;
    mApplicationInfo = new ApplicationInfo();
  
    //packageName为"android"
    mApplicationInfo.packageName = "android";
    mPackageName = "android";
    ...
}
```

## **ContextImpl.createAppContext**

得到System Contex后，调用 ContextImpl 的 createAppContext 方法：

```java
static ContextImpl createAppContext(ActivityThread mainThread, LoadedApk packageInfo) {
    if (packageInfo == null) throw new IllegalArgumentException("packageInfo");
    ContextImpl context = new ContextImpl(null, mainThread, packageInfo, null, null, null, 0,
            null);
    context.setResources(packageInfo.getResources());
    return context;
}
```

这里就是利用ActivityThread和LoadedApk构造出ContextImpl。 
ContextImpl的构造函数主要是完成一些变量的初始化，建立起ContextImpl与ActivityThread、LoadedApk、ContentResolver之间的关系。 

## **Application**

回到上面的ActivityThread中的attch方法中，针对系统进程，通过下面的代码创建了初始的Application：

```java
	//调用LoadedApk的makeApplication函数
	mInitialApplication = context.mPackageInfo.makeApplication(true, null);

	//启动Application,调用 application 的 onCreate 方法
	mInitialApplication.onCreate();
```

**/frameworks/base/core/java/android/app/LoadedApk.java**

```java
public Application makeApplication(boolean forceDefaultAppClass,
        Instrumentation instrumentation) {
    if (mApplication != null) {
        return mApplication;
    }
    Application app = null;
    String appClass = mApplicationInfo.className;
    if (forceDefaultAppClass || (appClass == null)) {
      	//系统进程中，对应下面的appClass
        appClass = "android.app.Application";
    }
    try {
        java.lang.ClassLoader cl = getClassLoader();
        if (!mPackageName.equals("android")) {
            ...
        }
        ContextImpl appContext = ContextImpl.createAppContext(mActivityThread, this);
   			//此处在mInstrumentation内newApplication最后通过反射创建出Application
        app = mActivityThread.mInstrumentation.newApplication(cl, appClass, appContext);
        appContext.setOuterContext(app);
    } catch (Exception e) {
        ...
    }
  	//一个进程支持多个Application，mAllApplications用于保存该进程中的Application对象
    mActivityThread.mAllApplications.add(app);
    mApplication = app;
    if (instrumentation != null) {
      	// 此时 instrumentation == null
      	...
    }
    // Rewrite the R 'constants' for all library apks.
    ...
    return app;
}
```

至此，createSystemContext函数介绍完毕。

当SystemServer调用createSystemContext完毕后： 
1、得到了一个ActivityThread对象，它代表当前进程 (此时为系统进程) 的主线程； 
2、得到了一个Context对象，对于SystemServer而言，它包含的Application运行环境与framework-res.apk有关。

## 为什么要createSystemContext

Android努力构筑了一个自己的运行环境。 
在这个环境中，进程的概念被模糊化了。组件的运行及它们之间的交互均在该环境中实现。

createSystemContext函数就是为SystemServer进程搭建一个和应用进程一样的Android运行环境。

Android运行环境是构建在进程之上的，应用程序一般只和Android运行环境交互。 
基于同样的道理，SystemServer进程希望它内部运行的应用， 
也通过Android运行环境交互，因此才调用了createSystemContext函数。

创建Android运行环境时， 
由于SystemServer的特殊性，调用了ActivityThread.systemMain函数； 
对于普通的应用程序，将在自己的主线程中调用ActivityThread.main函数。

## 回到上面的代码 **createSystemContext** 之后看 **startBootstrapServices**

**frameworks/base/services/java/com/android/server/SystemServer.java**

```java
private void startBootstrapServices() {
	Installer installer = mSystemServiceManager.startService(Installer.class);
	// Activity manager runs the show.
	mActivityManagerService = mSystemServiceManager.startService(
	ActivityManagerService.Lifecycle.class).getService();//调用了SystemServiceManager的startService方法，方法的参数是ActivityManagerService.Lifecycle.class
	mActivityManagerService.setSystemServiceManager(mSystemServiceManager);
	mActivityManagerService.setInstaller(installer);
	...
}
```

startService方法传入的参数是Lifecycle.class，Lifecycle继承自SystemService。

内部类Lifecycle对于AMS而言，就像一个适配器一样，让AMS能够像SystemService一样被SystemServiceManager通过反射的方式启动。

**frameworks/base/services/core/java/com/android/server/SystemServiceManager.java**

```java
@SuppressWarnings("unchecked")
public <T extends SystemService> T startService(Class<T> serviceClass) {
	try {
		...
		final T service;
		try {
			Constructor<T> constructor = serviceClass.getConstructor(Context.class);//得到传进来的Lifecycle的构造器constructor
			service = constructor.newInstance(mContext);//调用constructor的newInstance方法来创建Lifecycle类型的service对象
		} catch (InstantiationException ex) {
			...
		}
		// Register it.
		mServices.add(service);//将刚创建的service添加到ArrayList类型的mServices对象中来完成注册
		// Start it.
		try {
			service.onStart();//调用service的onStart方法来启动service，并返回该service
		} catch (RuntimeException ex) {
			throw new RuntimeException("Failed to start service " + name + ": onStart threw an exception", ex);
		}
		return service;
	} finally {
		Trace.traceEnd(Trace.TRACE_TAG_SYSTEM_SERVER);
	}
}
```

Lifecycle是AMS的内部类。

**frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java**

```java
   public static final class Lifecycle extends SystemService {
        private final ActivityManagerService mService;
        public Lifecycle(Context context) {
            super(context);
            mService = new ActivityManagerService(context);//创建AMS实例
        }
        @Override
        public void onStart() {
            mService.start();//当调用Lifecycle类型的service的onStart方法时，实际上是调用了此处AMS的start方法
        }
        public ActivityManagerService getService() {
            return mService;//我们知道SystemServiceManager的startService方法最终会返回Lifecycle类型的对象，紧接着又调用了Lifecycle的getService方法，这个方法会返回AMS类型的mService对象，这样AMS实例就会被创建并且返回
        }
    }
```

再看 **ActivityManagerService** 构造函数：

```java
public ActivityManagerService(Context systemContext) {
    ...
  	//AMS的运行上下文与 SystemServer 一致,是上面说到的 createSystemContext 创建的上下文
    mContext = systemContext;
		...
    //取出的是ActivityThread的静态变量sCurrentActivityThread
    //这意味着mSystemThread与SystemServer中的ActivityThread一致
    mSystemThread = ActivityThread.currentActivityThread();
    mUiContext = mSystemThread.getSystemUiContext();
		...
    mHandlerThread = new ServiceThread(TAG,
            THREAD_PRIORITY_FOREGROUND, false /*allowIo*/);
    mHandlerThread.start();
  	//处理AMS中消息的主力
    mHandler = new MainHandler(mHandlerThread.getLooper());
  	//UiHandler对应于Android中的UiThread
    mUiHandler = mInjector.getUiHandler(this);
		...
    /* static; one-time init here */
    if (sKillHandler == null) {
        sKillThread = new ServiceThread(TAG + ":kill",
                THREAD_PRIORITY_BACKGROUND, true /* allowIo */);
        sKillThread.start();
      	//用于接收消息，杀死进程
        sKillHandler = new KillHandler(sKillThread.getLooper());
    }
  
    //创建两个BroadcastQueue，前台的超时时间为10s，后台的超时时间为60s
    mFgBroadcastQueue = new BroadcastQueue(this, mHandler,
            "foreground", BROADCAST_FG_TIMEOUT, false);
    mBgBroadcastQueue = new BroadcastQueue(this, mHandler,
            "background", BROADCAST_BG_TIMEOUT, true);
    mBroadcastQueues[0] = mFgBroadcastQueue;
    mBroadcastQueues[1] = mBgBroadcastQueue;
  
    //创建变量，用于存储信息
    mServices = new ActiveServices(this);
    mProviderMap = new ProviderMap(this);
    mAppErrors = new AppErrors(mUiContext, this);

  	// 文件存储相关，创建 system 文件目录
    File dataDir = Environment.getDataDirectory();
    File systemDir = new File(dataDir, "system");
    systemDir.mkdirs();

    mAppWarnings = new AppWarnings(this, mUiContext, mHandler, mUiHandler, systemDir);
  
		//这一部分，和BatteryStatsService相关，进行BSS的初始化
    // TODO: Move creation of battery stats service outside of activity manager service.
    mBatteryStatsService = new BatteryStatsService(systemContext, systemDir, mHandler);
    mBatteryStatsService.getActiveStatistics().readLocked();
    mBatteryStatsService.scheduleWriteToDisk();
    mOnBattery = DEBUG_POWER ? true
            : mBatteryStatsService.getActiveStatistics().getIsOnBattery();
    mBatteryStatsService.getActiveStatistics().setCallback(this);
  
    //创建ProcessStatsService，感觉用于记录进程运行时的统计信息，例如内存使用情况，写入/proc/stat文件
    mProcessStats = new ProcessStatsService(this, new File(systemDir, "procstats"));

    //启动Android的权限检查服务，并注册对应的回调接口
    mAppOpsService = mInjector.getAppOpsService(new File(systemDir, "appops.xml"), mHandler);

    //用于定义ContentProvider访问指定Uri对应数据的权限
    mGrantFile = new AtomicFile(new File(systemDir, "urigrants.xml"), "uri-grants");

    //创建多用户管理器
    mUserController = new UserController(this);
    mVrController = new VrController(this);

    //获取OpenGL版本
    GL_ES_VERSION = SystemProperties.getInt("ro.opengles.version",
        ConfigurationInfo.GL_ES_VERSION_UNDEFINED);

    if (SystemProperties.getInt("sys.use_fifo_ui", 0) != 0) {
        mUseFifoUiScheduling = true;
    }

    mTrackingAssociations = "1".equals(SystemProperties.get("debug.track-associations"));
    mTempConfig.setToDefaults();
    mTempConfig.setLocales(LocaleList.getDefault());
    mConfigurationSeq = mTempConfig.seq = 1;

    // 以下的类，似乎用于管理和监控AMS维护的Activity Task信息
    // ActivityStackSupervisor是AMS中用来管理Activity启动和调度的核心类
    mStackSupervisor = createStackSupervisor();
    mStackSupervisor.onConfigurationChanged(mTempConfig);
    mKeyguardController = mStackSupervisor.getKeyguardController();
    mCompatModePackages = new CompatModePackages(this, systemDir, mHandler);
    mIntentFirewall = new IntentFirewall(new IntentFirewallInterface(), mHandler);
    mTaskChangeNotificationController =
            new TaskChangeNotificationController(this, mStackSupervisor, mHandler);
    mActivityStartController = new ActivityStartController(this);
    mRecentTasks = createRecentTasks();
    mStackSupervisor.setRecentTasks(mRecentTasks);
    mLockTaskController = new LockTaskController(mContext, mStackSupervisor, mHandler);
    mLifecycleManager = new ClientLifecycleManager();

  	//创建线程用于统计进程的CPU使用情况
    mProcessCpuThread = new Thread("CpuTracker") {
        @Override
        public void run() {
            synchronized (mProcessCpuTracker) {
                mProcessCpuInitLatch.countDown();
                mProcessCpuTracker.init();
            }
            while (true) {
                try {
                    try {
                        //计算更新信息的等待间隔
                        //同时利用wait等待计算出的间隔时间
                        synchronized(this) {
                            final long now = SystemClock.uptimeMillis();
                            long nextCpuDelay = (mLastCpuTime.get()+MONITOR_CPU_MAX_TIME)-now;
                            long nextWriteDelay = (mLastWriteTime+BATTERY_STATS_TIME)-now;
                            //Slog.i(TAG, "Cpu delay=" + nextCpuDelay
                            //        + ", write delay=" + nextWriteDelay);
                            if (nextWriteDelay < nextCpuDelay) {
                                nextCpuDelay = nextWriteDelay;
                            }
                            if (nextCpuDelay > 0) {
                                mProcessCpuMutexFree.set(true);
                                this.wait(nextCpuDelay);
                            }
                        }
                    } catch (InterruptedException e) {
                    }
                    //更新CPU运行统计信息
                    updateCpuStatsNow();
                } catch (Exception e) {
                    Slog.e(TAG, "Unexpected exception collecting process stats", e);
                }
            }
        }
    };

    mHiddenApiBlacklist = new HiddenApiSettings(mHandler, mContext);

    //加入Watchdog的监控
    Watchdog.getInstance().addMonitor(this);
    Watchdog.getInstance().addThread(mHandler);
		...
}

```

从代码来看，AMS的构造函数还是相对比较简单的，主要工作就是初始化一些变量。

接下来调用了 AMS 的 start() 方法

```java
private void start() {
  	//完成统计前的复位工作
    removeAllProcessGroups();
  
  	//开始监控进程的CPU使用情况
    mProcessCpuThread.start();
  
		//注册服务
    mBatteryStatsService.publish();
    mAppOpsService.publish(mContext);
    Slog.d("AppOps", "AppOpsService published");
    LocalServices.addService(ActivityManagerInternal.class, new LocalService());
    // Wait for the synchronized block started in mProcessCpuThread,
    // so that any other acccess to mProcessCpuTracker from main thread
    // will be blocked during mProcessCpuTracker initialization.
    try {
        mProcessCpuInitLatch.await();
    } catch (InterruptedException e) {
        Slog.wtf(TAG, "Interrupted wait during start", e);
        Thread.currentThread().interrupt();
        throw new IllegalStateException("Interrupted wait during start");
    }
}
```

AMS的start函数比较简单，主要是： 
1、启动CPU监控线程。该线程将会开始统计不同进程使用CPU的情况。 
2、发布一些服务，如BatteryStatsService、AppOpsService(权限管理相关)和本地实现的继承ActivityManagerInternal的服务。

## **将SystemServer纳入AMS的管理体系** 

AMS完成启动后，在SystemServer的startBootstrapServices函数中， 
下一个与AMS相关的重要调用就是AMS.setSystemProcess了

```java
private void startBootstrapServices() {
    ...........
    // Set up the Application instance for the system process and get started.
    mActivityManagerService.setSystemProcess();
    ...........
}
```

setSystemProcess 方法

```java
public void setSystemProcess() {
    try {
        //以下是向ServiceManager注册几个服务
        //AMS自己
        ServiceManager.addService(Context.ACTIVITY_SERVICE, this, /* allowIsolated= */ true,
                DUMP_FLAG_PRIORITY_CRITICAL | DUMP_FLAG_PRIORITY_NORMAL | DUMP_FLAG_PROTO);
     
        //注册进程统计信息的服务
        ServiceManager.addService(ProcessStats.SERVICE_NAME, mProcessStats);
      
        //用于打印内存信息用的
        ServiceManager.addService("meminfo", new MemBinder(this), /* allowIsolated= */ false,
                DUMP_FLAG_PRIORITY_HIGH);
      
        //用于输出进程使用硬件渲染方面的信息
        ServiceManager.addService("gfxinfo", new GraphicsBinder(this));
      
        //用于输出数据库相关的信息
        ServiceManager.addService("dbinfo", new DbBinder(this));
      
        if (MONITOR_CPU_USAGE) {
            //用于输出进程的CPU使用情况
            ServiceManager.addService("cpuinfo", new CpuBinder(this),
                    /* allowIsolated= */ false, DUMP_FLAG_PRIORITY_CRITICAL);
        }
      
        //注册权限管理服务
        ServiceManager.addService("permission", new PermissionController(this));
      
        //注册获取进程信息的服务
        ServiceManager.addService("processinfo", new ProcessInfoService(this));

      	//1、向PKMS查询package名为“android”的应用的ApplicationInfo
        ApplicationInfo info = mContext.getPackageManager().getApplicationInfo(
                "android", STOCK_PM_FLAGS | MATCH_SYSTEM_ONLY);
      	//2、调用installSystemApplicationInfo
        mSystemThread.installSystemApplicationInfo(info, getClass().getClassLoader());

     		//3、以下与AMS的进程管理有关
        synchronized (this) {
            ProcessRecord app = newProcessRecordLocked(info, info.processName, false, 0);
            app.persistent = true;
            app.pid = MY_PID;
            app.maxAdj = ProcessList.SYSTEM_ADJ;
            app.makeActive(mSystemThread.getApplicationThread(), mProcessStats);
            synchronized (mPidsSelfLocked) {
                mPidsSelfLocked.put(app.pid, app);
            }
            updateLruProcessLocked(app, false, null);
            updateOomAdjLocked();
        }
    } catch (PackageManager.NameNotFoundException e) {
        throw new RuntimeException(
                "Unable to find android system package", e);
    }
  	...
}
```

从上面的代码可以看出，AMS的setSystemProcess主要有四个主要的功能： 
1、注册一些服务； 
2、获取package名为“android”的应用的ApplicationInfo； 
3、调用ActivityThread的installSystemApplicationInfo； 
4、AMS进程管理相关的操作。

这四个主要的功能中，第一个比较简单，就是用Binder通信完成注册。

### 1. 获取ApplicationInfo 

如前所述，这部分相关的代码为：

```java
ApplicationInfo info = mContext.getPackageManager().getApplicationInfo(
        "android", STOCK_PM_FLAGS | MATCH_SYSTEM_ONLY);
// mContext.getPackageManager() 获取 PackageManager
// getApplicationInfo(...) 获取package名为”android”的ApplicationInfo
```

先看 mContext.getPackageManager() 方法

```java
@Override
public PackageManager getPackageManager() {
    if (mPackageManager != null) {
        return mPackageManager;
    }

    // 依赖于ActivityThread的getPackageManager函数 
    IPackageManager pm = ActivityThread.getPackageManager();
    if (pm != null) {
        // Doesn't matter if we make more than one instance.
        // 利用PKMS的代理对象，构建ApplicationPackageManager
        // 该类继承PackageManager
        return (mPackageManager = new ApplicationPackageManager(this, pm));
    }
    return null;
}
```

跟进一下ActivityThread中的getPackageManager：

```java
public static IPackageManager getPackageManager() {
    if (sPackageManager != null) {
        return sPackageManager;
    }
    //依赖于Binder通信，获取到PKMS对应的BpBinder
    IBinder b = ServiceManager.getService("package");
		//得到PKMS对应的Binder服务代理
    sPackageManager = IPackageManager.Stub.asInterface(b);

    return sPackageManager;
}
```

从上面的代码我们可以看到，AMS获取PKMS用到了Binder通信。

实际上，PKMS由SystemServer创建，与AMS运行在同一个进程。 
AMS完全可以不经过Context、ActivityThread、Binder来获取PKMS。

推断出原生代码这么做的原因是： 
SystemServer进程中的服务，也使用Android运行环境来交互， 
保留了组件之间交互接口的统一，为未来的系统保留了可扩展性。

得到PKMS的代理对象后，AMS调用PKMS的getApplicationInfo接口，获取package名为”android”的ApplicationInfo。

### 2. 调用ActivityThread的installSystemApplicationInfo

```java
	mSystemThread.installSystemApplicationInfo(info, getClass().getClassLoader());
```

AMS中的mSystemThread就是SystemServer中创建出的ActivityThread。 
因此我们跟进一下ActivityThread的installSystemApplicationInfo函数：

```java
public void installSystemApplicationInfo(ApplicationInfo info, ClassLoader classLoader) {
    synchronized (this) {
        //调用SystemServer中创建出的ContextImpl的installSystemApplicationInfo函数
        getSystemContext().installSystemApplicationInfo(info, classLoader);      
        getSystemUiContext().installSystemApplicationInfo(info, classLoader);

        // give ourselves a default profiler
        mProfiler = new Profiler();
    }
}
```

继续跟进ContextImpl的installSystemApplicationInfo函数：

```java
void installSystemApplicationInfo(ApplicationInfo info, ClassLoader classLoader) {
    //前面已经提到过mPackageInfo的类型为LoadedApk
    mPackageInfo.installSystemApplicationInfo(info, classLoader);
}
```

mPackageInfo 是之前加载好持有的 loadedApk：

```java
void installSystemApplicationInfo(ApplicationInfo info, ClassLoader classLoader) {
    //这个接口仅供系统进程调用，故这里断言一下
    assert info.packageName.equals("android");

    mApplicationInfo = info;
    mClassLoader = classLoader;
}
```

至此，我们知道了installSystemApplicationInfo的真相就是： 

1. 将“android”对应的ApplicationInfo（即framework-res.apk对应的ApplicationInfo），
2. 加入到SystemServer之前调用createSystemContext时，创建出的LoadedApk中。

毕竟SystemServer创建System Context时，PKMS并没有完成对手机中文件的解析，初始的LoadedApk中并没有持有有效的ApplicationInfo。

### 3. AMS进程管理相关的操作。

由于framework-res.apk运行在SystemServer进程中，而AMS是专门用于进程管理和调度的， 
因此SystemServer进程也应该在AMS中有对应的管理结构。

于是，AMS的下一步工作就是将SystemServer的运行环境和一个进程管理结构对应起来，并进行统一的管理。