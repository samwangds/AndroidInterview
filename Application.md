面试官提了一个问题，我们来看看 😎、😨 和 🤔️ 三位同学的表现如何吧

---

>😎 自认为无所不知，水平已达应用开发天花板，目前月薪 10k

**面试官**：说说 Application 的作用。

😎：Application 是应用进程创建后就会创建的系统组件，所以可以用它来做一些初始化操作；Application 生命周期和应用进程一样长，所以可以用来给类库提供 Context; 因为在所有 Context 可以获得 Application 所以可以用来保存和传递全局变量。

**面试官**：你平常开发会把全局变量放在 Application ? 那应用在后台被回收，重新打开的时候值丢失怎么办？

😎：会啊，很方便， 做一下容错判空就可以了

**面试官**：好的，回去等通知吧



---

>😨 业余时间经常打游戏、追剧、熬夜，目前月薪 15k

**面试官**：说说对 Application 的理解 

😨：作用：做初始化操作、提供上下文。另外 Application 是一个 Context ，它直接继承了 ContextWrapper ；这个 ContextWrapper 的成员变量 mBase 可以用来存放系统实现的 ContextImpl，这样我们在调用 Application 的 Context 方法时，都是通过静态代理的方式最终调用到 ContextImpl 的方法。我们调用 ContextWrapper 的 getBaseContext 方法就能拿到 ContextImpl 的实例

**面试官**：你平常开发会把全局变量放在 Application ? 那应用在后台被回收，重新打开的时候值丢失怎么办？

😨：不会，保存全局变量用静态变量，或单例可以把它们聚集在更合适的位置。
避免应用被回收数据丢失，可以页面传递参数时，通过 Intent 传递参数，这样被回收后打开重新从 Intent 取参还是有值的。数据量大的话也可以考虑数据持久化；另一个方法是通过 `onSaveInstanceState` 和 `onRestoreInstanceState` 分别在被回收时保存相应的数据以及在重新打开时恢复数据。

**面试官**：讲一下 Application 的生命周期吧

😨：相比 Activity ，Application 的生命周期简直不要太简单。首先创建的时候会调用构造函数，然后系统准备好 ContextImpl 通过 `attachBaseContext( Context )` 方法注入到 Application，接着调用我们最熟悉的 onCreate 方法。API 里还有一个 `onTerminate` 方法在进程被杀死的时候会回调，不过仅在模拟器生效，就不需要关注了。

**面试官**：那你能接着说一下 Application 的初始化流程吗？

😨：基本上就是上面说的那些，再细没有去了解了

**面试官**：好的，回去等通知吧

---

>🤔️ 坚持每天学习、不断的提升自己，目前月薪 30k

**面试官**：说一下 Application 的初始化流程

🤔️：Application 的初始化是在应用进程创建完成后：

* ActivityThread 调用 AMS 的 Binder 对象（ IActivityManager ）的  attachApplication 方法
* AMS 收到请求后调用再去调用 ActivityThread 的 bindApplication 方法
* ActivityThread 这边收到请求再组装一个 AppBindData 对象，把所有参数封装进去，再通过 handler 发到主线程执行
* 主线程 loop 到这条消息，调用 handleBindApplication 来真正处理初始化 Application

handleBindApplication 和我们谈 “Context” 那次，Activity 的初始化差不多。回顾一下：

* ClassLoader 加载 Application 类，实例化
* 初始化 Applicaction 用的 ContextImpl
* 通过 Application.attach( Context ) 方法，调用 `attachBaseContext( Context )` 将 ContextImpl 注入到 Application 
* 最后调用 Application.OnCreate() 

这样 Application 就初始化完成了

**面试官**：为什么进程创建完成不直接调 handleBindApplication 去创建 Application 呢，又去 AMS 那边绕了一圈

🤔️：调用 AMS 的 attachApplication 不仅仅是为了创建 Application ，还有在进程创建前可能调用了应用的四大组件没办法启动；现在进程创建好了，创建好 Application 也要处理这些待启动的组件。所以需要通过 AMS 统一调度，如果 Application 的创建及 onCreate 回调耗时的话，也会影响这些待启动组件启动时间

**面试官**：可以，我们再来聊聊别的。

---

> 看完了这三位同学的面试表现，你有什么感想呢？欢迎关注 “Android 面试官” 在公众号后台留言讨论

![img](img/qrcode.jpg)

<center><B><font color=red> 听说🤔️同学也关注我了哦 <font></B></center>

