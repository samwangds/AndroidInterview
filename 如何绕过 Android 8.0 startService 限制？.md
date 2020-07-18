>应用在后台运行时，会消耗一部分有限的设备资源，例如 RAM。 这可能会影响用户体验，如果用户正在使用占用大量资源的应用（例如玩游戏或观看视频），影响会尤为明显。 为了提升用户体验，Android 8.0（API 级别 26）对应用在后台运行时可以执行的操作施加了限制。

##  限制了什么？

*1. 后台应用对后台服务的访问受到限制*

在不与用户直接交互的后台应用中，运行 Service 会消耗系统资源，这可能会影响前台应用的正常运行。Android 8.0 及更高版本**不允许后台应用运行后台服务**，需要通过 startForegroundService() 指定为前台服务运行，或者使用 JobScheduler 替代。

*2. 注册隐式广播接收器受到限制*

对于一些系统隐式广播（非全部），系统不允许应用在 AndroidManifest 中注册对应的广播接收器，从而避免系统广播导致诸多应用快速、连续消耗系统资源，从而影响用户体验，需要通过 Context.registerReceiver() 动态注册或 JobScheduler 代替。

*3. 降低了后台应用接收位置更新的频率*

为节约电池电量、保持良好的用户体验和确保系统健康运行，在运行 Android 8.0 的设备上使用后台应用时，降低了后台应用接收位置更新的频率。

## 什么是前台应用?

系统可以区分前台和后台应用。如果满足以下任意条件，应用将被视为处于前台：

1. 具有可见 Activity
2. 具有前台 Service
3. 另一个前台应用已关联到该应用（绑定 Service 或使用 content providers）

如果以上条件均不满足，应用将被视为处于后台。

## 正确理解后台服务限制

**不允许后台应用运行后台服务**

官网的这句描述很简单，但你真的明白它的含义吗？顺着这句话推导一下：

后台应用无法启动后台服务

-> 前台应用可以启动后台服务

-> *A 为前台应用，则 A 就能启动后台服务*

基于这个结论，再结合后台服务的种类，对以下三种场景实践验证，结果如下：

1. 若后台服务属于 A 应用进程，则能正常启动
2. 若后台服务属于 B 应用进程，且 B 是前台应用，则能正常启动
3. 若后台服务属于 B 应用进程，且 B 是后台应用，则*无法启动！*

通过第三种场景的验证结果，可以知道 *不允许后台应用运行后台服务* 这个描述是不准确、有歧义的，更精准的描述应该是：

**不允许启动属于后台应用的后台服务**

## 后台服务限制源码分析

若在 Android 8.0 设备上通过 startService 启动一个属于后台应用的后台服务，会直接崩溃：

```java
Caused by: java.lang.IllegalStateException: Not allowed to start service Intent
        { act=intent.action.ServerService pkg=com.server }: app is in background uid null
   at android.app.ContextImpl.startServiceCommon(ContextImpl.java:1577)
   at android.app.ContextImpl.startService(ContextImpl.java:1532)
   at android.content.ContextWrapper.startService(ContextWrapper.java:664)
   ...
```

下面以此异常为线索，一步一步来看源码中是如何限制的。异常在 ContextImpl 中抛出：

```java
private ComponentName startServiceCommon(Intent service, boolean requireForeground,
        UserHandle user) {
    ...
    ComponentName cn = ActivityManager.getService().startService(
        mMainThread.getApplicationThread(), service, service.resolveTypeIfNeeded(
                    getContentResolver()), requireForeground,
                    getOpPackageName(), user.getIdentifier());
    if (cn != null) {
        if (cn.getPackageName().equals("!")) {
            ...
        } else if (cn.getPackageName().equals("?")) {
            //这里抛出启动服务限制的异常
            throw new IllegalStateException(
                    "Not allowed to start service " + service + ": " + cn.getClassName());
        }
    }
    return cn;
}
```

可见关键点是 cn.getPackageName().equals("?") 条件成立， 继续看 AMS startService 方法中是如何返回这个 ComponentName 的：

```java
public ComponentName startService(IApplicationThread caller, Intent service,
        String resolvedType, boolean requireForeground, String callingPackage, int userId)
        throws TransactionTooLargeException {
    ...
    ComponentName res;
    try {
        res = mServices.startServiceLocked(caller, service,
                resolvedType, callingPid, callingUid,
                requireForeground, callingPackage, userId);
    } finally {
        Binder.restoreCallingIdentity(origId);
    }
    return res;
}
```

AMS 中转调 ActiveServices 的 startServiceLocked 方法去处理服务的启动，关键代码如下：
```java
ComponentName startServiceLocked(IApplicationThread caller, Intent service, String resolvedType,
        int callingPid, int callingUid, boolean fgRequired, String callingPackage,
        final int userId, boolean allowBackgroundActivityStarts)
        throws TransactionTooLargeException {
    ...
    //非前台服务，需要检测是否满足后台服务启动条件，不满足则限制启动
    if (forcedStandby || (!r.startRequested && !fgRequired)) {
        final int allowed = mAm.getAppStartModeLocked(r.appInfo.uid, r.packageName,
                r.appInfo.targetSdkVersion, callingPid, false, false, forcedStandby);
        if (allowed != ActivityManager.APP_START_MODE_NORMAL) { //不满足启动条件
            if (allowed == ActivityManager.APP_START_MODE_DELAYED || forceSilentAbort) {
                //这里返回 null 代表此场景下静默的限制启动，不通知应用
                return null;
            }
            ...
            //这里代表用户应知道此场景下不允许启动，所以返回 ComponentName，明确的通知应用
            //注意返回了 "?"，是导致应用崩溃的原因
            UidRecord uidRec = mAm.mProcessList.getUidRecordLocked(r.appInfo.uid);
            return new ComponentName("?", "app is in background uid " + uidRec);
        }
    }
}
```

