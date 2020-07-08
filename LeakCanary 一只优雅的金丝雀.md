LeakCanary 是 Square 公司的一个开源库。通过它可以在 App 运行过程中检测内存泄漏，当内存泄漏发生时会生成发生泄漏对象的引用链，并通知程序开发人员。

说一个锦上添花的小知识点：
17世纪，英国矿井工人发现，金丝雀对瓦斯这种气体十分敏感。空气中哪怕有极其微量的瓦斯，金丝雀也会停止歌唱；而当瓦斯含量超过一定限度时，虽然鲁钝的人类毫无察觉，金丝雀却早已毒发身亡。当时在采矿设备相对简陋的条件下，工人们每次下井都会带上一只金丝雀作为“瓦斯检测指标”，以便在危险状况下紧急撤离。

![640](img/640.jpeg)

LeakCanary 就是能敏感发现内存泄漏的金丝雀，来帮助我们尽量避免 OOM 的危险。这一理念也在它的 Logo 设计中体现：

![image-20200707233520753](img/image-20200707233520753.png)

## 如何初始化的

引入 LeakCanary 只需要在  app 的 build.gradle 文件增加以下代码
```
dependencies {
  // debugImplementation because LeakCanary should only run in debug builds.
  debugImplementation 'com.squareup.leakcanary:leakcanary-android:2.4'
}
```
**That’s it, there is no code change needed!**

为什么一行就能搞定呢。

```kotlin
class AppWatcherInstaller : ContentProvider() {
  override fun onCreate(): Boolean {
    val application = context!!.applicationContext as Application
    AppWatcher.manualInstall(application) // auto install 
    return true
  }
}
```
应用主进程启动时会自动初始化定义在 Manifest 里的 ContentProvider，也就能自动调用 `AppWatcher.manualInstall`。是不是觉得这样很优雅，每个第三方库都整个 ContentProvider 岂不是重复劳动？Jetpack 的 App Startup  了解一下，基于这个原理的封装。 https://developer.android.google.cn/topic/libraries/app-startup



那我就是想自己手动初始化（比如启动速度优化），要怎么处理？

答案就在定义这个 ContentProvider 的地方：

```xml
<provider
    android:name="leakcanary.internal.AppWatcherInstaller$MainProcess"
    android:authorities="com.sam.demo.leakcanary-installer"
    android:enabled="@bool/leak_canary_watcher_auto_install" // here
    android:exported="false" />

// 我们在 app 重写这个 bool 值就可以取消自动安装
<resources>
  <bool name="leak_canary_watcher_auto_install">false</bool>
</resources>
// 然后手动在合适的地方调用 AppWatcher.manualInstall 初始化
```

不过话又说回来，这个库是在 debug 环境下运行的，谁会闲着没事去优化 Debug 版本呢，净装逼。

emmm，那问题来了，我在 release 使用会怎样呢？做如下修改，在 Build Variants 选择 release 跑起来看看：

```groovy
dependencies {
  implementation 'com.squareup.leakcanary:leakcanary-android:2.4'
}
```

当当当，结果一打开 app 就报错了：

```verilog
? E: FATAL EXCEPTION: main
    Process: com.sam.demo, PID: 28838
    java.lang.Error: LeakCanary in non-debuggable build
    
    LeakCanary should only be used in debug builds, but this APK is not debuggable.
    Please follow the instructions on the "Getting started" page to only include LeakCanary in
    debug builds: https://square.github.io/leakcanary/getting_started/
    
    If you're sure you want to include LeakCanary in a non-debuggable build, follow the 
    instructions here: https://square.github.io/leakcanary/recipes/#leakcanary-in-release-builds
        at leakcanary.internal.InternalLeakCanary.checkRunningInDebuggableBuild(InternalLeakCanary.kt:160)
```

报错的地方的调用链

```kotlin
   AppWatcher.manualInstall(application) 
-> InternalAppWatcher.install(application)
-> nternalAppWatcher.onAppWatcherInstalled(application)
-> InternalLeakCanary.invoke()
-> InternalLeakCanary.checkRunningInDebuggableBuild()

  private fun checkRunningInDebuggableBuild() {
    if (isDebuggableBuild) {
      return
    }
    if (!application.resources.getBoolean(R.bool.leak_canary_allow_in_non_debuggable_build)) {
      throw Error(...)
    }
  }
```

所以如果你真的想作死在 release 也引入 leakCanary，只需要：

```XML
// 在 app 重写这个 bool 值
// 自己开发 SDK 时，这种配置方式学会了吗？
<resources>
  <bool name="leak_canary_allow_in_non_debuggable_build">true</bool>
</resources>
```

## 监听泄漏的时机

### Activity

没啥好说的，通过 registerActivityLifecycleCallbacks 监听 Activity 生命周期回调，在 onActivityDestroyed 时，`objectWatcher.watch(activity, ...)`

### Fragment、fragment.view

则是尝试从不同的 fragmentManager 加监听（emmm 策略模式）

