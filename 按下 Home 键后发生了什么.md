后台有位小伙伴分享了一个头条的面试题：按下手机的 Home 键，有哪些动作和事件发生？

今天我们就来分析一下，本文源码基于 Android - 28

## 事件的分类

安卓系统中的事件，主要有以下几种：

* 按键事件（KeyEvent） 
  由物理按键产生的事件，如：Home, Back, Volume Up, Volume Down, Camera 等。今天主要分析的就是这类事件。
* 触摸事件（TouchEvent）
  在屏幕上点击拖动，以及由它们组合的各种事件。
* 鼠标事件（MouseEvent）
  鼠标操作产生的事件
* 轨迹球事件 （TrackBallEvent）
  知道轨迹球的，怕不是要暴露年龄

安卓针对上面这些事件共性，提取了一个统一的抽象类 InputEvent 。InputEvent 提供了几个常用的抽象方法，比如 getDevice() 获得当前事件的“硬件源”，getEventTime() 获取事件发生的时间。

InputEvent 有两个子类：

- KeyEvent 用于描述按键事件
- MotionEvent 用来描述 Movement 类型的事件(通过 mouse, pen, finger, trackball 产生)。

而我们要监听这些事件一般也是通过对 View 设置相应的监听实现

```JAVA
setOnKeyListener(OnKeyListener)
setOnTouchListener(OnTouchListener)
...
```

或者也可以直接复写相关的方法

```JAVA
boolean onKeyDown(int keyCode, KeyEvent event)
boolean onTouchEvent(MotionEvent event)
....
```

## 事件处理的准备工作

> 事件处理设计的整体思路是驱动层会有一个消息队列来存放事件，会有一个 Reader 来不停的读取事件，一个 Dispatcher 来分发消息队列中的事件。Dispatcher 分发的事件最后会通过 jni 上报到 InputManagerService，然后通过接口最后传递给PhoneWindow，PhoneWindow 再根据不同的事件类型来做不同的处理。

我们先看一下 Reader、Dispatcher 是怎么来的。

SystemServer 在 startOtherServices() 方法中启动 InputMangerService

```java
InputManagerService inputManager = new InputManagerService(context);
inputManager.setWindowManagerCallbacks(wm.getInputMonitor());
inputManager.start();
```

接下来我们看一下  InputMangerService 的构造方法

```JAVA
public InputManagerService(Context context) {
    this.mHandler = new InputManagerHandler(DisplayThread.get().getLooper()); 
    mPtr = nativeInit(this, mContext, mHandler.getLooper().getQueue());
    ...
}
```

主要是调用 nativeInit 方法传入一个消息队列，nativeInit 方法是为了去初始化一些 native 对象。最终是为了 new 一个 native 层的 InputManager 。 调用链如（想要了解详情的同学可以在相应的源码类里查看）:

```java
   new InputManagerService() // InputManagerService.java
-> nativeInit(..., queue) // com_android_server_input_InputManagerService.cpp
-> new NativeInputManager(..., looper) // com_android_server_input_InputManagerService.cpp
-> new InputManager(eventHub, ...) //InputManager.cpp      
```

重点就在这个 InputManager 里
```c++
InputManager::InputManager(...) {
    mDispatcher = new InputDispatcher(dispatcherPolicy);
    mReader = new InputReader(eventHub, readerPolicy, mDispatcher);
    initialize();
}

void InputManager::initialize() {
    mReaderThread = new InputReaderThread(mReader);
    mDispatcherThread = new InputDispatcherThread(mDispatcher);
}
```

我们可以看到，在 InputManager 里准备好了 mReader、mDispatcher，以及相关的两个线程，那么接下来当然就是把线程跑起来，好去读事件和分发事件。

启动事件读取和分发线程的调用链如下：

```JAVA
   SystemServer.startOtherServices()	
-> inputManagerService.start()
-> nativeStart() // com_android_server_input_InputManagerService.cpp
-> InputManager.start() // InputManager.cpp
```

最后的这个 start 方法很简单，就是把两个线程跑起来

```c++
status_t InputManager::start() {
    status_t result = mDispatcherThread->run("InputDispatcher", PRIORITY_URGENT_DISPLAY);
    result = mReaderThread->run("InputReader", PRIORITY_URGENT_DISPLAY);
}
```