至此可以知道关键在于 mAm.getAppStartModeLocked 方法，如果返回 APP_START_MODE_NORMAL 则代表满足启动条件，不会被限制。

mAm 为 ActivityManagerService，继续看 ActivityManagerService 的 getAppStartModeLocked 方法：

```java
int getAppStartModeLocked(int uid, String packageName, int packageTargetSdk,
        int callingPid, boolean alwaysRestrict, boolean disabledOnly, boolean forcedStandby) {
    UidRecord uidRec = mProcessList.getUidRecordLocked(uid);
    //注意传入的 alwaysRestrict、forcedStandby 都为 false，暂不关注
    if (uidRec == null || alwaysRestrict || forcedStandby || uidRec.idle) {
        ...
        final int startMode = (alwaysRestrict)
                ? appRestrictedInBackgroundLocked(uid, packageName, packageTargetSdk)
                : appServicesRestrictedInBackgroundLocked(uid, packageName,
                        packageTargetSdk);
        return startMode;
    }
    return ActivityManager.APP_START_MODE_NORMAL;
}
```

uidRec 为该服务所在应用，uidRec == null 代表应用还未启动，uidRec.idle 代表应用处于后台。应用未启动可以看作处于后台，当然也是不允许启动后台服务的。

继续看 appServicesRestrictedInBackgroundLocked 方法：

```java
int appServicesRestrictedInBackgroundLocked(int uid, String packageName, int packageTargetSdk) {
    if (mPackageManagerInt.isPackagePersistent(packageName)) {
        //系统永久应用不做限制
        return ActivityManager.APP_START_MODE_NORMAL;
    }
    if (uidOnBackgroundWhitelist(uid)) {
        //白名单应用不做限制
        return ActivityManager.APP_START_MODE_NORMAL;
    }
    if (isOnDeviceIdleWhitelistLocked(uid, false)) {
        //白名单应用不做限制
        return ActivityManager.APP_START_MODE_NORMAL;
    }
    // 其他应用走通用的限制策略
    return appRestrictedInBackgroundLocked(uid, packageName, packageTargetSdk);
}
```

普通应用会走通用的限制策略，继续看 appRestrictedInBackgroundLocked 方法：

```java
int appRestrictedInBackgroundLocked(int uid, String packageName, int packageTargetSdk) {
    if (packageTargetSdk >= Build.VERSION_CODES.O) {
        return ActivityManager.APP_START_MODE_DELAYED_RIGID;
    }
    //tartget API 小于 8.0 的应用走旧的限制策略，众所周知的不会被限制
    ...
}
```
可以看到这里对 tartget API 做的限制，8.0 及以上的应用会被限制启动服务，是上层抛出异常的根本原因。

## 适配 Android 8.0 startService 限制策略

了解了系统的限制原理后，结合上文对 AMS 启动服务限制的源码分析，列举可能的适配方案：

1. 使用 startForegroundService 代替
2. 使用 JobScheduler 代替
3. 设置应用为 Persisten 系统永久应用类型
4. 将应用加入到系统白名单
5. 将应用的 targetSdkVersion 调整为小于 Android 8.0 的版本
6. 启动服务前，先将服务所在应用从后台切换到前台

方案 1 是工作量较小的兼容旧代码方案，但会显示一条通知，这可能不是我们想要的

方案 2 是官方建议方案，兼容工作量比方案 1 多

方案 3 和方案 4 需要系统侧配合，适用于系统或预装应用，对绝大多数的第三方应用来说不可行

方案 5 可行，但极不推荐这种固步自封的方式

方案 6 可行，但不符合谷歌推进此限制策略的意愿，违背提高用户体验的初衷

## 如何绕过 Android 8.0 startService 限制？

别忘了标题，最终想要实现的是绕过 Android 8.0 startService 的限制，即不修改为前台服务，调用 startService 方法，仍旧可以启动属于后台应用的后台服务，怎么实现呢？

通过上面的方案 6 ：**启动服务前，先将服务所在应用从后台切换到前台** 便可实现，如何将应用从后台切换到前台呢？上文介绍了应用被视为处于前台的条件：

1. 具有可见 Activity
2. 具有前台 Service
3. 另一个前台应用已关联到该应用

依据条件 1 可想到一种实现方案：

>如果应用处于后台，就启动一个透明的、用户无感知的 Activity，将应用切换到前台，然后再通过 startService 启动服务，随后 finish 掉透明 Activity。

调用端这样 startService ：

```java
Intent serviceIntent = new Intent();
serviceIntent.setAction("com.ahab.server.service");
serviceIntent.setPackage("com.ahab.server");
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
    try{
        context.startService(serviceIntent);
    }catch (Exception e){
        Intent activityIntent = new Intent();
        activityIntent.setAction("com.ahab.server.TranslucentActivity");
        activityIntent.setPackage("com.ahab.server");
        activityIntent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
        context.startActivity(activityIntent);
    }
}else{
    context.startService(serviceIntent);
}
```

启动透明 Activity 后 startService：

```java
public class TranslucentActivity extends Activity {

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        Intent serviceIntent = new Intent();
        serviceIntent.setAction("com.ahab.server.service");
        serviceIntent.setPackage("com.ahab.server");
        context.startService(serviceIntent);
        finish();
    }
}
```

可以看到代码实现十分简单。上面都是围绕 startService 方式来讲，没有提及 **bindService** 服务启动方式，系统并未直接对 bindService 限制服务启动。
