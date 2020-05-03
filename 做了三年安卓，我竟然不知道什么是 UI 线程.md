**面试官**：说说什么是 UI 线程？
  
😏：就是用来刷新 UI 所在的线程嘛
  
**面试官**：多说点
  
😏：UI 是单线程刷新的，如果多个线程可以刷新 UI 就无所谓是不是 UI 线程了，单线程的好处是，UI 框架里不需要到处上锁，做线程同步，写起来也比较简单有效
  
**面试官**：你说的这个 UI 线程，它到底是哪个线程？是主线程吗？
  
😏：拿 Activity 来说，我们在 Activity 里异步做完耗时操作，要刷新 UI 可以调用 Activity.runOnUiThread 方法，在 UI 线程中执行，那么我们看下这个方法自然就知道 UI 线程是哪个线程了。
  
```java
public final void runOnUiThread(Runnable action) {
    if (Thread.currentThread() != mUiThread) {
        mHandler.post(action);
    } else {
        action.run();
    }
}
```
这个方法会判断当前是不是在主线程，不是呢就通过 mHandler 抛到主线程去执行。
这个 mHandler 是 Activity 里的一个全局变量，在 Activity 创建的时候通过无参构造函数 `new Handler()` 一起创建了。因为是无参，所以创建时用的哪个线程，Handler 里的 Looper 用的就是哪个线程了。创建 Activity 是在应用的主线程，因此 mHandler.post 去执行的线程也是主线程。
刚也说 了，runOnUiThread 方法里，先判断是不是在 UI 线程，这个 mUiThread 又是什么时候赋值的呢，答案还在 Activity 的源码里
```java
final void attach(Context context, ...) {
 ...省略无关代码
 mUiThread = Thread.currentThread();
}
```
在 Activity.attach 方法里，我们把当前线程赋值给 mUiThread，那当前线程是什么线程呢，也是主线程。至于为什么创建 Activity 和 attach 都是主线程，那又是另外一个故事了
通过前面的分析，我们知道了，对于 Activity 来讲 UI 线程就是主线程
  
**面试官**：所以你的结论是 UI 线程就是主线程？
  
😏：这是你说的，记住这个开发的时候不会错，但是不够准确。在子线程里刷新 UI 的时候会抛一个异常 `ViewRootImpl$CalledFromWrongThreadException: Only the original thread that created a view hierarchy can touch its views.` 大意是只有最初始创建 View 层级关系的线程可以 touch view，这里指的也就是 ViewRootImpl 创建时所在的线程，严格来说这个线程不一定是主线程。这一点呢，读 View.post 方法也可以得到相同的结论。所以对于 View 来说，UI 线程就是 ViewRootImpl 创建时所在的线程，Activity 的 DecorView 对应的 ViewRootImpl 是在主线程创建的
  
**面试官**：这个 ViewRootImpl 什么时候创建
  
😏：Activity 创建好之后，应用的主线程会调用 ActivityThread.handleResumeActivity，这个方法会把 Activity 的 DecorView 添加到 WindowManger 里，就是在这个时候创建的 ViewRootImpl
  
**面试官**：那可以在异步线程刷新 View 吗？
  
😏：刚才我们说了，只要是 ViewRootImpl 创建的线程就可以 touch view，然后 WindowManger.addView 的时候又会去创建 ViewRootImpl，所以我们只要在子线程调用 WindowManger.addView，这个时候添加的这个 View，就只能在这个子线程刷新了，这个子线程就是这个 View 的 UI 线程了。
  
**面试官**：好，我们再聊点别的