- O（奥利奥）以上 -> activity.fragmentManager (AndroidOFragmentDestroyWatcher.kt)
- androidX -> activity.supportFragmentManager (AndroidXFragmentDestroyWatcher.kt)
- support 包 -> activity.supportFragmentManager (AndroidSupportFragmentDestroyWatcher.kt)

调用 fragmentManager.registerFragmentLifecycleCallbacks 监听。

而上面用到的 activity 也是通过 registerActivityLifecycleCallbacks 的 onActivityCreated 拿到的

```kotlin
AndroidOFragmentDestroyWatcher.kt

override fun onFragmentViewDestroyed(
  fm: FragmentManager,
  fragment: Fragment
) {
  val view = fragment.view
  // 观察 view 是否回收
  objectWatcher.watch(view,...) 
}

override fun onFragmentDestroyed(
  fm: FragmentManager,
  fragment: Fragment
) {
  // 观察 fragment 对象是否回收
  objectWatcher.watch(fragment,...) 
}
```

### ViewModel

在前面讲到的 AndroidXFragmentDestroyWatcher.kt 里，还会额外监听

```KOTLIN
 override fun onFragmentCreated(
   fm: FragmentManager,
   fragment: Fragment,
   savedInstanceState: Bundle?
 ) {
   ViewModelClearedWatcher.install(fragment, objectWatcher, configProvider)
 }

```

install 的具体实现是在这个 fragment 的 ViewModelProvider 取一个 `ViewModelClearedWatcher`。这也是一个 ViewModel , 在它被回收时会回调 onCleared 方法将所有 ViewModel 加入观察

```KOTLIN
init {
  // We could call ViewModelStore#keys with a package spy in androidx.lifecycle instead,
  // however that was added in 2.1.0 and we support AndroidX first stable release. viewmodel-2.0.0
  // does not have ViewModelStore#keys. All versions currently have the mMap field.
  // 通过反射获取以这个 fragment 为 onwer 所有的 viewModel
  viewModelMap = try {
    val mMapField = ViewModelStore::class.java.getDeclaredField("mMap")
    mMapField.isAccessible = true
    @Suppress("UNCHECKED_CAST")
    mMapField[storeOwner.viewModelStore] as Map<String, ViewModel>
  } 
}

override fun onCleared() {
  if (viewModelMap != null && configProvider().watchViewModels) {
    viewModelMap.values.forEach { viewModel ->
      objectWatcher.watch( viewModel, ... )
    }
  }
}
```

## 如何检测一个对象是否泄露

一个 JVM 的基础知识：

```JAVA
/**
 * <p> Suppose that the garbage collector determines at a certain point in time
 * that an object is <a href="package-summary.html#reachability">weakly
 * reachable</a>.  At that time it will atomically clear all weak references to
 * that object and all weak references to any other weakly-reachable objects
 * from which that object is reachable through a chain of strong and soft
 * references.  At the same time it will declare all of the formerly
 * weakly-reachable objects to be finalizable.  At the same time or at some
 * later time it will enqueue those newly-cleared weak references that are
 * registered with reference queues.
 */
public class WeakReference<T> extends Reference<T> {
    public WeakReference(T referent, ReferenceQueue<? super T> q) {
        super(referent, q);
    }
}
```

Java 中的 WeakReference 是弱引用类型，每当发生 GC 时，它所持有的对象如果没有被其他强引用所持有，那么它所引用的对象就会被回收，同时或者稍后的时间这个 WeakReference 会被入队到 ReferenceQueue. LeakCanary 中对内存泄露的检测正是基于这个原理。

实现要点：

- 当一个 Object 需要被回收时，对应生成一个 key ，封装到自定义的 KeyedWeakReference 中，并且在 KeyedWeakReference 的构造器中传入自定义的 ReferenceQueue。
- 同时将这个 KeyedWeakReference 缓存一份到 Map 中（ ObjectWatcher.watchedObjects ）
- 最后主动触发 GC，遍历自定义 ReferenceQueue 中所有的记录，并根据获取的 KeyedWeakReference 里 key 的值，移除 Map 中相应的项。

**经过上面 3 步之后，还保留在  Map 中的就是：应当被 GC 回收，但是实际还保留在内存中的对象，也就是发生泄漏了的对象。**

我们来看下具体的代码：

```kotlin
ObjectWatcher.kt
fun watch(
   watchedObject: Any,
   description: String
 ) {
   // 遍历 queue ，从 watchedObjects 移除相应项
   removeWeaklyReachableObjects() 
 
   val key = UUID.randomUUID().toString()
   val watchUptimeMillis = clock.uptimeMillis()
   val reference =
     KeyedWeakReference(watchedObject, key, description, watchUptimeMillis, queue)
   }

   watchedObjects[key] = reference
   checkRetainedExecutor.execute {
     // checkRetainedExecutor 通过 Handler.postDelayed 实现
     // 默认延迟 5s , 去保留的对象里查看一下这个 key 是否还在
     moveToRetained(key)
   }
 }

 fun moveToRetained(key: String) {
   removeWeaklyReachableObjects() // 再检查一遍是否已经回收
   val retainedRef = watchedObjects[key]
   if (retainedRef != null) {
     retainedRef.retainedUptimeMillis = clock.uptimeMillis()
     onObjectRetainedListeners.forEach { it.onObjectRetained() }
   }
 }
```

