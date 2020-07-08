面试官提了一个问题，我们来看看 😎、😨 和 🤔️ 三位同学的表现如何吧

---
>😎 自认为无所不知，水平已达应用开发天花板，目前月薪 10k

**面试官**：Android 开发经常接触到的 Context 是什么呢？

😎：Context 是一个关于应用环境的抽象类，它的实现由安卓系统提供。用来访问一些应用内资源、类，也可以调用系统服务开启 Activity 、Service 、发送和接收广播等

**面试官**：那一个应用里有几个 Context 呢？

😎：四大组件里 Activity 和 Service 都是 Context , 应用的 Context 数就是 Activity 、Service、Application 的个数之和，顺便说一下 Application 如果是多进程应用就会有多个

**面试官**：那它们有什么区别呢？

😎：好像都差不多，平常开发的时候用哪个 Context 效果都一样，主要不同就是 Application 的生命周期和应用一样，所以在初始化一些第三方库的时候如果要传 Context 要用 Application

**面试官**：说到 Application ， getApplication 和 getApplicationContext 有什么区别？

😎：没区别~

**面试官**：好的，回去等通知吧

---
>😨 业余时间经常打游戏、追剧、熬夜，目前月薪 15k

**面试官**：Android 有哪些类型的 Context ，它们有什么区别

😨：应用里有 Activity 、Service、Application 这些 Context ，我们先说说它们的共同点，它们都是 ContextWrapper 的子类，而 ContextWrapper 的成员变量 mBase 可以用来存放系统实现的 ContextImpl，这样我们在调用如 Activity 的 Context 方法时，都是通过静态代理的方式最终调用到 ContextImpl 的方法。我们调用 ContextWrapper 的 getBaseContext 方法就能拿到 ContextImpl 的实例

再说它们的不同点，它们有各自不同的生命周期；在功能上，只有 Activity 显示界面，正因为如此，Activity 继承的是 ContextThemeWrapper 提供一些关于主题，界面显示的能力，间接继承了 ContextWrapper ； 而 Applicaiton 、Service 都是直接继承 ContextWrapper ，所以我们要记住一点，凡是跟 UI 有关的，都应该用 Activity 作为 Context 来处理，否则要么会报错，要么 UI 会使用系统默认的主题。

**面试官**： 在 Activity 里，this 和 getBaseContext 有什么区别

😨：this 呢，指的就是 Activity 本身的这个实例，而 getBaseContext ，是 Activity 间接继承的 ContextWrapper 的一个方法，用来返回系统提供的 ContextImpl 对象


**面试官**：getApplication 和 getApplicationContext 有什么区别？

😨：用起来没区别，都是返回应用的 Application 对象；但是从来源上， getApplication 是 Activity 、 Service 里的方法， 而 getApplicationContext 则是 Context 里的抽象方法，所以能调用到的它们的地方不一样。

**面试官**：ContextImpl 实例是什么时候生成的，在 Activity 的 onCreate 里能拿到这个实例吗

😨：这个都是系统处理的，具体时机没有跟进去看。onCreate 是 Activity 的第一个生命周期回调，应该还没准备好，不能获取吧。

**面试官**：好的，回去等通知吧

---
>🤔️ 坚持每天学习、不断的提升自己，目前月薪 30k

**面试官**：ContextImpl 实例是什么时候生成的，在 Activity 的 onCreate 里能拿到这个实例吗

🤔️：先说结论，可以。我们开发的时候，经常会在 onCreate 里拿到 Application，如果用 getApplicationContext 取，最终调用的就是 ContextImpl 的 getApplicationContext 方法，如果调用的是 getApplication 方法，虽然没调用到 ContextImpl ，但是返回 Activity 的成员变量 mApplication 和 ContextImpl 的时机是一样的。

再说下它的原理，Activity 真正开始启动是从 ActivityThread.performLaunchActivity 开始的，这个方法做了这些事：
* 通过 ClassLoader 去加载目标 Activity 的类，从而创建 对象
* 从 packageInfo 里获取 Application 对象
* 调用 createBaseContextForActivity 方法去创建 ContextImpl
* 调用 activity.attach ( contextImpl , application) 这个方法就把 Activity 和 Application 以及 ContextImpl 关联起来了，就是上面结论里说的时机一样
* 最后调用 activity.onCreate 生命周期回调

通过以上的分析，我们知道了 Activity 是先创建类，再初始化 Context ，最后调用 onCreate , 从而得出问题的答案。
不仅 Activity 是这样， Application 、Service 里的 Context 初始化也都是这样的。

**面试官**：那 ContentProvider 里的 Context 又是什么时候初始化的呢？

🤔️：ContentProvider 本身不是 Context ，但是它有一个成员变量 mContext ，是通过构造函数传入的。那么这个问题就变成了，ContentProvider 什么时候创建。
应用创建 Application 是通过调用 ActivityThread.handleBindApplication 方法，这个方法的相关流程有：
* 创建 Application 
* 初始化 Application 的 Context 
* 调用 installContentProviders 并传入刚创建好的 Application 来创建 ContentProvider
* 调用 Application.onCreate

得出结论，ContentProvider 的 Context 是在 Applicaiton 创建之后，但是 onCreate 方法调用之前初始化的

**面试官**：四大组件就剩 BroadcastReceiver ，说说它方法里的 Context 是哪来的

🤔️：广播接收器，分动态注册和静态注册。
动态注册很简单，在调用 Context.registerReceiver 动态注册 BroadcastReceiver 时，会生成一个 ReceiverDispatcher 会持有这个 Context ，这样当有广播分发到它时，调用 onReceiver 方法就可以把 Context 传递过去了。当然，这也是为什么不用的时候要 unregisterReceiver 取消注册，不然这个 Context 就泄漏了哦。
静态注册时，在分发的时候最终调用的是 ActivityThread.handleReceiver ，这个方法直接通过 ClassLoader 去创建一个 BroadcastReceiver 的对象，而传递给 onReceiver 方法的 Context 则是通过 context.getReceiverRestrictedContext() 生成的一个以 Application 为 mBase 的 ContextWrapper 。注意这边的 Context 不是 Application 哈。

**面试官**：可以，我们再来聊聊别的。

---

> 看完了这三位同学的面试表现，你有什么感想呢？欢迎关注 “Android 面试官” 在公众号后台留言讨论

![img](img/qrcode.jpg)

<center><B><font color=red> 听说🤔️同学也关注我了哦 <font></B></center>