至此，事件处理工作所需要的对象和线程都已经准备好了。前面那些代码看完记不住就算了，记住这句话：**SystemServer 启动 IMS 时，会创建一个 native 的 InputManager 对象，这个 InputManager 会通过 mReader 不断读事件，再通过 mDispatcher 不断分发事件**。

接下来我们看下，事件是怎么读取和分发的。

## 事件的读取

InputReaderThread 等待按键消息到来，该 Thread 在 threadLoop 中无尽的调用 InputReader 的 loopOnce 方法。

```c++
bool InputReaderThread::threadLoop() {
    mReader->loopOnce();
    return true;
}
```

在 loopOnce 方法中会通过 EventHub 来获取事件，并放入 buffer 中：

```c++
void InputReader::loopOnce() {
    // EventHub 从驱动读取事件
    size_t count = mEventHub->getEvents(timeoutMillis, mEventBuffer, EVENT_BUFFER_SIZE);
    ...
    if (count) {
    	 // 获取的事件是 RawEvent，需要处理成 KeyEvent、MotionEvent 等各种类型等
       processEventsLocked(mEventBuffer, count);
    }
    // 将队列中事件刷给监听器，监听器实际上就是 InputDispatcher 事件分发器。
    mQueuedListener->flush();
}
```

InputDispatcher 收到事件后调用，如果是 KeyEvent 会调用 notifyKey，如果是 MotionEvent 则会调用 notifyMotion

```c++
void InputDispatcher::notifyKey(const NotifyKeyArgs* args) {
    KeyEvent event;
    event.initialize(args->deviceId, args->source, args->action,
            flags, keyCode, args->scanCode, metaState, 0,
            args->downTime, args->eventTime);
    
    // 通过 NatvieInputManager 在 Event 入队前做一些处理
    mPolicy->interceptKeyBeforeQueueing(&event, /*byref*/ policyFlags);

    ...
    KeyEntry* newEntry = new KeyEntry(args->eventTime, args->source,
                args->action, flags, keyCode,...)
    // 事件放入队尾
    needWake = enqueueInboundEventLocked(newEntry);
}
```

以上，便是 InputReader 获取到设备事件通知 InputDispatcher 并存放到事件队列中的流程。

## 事件的分发

下面将介绍 InputDispatcher 如何从事件队列中读取事件并分发出去。

首先在 InputDispatcherThread 的 threadLoop 中无尽的调用 dispatchOnce 方法

```c++
bool InputDispatcherThread::threadLoop() {
    mDispatcher->dispatchOnce();
    return true;
}
```

该方法两个功能：

- 调用 dispatchOnceInnerLocked 分发事件；
- 调用 runCommandsLockedInterruptible 来处理 CommandQueue 中的命令，出队并处理，直到队列为空。

```c++
void InputDispatcher::dispatchOnce() {
     if (!haveCommandsLocked()) {
         dispatchOnceInnerLocked(&nextWakeupTime);
     }
     if (runCommandsLockedInterruptible()) {
         nextWakeupTime = LONG_LONG_MIN;
     }
  ...
}
```

在 dispatchOnceInnerLocked 中会处理多种类型的事件，这里关注按键类型的（其他如触摸，设备重置等事件流程稍有区别）。一通调用后到 PhoneWindowManager , 终于回到 java 了，

```c++
	 InputDispatcher::dispatchOnceInnerLocked
-> dispatchKeyLocked  
-> doInterceptKeyBeforeDispatchingLockedInterruptible     
-> NativeInputManager.interceptKeyBeforeDispatching
-> PhoneWindowManager.interceptKeyBeforeDispatching  
...
     
// 以下 NativeInputManager.doInterceptKeyBeforeDispatchingLockedInterruptible 的部分代码 
nsecs_t delay = mPolicy->interceptKeyBeforeDispatching(commandEntry->inputWindowHandle,
            &event, entry->policyFlags);
if (delay < 0) {
  // Home 事件将被拦截
  entry->interceptKeyResult = KeyEntry::INTERCEPT_KEY_RESULT_SKIP;    
} 
// 未被拦截的继续处理分发，本篇暂不分析
```

