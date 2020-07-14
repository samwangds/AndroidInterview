

今天我们一起来看五道 Activity 生命周期的面试题，相信看完之后面试官再问到相关的问题，你就能胸有成竹了。

## A Activity 打开 B Activity 时都有哪些生命周期回调。

这道题相信很多同学都有遇到过，简单 A.onPause -> B.onCreate -> B.onStart -> B.onResume -> A.onStop .
Naive ! 这样回答只是及格，因为仅在 B Activity 的 launchMode 为 standard 或者 B Activity 没有可复用的实例时是这样的。

当 B Activity 的  launchMode 为 singleTop 且 B Activity 已经在栈顶时（一些特殊情况如通知栏点击、连点），此时只有 B 页面自己有生命周期变化:
B.onPause -> B.onNewIntent -> B.onResume

当 B Activity 的  launchMode 为 singleInstance ，singleTask 且对应的 B Activity 有可复用的实例时，生命周期回调是这样的:
 A.onPause -> B.onNewIntent -> B.onRestart -> B.onStart -> B.onResume -> A.onStop -> ( 如果 A 被移出栈的话还有一个 A.onDestory)

把几种情况都回答出来就能加分啦，同时也要做好聊 launchMode 的准备。

## 弹出 Dialog 对生命周期有什么影响

我们知道，生命周期回调都是 AMS 通过 Binder 通知应用进程调用的；而弹出 Dialog、Toast、PopupWindow 本质上都直接是通过 WindowManager.addView() 显示的（没有经过 AMS），所以不会对生命周期有任何影响。

如果是启动一个 Theme 为 Dialog 的 Activity , 则生命周期为：
A.onPause -> B.onCreate -> B.onStart -> B.onResume
注意这边没有前一个 Activity 不会回调 onStop，因为只有在 Activity 切到后台不可见才会回调 onStop；而弹出 Dialog 主题的 Activity 时前一个页面还是可见的，只是失去了焦点而已所以仅有 onPause 回调。

## Activity 在 onResume 之后才显示的原因是什么

虽然我们设置 Activity 的布局一般都是在 onCreate 方法里调用 setContentView 。里面是直接调用 window 的 setContentView，创建一个 DecorView 用来包住我们创建的布局。详情如下：


```java
PhoneWindow.java
public void setContentView(int layoutResID) {
    if (mContentParent == null) {
        installDecor();
    } 
    ...
    // 加载布局，添加到 mContentParent
    // mContentParent 又是 DecorView 的一个子布局  
    mLayoutInflater.inflate(layoutResID, mContentParent);
}
```

然而这一步只是加载好了布局，生成一个 ViewTree , 具体怎么把 ViewTree 显示出来，答案就在下面：

```java
ActivityThread.java
public void handleResumeActivity(...){
    // onResume 回调
    ActivityClientRecord r = performResumeActivity(...)
    final Activity a = r.activity;
    if (r.window == null && !a.mFinished && willBeVisible) {
        r.window = r.activity.getWindow();
        View decor = r.window.getDecorView();
        ViewManager wm = a.getWindowManager();
        wm.addView(decor, l);// 重点
    }
}
```

  WindowManager 的 addView 方法最终将 DecorView 添加到 WMS ，实现绘制到屏幕、接收触屏事件。具体的调用链如下：

```java
   WindowManagerImpl.addView
-> WindowManagerGlobal.addView
-> ViewRootImpl.setView     
-> ViewRootImpl.requestLayout() // 执行 View 的绘制流程
   // 通过 Binder 调用 WMS ，WMS 会添加一个 Window 相关的对象
   // 应用端通过 mWindowSession 调用 WMS
   // WMS 通过 mWindow (一个 Binder 对象) 调用应用端  
   mWindowSession.addToDisplay(mWindow) 
```

综上，在 onResume 回调之后，会创建一个 ViewRootImpl ，有了它之后应用端就可以和 WMS 进行双向调用了。同时也是通过 ViewRootImpl 从 WMS 申请 Surface 来绘制 ViewTree 。

## onActivityResult 在哪两个生命周期之间回调

onActivityResult 不属于 Activity 的生命周期，一般被问到这个问题时大家都会懵逼。其实答案很简单，onActivityResult 方法的注释中就写着答案：**You will receive this call immediately before onResume() when your activity is re-starting.**  跟一下代码（TransactionExecutor.execute 有兴趣的可以自己打断点跟一下），会发现 onActivityResult 回调先于该 Activity 的所有生命周期回调，从 B Activity 返回 A Activity 的生命周期调用为：
`B.onPause -> A.onActivityResult -> A.onRestart -> A.onStart -> A.onResume`

## onCreate 方法里写死循环会 ANR 吗

ANR 的四种场景：

1. Service TimeOut:  service 未在规定时间执行完成： 前台服务 20s，后台 200s
2. BroadCastQueue TimeOut: 未在规定时间内未处理完广播：前台广播 10s 内, 后台 60s 内
3. ContentProvider TimeOut:  publish 在 10s 内没有完成
4. Input Dispatching timeout:  5s 内未响应键盘输入、触摸屏幕等事件

我们可以看到，Activity 的生命周期回调的阻塞并不在触发 ANR 的场景里面，所以并不会直接触发 ANR。只不过死循环阻塞了主线程，如果系统再有上述的四种事件发生，就无法在相应的时间内处理从而触发 ANR。



## 总结 

Activity 的生命周期很基础而且也很重要，这也是面试常问的原因。相关的面试题可以涉及到 framework 的一些知识，平常在处理一些问题的时候最好不要只是打下日志看下结果，多钻进去源码看看，才能有更多收获，也记得更牢。