5S 后检查对象还在的话，调用 onObjectRetained 方法通知处理，调用到的是 InternalLeakCanary 的实现

```kotlin
override fun onObjectRetained() {
  if (this::heapDumpTrigger.isInitialized) {
    // 看名字就知道，这是触发 heap dump 相关的逻辑
    heapDumpTrigger.onObjectRetained()
  }
}
```

接着看 HeapDumpTrigger 里的相关调用：

```kotlin
   onObjectRetained()
-> scheduleRetainedObjectCheck(...)
-> checkRetainedObjects(reason)


ivate fun checkRetainedObjects(reason: String) {
val config = configProvider()

var retainedReferenceCount = objectWatcher.retainedObjectCount

if (retainedReferenceCount > 0) {
  // 执行一次 GC ，再来看还剩下多少对象未被回收
  // 小细节：GC 之后还 sleep(100) 等回收的引用入队
  gcTrigger.runGc() 
  retainedReferenceCount = objectWatcher.retainedObjectCount
}

// checkRetainedCount 以下两种情况 return true 不继续后面的流程
// 1. 若之前有显示有泄漏，且当前已经全部回收，显示无泄漏的通知  
// 2. 存留的对象超过 5 个（默认，可配）且 (app 可见或不可见超过 5s), 
//    延迟 2s 再进行检查（避免应用卡频繁卡） 
// 判断 app 是否可见代码：VisibilityTracker.kt
if (checkRetainedCount(retainedReferenceCount, config.retainedVisibleThreshold)) return

if (!config.dumpHeapWhenDebugging && DebuggerControl.isDebuggerAttached) {
  // 如果正在调试，可能影响检测结果，还是晚点再试吧
  scheduleRetainedObjectCheck(
        reason = "debugger is attached",
        rescheduling = true,
        delayMillis = WAIT_FOR_DEBUG_MILLIS
    )
    return
  }

  val now = SystemClock.uptimeMillis()
  val elapsedSinceLastDumpMillis = now - lastHeapDumpUptimeMillis
  if (elapsedSinceLastDumpMillis < WAIT_BETWEEN_HEAP_DUMPS_MILLIS) {
    // 一分钟之内才 dump 过，等离上次 dump 1分钟了再来吧
    scheduleRetainedObjectCheck(
        reason = "previous heap dump was ${xx}ms ago (< ${xx}ms)",
        rescheduling = true,
        delayMillis = WAIT_BETWEEN_HEAP_DUMPS_MILLIS - elapsedSinceLastDumpMillis
    )
    return
  }
  
  // 终极操作
  // 通过 Debug.dumpHprofData(filePath)  dump heap
  // objectWatcher.clearObjectsWatchedBefore(heapDumpUptimeMillis) 清除这次 dump 开始以前的所有引用
  // HeapAnalyzerService.runAnalysis 通过一个 IntentService 去分析 heap
  dumpHeap(retainedReferenceCount, retry = true) 
}
```

HeapAnalyzerService 里调用的是 Shark 库对 heap 进行分析，分析的结果再返回到 DefaultOnHeapAnalyzedListener.onHeapAnalyzed 进行分析结果入库、发送通知消息。

Shark 🦈 ：Shark is the heap analyzer that powers LeakCanary 2. It's a Kotlin standalone heap analysis library that runs at **high speed** with a **low memory footprint**.

还记得比较早的版本，用的是 Square 公司出品叫 **haha** 库，库写这么好就算了，起名字都这么会起，真是厉害了。

## 彩蛋环节

鲨鱼咬着金丝雀，爱雀人士表示强烈谴责（手动狗头）

```xml
                   ^`.                 .=""=.
   ^_              \  \               / _  _ \
   \ \             {   \             |  d  b  |
   {  \           /     `~~~--__     \   /\   /
   {   \___----~~'              `~~-_/'-=\/=-'\,
    \                         /// a  `~.      \ \
    / /~~~~-, ,__.    ,      ///  __,,,,)      \ |
    \/      \/    `~~~;   ,---~~-_`/ \        / \/
                     /   /            '.    .'
                    '._.'             _|`~~`|_
                                      /|\  /|\
```

又有彩蛋，我都爱上看源码了，出自：https://github.com/square/leakcanary/blob/main/docs/shark.md

## 总结

本文分析了 LeakCanary 的初始化、添加对象监听的时机、内存泄漏检测的流程及原理。我们发现这个库不仅好用，而且实现也很巧妙。阅读这样的源码，不但能应对面试，也有助于在平常工作中写一手漂亮的代码。

本文基于：**leakcanary-android:2.4**