## PhoneWindowManager

话不多说，继续看代码，离胜利不远了！源码的注释写得很清楚，我这边就不翻译了。真的不是因为懒，是想让你们提高点英语阅读水平 。(-> <-）

```JAVA
public long interceptKeyBeforeDispatching(KeyEvent event, ...) {
 // First we always handle the home key here, so applications
 // can never break it, although if keyguard is on, we do let
 // it handle it, because that gives us the correct 5 second
 // timeout.
 if (keyCode == KeyEvent.KEYCODE_HOME) {
     // If we have released the home key, and didn't do anything else
     // while it was pressed, then it is time to go home!
     if (!down) {
         cancelPreloadRecentApps();
         mHomePressed = false;
         ... 
         // Delay handling home if a double-tap is possible.
         if (mDoubleTapOnHomeBehavior != DOUBLE_TAP_HOME_NOTHING) {
             mHomeDoubleTapPending = true;
             mHandler.postDelayed(mHomeDoubleTapTimeoutRunnable,
                     ViewConfiguration.getDoubleTapTimeout());
             return -1;
         }
         handleShortPressOnHome(); //短按

         //-1 调用处的事件结果就会赋值 INTERCEPT_KEY_RESULT_SKIP
         return -1;
     }
   	 ...
     // Remember that home is pressed and handle special actions.
     if (repeatCount == 0) {
         mHomePressed = true;
         if (mHomeDoubleTapPending) {
             handleDoubleTapOnHome();//双击
         } else if (mDoubleTapOnHomeBehavior == DOUBLE_TAP_HOME_RECENT_SYSTEM_UI) {
             preloadRecentApps();//最近 app
         }
     } else if ((event.getFlags() & KeyEvent.FLAG_LONG_PRESS) != 0) {
         if (!keyguardOn) {
             handleLongPressOnHome(event.getDeviceId());//长按
         }
     }
     return -1;
}
}
```

Home 的相关事件都在这处理啦。接下来我们就看一下 handleShortPressOnHome 短按 Home 进入 Luncher 是怎么实现的吧。

```java
private void handleShortPressOnHome() {
    ...
    // Go home! 坚持下，看完我们就 Go Home!
    launchHomeFromHotKey();
}
```

Go Home!

```java
void launchHomeFromHotKey(final boolean awakenFromDreams, final boolean respectKeyguard) {
    if (respectKeyguard) {
        // 处理一些锁屏的情况，可能直接 return 
    }
    // no keyguard stuff to worry about, just launch home!
    if (mRecentsVisible) {
    	  // 延时后台的打开 Activity 的操作，避免打扰用户的操作
        // 虽然方法名 stop 实际实现是延时 5s
        ActivityManager.getService().stopAppSwitches();
        // Hide Recents and notify it to launch Home
        hideRecentApps(false, true);
    } else {
        // Otherwise, just launch Home
        startDockOrHome(true /*fromHomeKey*/, awakenFromDreams);
    }
}
```
最后看一下跳转到 Home 的一些细节
```java
 void startDockOrHome(boolean fromHomeKey, boolean awakenFromDreams) {
     ActivityManager.getService().stopAppSwitches();
     // 关闭系统弹窗，如输入法
     sendCloseSystemWindows(SYSTEM_DIALOG_REASON_HOME_KEY);

     Intent dock = createHomeDockIntent();
     if (dock != null) { // 开启应用抽屉
         startActivityAsUser(dock, UserHandle.CURRENT);
         return;
     }
   
     intent = mHomeIntent //省略部分逻辑
     // 开启 Home 页面
     startActivityAsUser(intent, UserHandle.CURRENT);
 }
```



## 怎么回答

**面试官**：按下手机的 Home 键，有哪些动作和事件发生

🤔️：按下 Home 键后，底层驱动会获取这个事件， IMS 通过 Reader 读取驱动捕获的事件，再通过 Dispatcher 对事件进行分发。Dispatcher 分发事件前，PhoneWindowManager 会对 Home 和其它系统事件进行拦截处理，其中短按 Home 键的处理有：关闭相应的系统弹窗，延迟其它待打开的 Activity，最后使用 Intent 打开 Home 或者 Dock 页面。
