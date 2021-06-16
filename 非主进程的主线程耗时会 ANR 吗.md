> 思考一个问题，一个 Service 运行在独立的进程里，在这个 Service 的 onCreate 方法里执行耗时操作会造成 ANR 吗？

## 直接说结论

会，但是不会有 ANR 的弹窗。好了赶时间、或者背答案的选手可以退出去了。

## 基础知识

#### ANR 的四种场景：

1. Service TimeOut:  service 未在规定时间执行完成： 前台服务 20s，后台 200s
2. BroadCastQueue TimeOut: 未在规定时间内未处理完广播：前台广播 10s 内, 后台 60s 内
3. ContentProvider TimeOut:  publish 在 10s 内没有完成
4. Input Dispatching timeout:  5s 内未响应键盘输入、触摸屏幕等事件

 ANR 的根本原因是：应用未在规定的时间内处理 AMS 指定的任务才会 ANR。

所以 Service 未在指定时间内执行完成而且非主进程的 Service 仍然需要通过 AMS 进行通信。这也能说明一定会产生 ANR 的。

## 实验

理论上是会，那我们就写个小 demo 来试一下。

写一个 Service：

```kotlin
class NoZuoNoDieService: Service() {
    override fun onBind(intent: Intent?): IBinder? {
         return null
    }

    override fun onCreate() {
        super.onCreate()
        // 小睡会模拟耗时
        Thread.sleep(21_000)
    }
}
```

然后在 AndroidManifest 里声明在独立进程

```XML
<service
    android:name=".service.NoZuoNoDieService"
    android:process=":whatever"
    />
```

最后我们把这个 Service 调起来作死

```kotlin
startService(Intent(this, NoZuoNoDieService::class.java))
```

应用并没有 ANR 弹窗，不过 logcat 有 ANR 相关信息，看一下 trace 文件：

```log
----- pid 14608 at 2021-03-23 19:56:20 -----
Cmd line: demo.com.sam.demofactory:whatever
... 省略无关信息
"main" prio=5 tid=1 Sleeping
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x73f13a80 self=0x78e7cc2a00
  | sysTid=14608 nice=0 cgrp=default sched=0/0 handle=0x796c96d9a8
  | state=S schedstat=( 57816925 3067496 80 ) utm=2 stm=3 core=4 HZ=100
  | stack=0x7fe1d03000-0x7fe1d05000 stackSize=8MB
  | held mutexes=
  at java.lang.Thread.sleep(Native method)
  - sleeping on <0x0a71f3e8> (a java.lang.Object)
  at java.lang.Thread.sleep(Thread.java:373)
  - locked <0x0a71f3e8> (a java.lang.Object)
  at java.lang.Thread.sleep(Thread.java:314)
  at demo.com.sam.demofactory.service.NoZuoNoDieService.onCreate(NoZuoNoDieService.kt:15)
  at android.app.ActivityThread.handleCreateService(ActivityThread.java:3426)
  at android.app.ActivityThread.-wrap4(ActivityThread.java:-1)
  at android.app.ActivityThread$H.handleMessage(ActivityThread.java:1744)
  at android.os.Handler.dispatchMessage(Handler.java:106)
  at android.os.Looper.loop(Looper.java:171)
  at android.app.ActivityThread.main(ActivityThread.java:6611)
  at java.lang.reflect.Method.invoke(Native method)
  at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:438)
  at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:807)
```

## 没有弹窗的 ANR

首先来复习一下 [Service 的启动流程](Service 启动流程.md)

![](img/Service 创建流程图.jpg)

AMS 真正去启动 Service 调用的是 ActiveServices.realStartServiceLocked 方法：

```JAVA
// API 29 com.android.server.am.ActiveServices 
private final void realStartServiceLocked(ServiceRecord r,
            ProcessRecord app, boolean execInFg) throws RemoteException {
  ...
  bumpServiceExecutingLocked(r, execInFg, "create");
}

private final void bumpServiceExecutingLocked(ServiceRecord r, boolean fg, String why) {
  ...
  scheduleServiceTimeoutLocked(r.app);  
}

// 通过 Handler 延时发一条消息，延时时间则为 Service 触发的 ANR 时长
// SERVICE_TIMEOUT = 20*1000
// SERVICE_BACKGROUND_TIMEOUT = SERVICE_TIMEOUT * 10;
void scheduleServiceTimeoutLocked(ProcessRecord proc) {
    if (proc.executingServices.size() == 0 || proc.thread == null) {
        return;
    }
    Message msg = mAm.mHandler.obtainMessage(
            ActivityManagerService.SERVICE_TIMEOUT_MSG);
    msg.obj = proc;
    mAm.mHandler.sendMessageDelayed(msg,
            proc.execServicesFg ? SERVICE_TIMEOUT : SERVICE_BACKGROUND_TIMEOUT);
}
```

