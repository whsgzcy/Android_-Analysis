## 剖析ActivityManagerService
**参考**[here](https://juejin.im/post/5a055ab45188252ae93a6932).

我们开发Android应用时，少不了要看很多log，无论是借助工具查看，还是adb shell logcat > /*/t.log，总会看到非Application的log，每一行的log都是有他的意义，如果我能够熟悉Android从启动到应用，每一行打印的log，我想，我深入的不仅仅是在Android这一方面，其中解决问题的能力会在各个方面都适用；

ActivityManagerService 简称AMS，本文主要围绕三个方面来阐述AMS如何管理Activity：

|   | 剖析ActivityManagerService  |  |
|:------------- |:---------------:| -------------:|
| AMS启动流程      | Activity Stack介绍 |  Activity Task介绍 |

先基本了解一下 AMS是什么？

* AMS是Android系统的一个进程

* 用于管理系统四大组件的运行状态

### AMS的启动流程？
AMS的初始化，API 19和API 25源码进行对比

在API 19源码中，随着SystemServer类main方法的调用，实例化内部类ServerThread的对象：

```
代码1.1 (来源:  SystemServer.java):

public static void main(String[] args) {
    ......(省略了一小戳代码)
    ServerThread thr = new ServerThread();
    thr.initAndLoop();
}
```

继续看内部类ServerThread的initAndLoop方法，这个方法有1000+行，着实比较恶心

```
代码1.2 (来源:  SystemServer.java):

public void initAndLoop() {
    ......
    context = ActivityManagerService.main(factoryTest);
    ......
    ActivityManagerService.setSystemProcess();
}
```
该方法是非常强大的，诸多系统服务都在此初始化，如AMS、PowerManagerService、WindowManagerService、NetworkManagementService等等，
调用ActivityManagerService的静态方法main()，即可得到AMS的上下文

```
代码1.3 (来源:  ActivityManagerService.java):

public static final Context main(int factoryTest) {
    // 创建了一个AThread线程，并开启
    AThread thr = new AThread();
    thr.start();

    synchronized (thr) {
        while (thr.mService == null) {
            try {
                thr.wait();
            } catch (InterruptedException e) {
            }
        }
    }

    ActivityManagerService m = thr.mService;
    mSelf = m;
    ActivityThread at = ActivityThread.systemMain();
    mSystemThread = at;
    Context context = at.getSystemContext();

    m.mContext = context;
    ......

    return context;
}
```

这段代码，虽然开启了Athread线程，但是他立马进入了等待状态，为什么？

因为下面要使用到AMS对象，如果AMS的对象还未初始化，我们贸然使用，肯定会导致系统宕机，所以，Athread中肯定对AMS进行了实例化，那等待的线程如何去唤醒他？

```
代码1.4 (来源:  ActivityManagerService.java):

static class AThread extends Thread {
    ActivityManagerService mService;

        public AThread() {
            super("ActivityManager");
        }
        @Override
        public void run() {
           ......
            // 实例化了AMS对象
            ActivityManagerService m = new ActivityManagerService();

            synchronized (this) {
                mService = m;
                notifyAll();
            }
         ......
        }
    }
```

以上的代码中。实例化了AMS对象并复制给mService，notifyAll()的职责就是对等待的AThread线程进行唤醒，此时即可跳出1.3中的while循环，而后返回context，

然后调用ActivityManagerService.setSystemProcess();即可向Server Manager注册，到此AMS整个启动流程到此结束。

刚我们一起了解下载API 19的AMS启动流程，那和API 25的源码相比较，有何改变？

同样是SystemServer类的main()方法，但是一行简单的代码了事

```
代码1.5 (来源:  SystemServer.java):

public static void main(String[] args) {
   new SystemServer().run();
}
```
```
代码1.6 (来源:  SystemServer.java):

private void run() {
   ......
   // 在这里启动各种系统级服务
   startBootstrapServices();
   startCoreServices();
   startOtherServices();
   ......
}
```

```
代码1.7 (来源:  SystemServer.java):

private void startBootstrapServices() {
    ......
    // 这里直接通过SystemServiceManager直接开启AMS
    mActivityManagerService = mSystemServiceManager.startService(
    ActivityManagerService.Lifecycle.class).getService();

    mActivityManagerService.setSystemServiceManager(mSystemServiceManager);
    mActivityManagerService.setInstaller(installer);
    ......
}
```

```
代码1.8 (来源:  ActivityManagerService.java):

public static final class Lifecycle extends SystemService {
        private final ActivityManagerService mService;

        public Lifecycle(Context context) {
            super(context);
            // 得到了AMS的实例
            mService = new ActivityManagerService(context);
        }

        @Override
        public void onStart() {
            mService.start();
        }

        public ActivityManagerService getService() {
            return mService;
        }
}
```

API 25的源码相比较API 19来说，通俗易懂；

### Activity Stack 介绍
Stack意为堆栈，作为Activity的堆栈，主要还是由以下三大部分组成

#### Activity State
该枚举定义的各种状态和Activity的生命周期有着千丝万缕的关系

```
代码2.1 (来源:  ActivityStack.java):

enum ActivityState {
     INITIALIZING,    // 初始化中
     RESUMED,         // 恢复
     PAUSING,         // 暂停中
     PAUSED,          // 已暂停
     STOPPING,        // 停止中
     STOPPED,         // 已停止
     FINISHING,       // 完成中
     DESTROYING,      // 销毁中
     DESTROYED        // 已销毁
}
```

#### ArrayList

在Activity Stack中定义了一些ArrayList，用来保存特定状态的Activity，比如：

```
代码2.2 (来源:  ActivityStack.java):

ArrayList<TaskRecord> mTaskHistory；
ArrayList<TaskGroup> mValidateAppTokens；
ArrayList<ActivityRecord> mLRUActivities；
ArrayList<ActivityRecord> mNoAnimActivities；
ArrayList<ActivityRecord> mStoppingActivities;
ArrayList<ActivityRecord> mFinishingActivities;
......
```

#### 记录在特殊状态下的Activity

```
代码2.3 (来源:  ActivityStack.java):

ActivityRecord mPausingActivity = null;
ActivityRecord mLastPausedActivity = null;
ActivityRecord mLastNoHistoryActivity = null;
ActivityRecord mResumedActivity = null;
ActivityRecord mLastStartedActivity = null;
......
```

正因为有了这三大部分，AMS即可通过Activity Stack实现了对系统组件的记录，管理以及查询功能；

adb shell dumpsys activity

可查看Activity Stack的信息