如果 Service 在规定时间内启动完成，则这个消息会被 remove 掉，我们今天要看一下超时之后，收到这个消息是怎么处理的。

```JAVA
// com.android.server.am.ActivityManagerService.MainHandler
final class MainHandler extends Handler {
  @Override
  public void handleMessage(Message msg) {
       switch (msg.what) {
           case SERVICE_TIMEOUT_MSG: {
                mServices.serviceTimeout((ProcessRecord)msg.obj);
            } break;
       }
}
```

mServices 是 ActiveServices：

```JAVA
// com.android.server.am.ActiveServices#serviceTimeout
void serviceTimeout(ProcessRecord proc) {
  ...
  if (timeout != null && mAm.mProcessList.mLruProcesses.contains(proc)) {
    Slog.w(TAG, "Timeout executing service: " + timeout);
    ...
    mAm.mHandler.removeCallbacks(mLastAnrDumpClearer);
    mAm.mHandler.postDelayed(mLastAnrDumpClearer, LAST_ANR_LIFETIME_DURATION_MSECS);
    anrMessage = "executing service " + timeout.shortInstanceName;
  }   
  ...
  if (anrMessage != null) {
    proc.appNotResponding(null, null, null, null, false, anrMessage);
  }
}
```
appNotResponding 就是最终处理 ANR 逻辑的代码了
```JAVA
// com.android.server.am.ProcessRecord
void appNotResponding(String activityShortComponentName,
                      ApplicationInfo aInfo,
                      ...) {
	...
  if (isMonitorCpuUsage()) {
  	mService.updateCpuStatsNow();
	}
  ...
  if (isSilentAnr() && !isDebugging()) {
      kill("bg anr", true);
      return;
  }   
  
   // Bring up the infamous App Not Responding dialog
   Message msg = Message.obtain();
   msg.what = ActivityManagerService.SHOW_NOT_RESPONDING_UI_MSG;
   msg.obj = new AppNotRespondingDialog.Data(this, aInfo, aboveSystem);

   mService.mUiHandler.sendMessage(msg);
}
```

如果 isSilentAnr() 为 true，则直接杀死所在进程，不会发送显示 ANR 弹窗的消息。来看下相关方法

```JAVA
boolean isSilentAnr() {
    return !getShowBackground() && !isInterestingForBackgroundTraces();
}
private boolean getShowBackground() {
    return Settings.Secure.getInt(mService.mContext.getContentResolver(),
            Settings.Secure.ANR_SHOW_BACKGROUND, 0) != 0;
}
private boolean isInterestingForBackgroundTraces() {
    // The system_server is always considered interesting.
    if (pid == MY_PID) {
        return true;
    }

    // A package is considered interesting if any of the following is true :
    //
    // - It's displaying an activity.
    // - It's the SystemUI.
    // - It has an overlay or a top UI visible.
    //
    // NOTE: The check whether a given ProcessRecord belongs to the systemui
    // process is a bit of a kludge, but the same pattern seems repeated at
    // several places in the system server.
    return isInterestingToUserLocked() ||
            (info != null && "com.android.systemui".equals(info.packageName))
            || (hasTopUi() || hasOverlayUi());
}
```

我们得到两个结论：

- ANR_SHOW_BACKGROUND 可以在**开发者选项**里配置显示所有“应用程序无响应”，开启后即使是该场景也会有 ANR 弹窗
- 未开启上述配置情况下，如果是后台进程，则不显示 ANR 弹窗

## 小结

会有这个问题，是因为项目在新版本上线时，真的发生了这个问题。有时候一念之间，以为是非主进程就可以胡作非为。最后收获了一个 TOP 1 的 ANR 问题。虽然对用户产生的影响不大，但是对技术的侮辱性极强。
