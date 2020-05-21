面试官提了一个问题，我们来看看 😎、😨 和 🤔️ 三位同学的表现如何吧

---

>😎 自认为无所不知，水平已达应用开发天花板，目前月薪 10k

**面试官**：说一说 Android 的消息机制

😎：就是可以通过 Handler 发送消息，这个消息会在 Handler 指定的线程处理。处理线程默认为 new Handler 时所在的线程；也可以通过构造函数传入一个 Looper ，则处理线程为 Looper 所在的线程。

**面试官**：这个消息机制有哪些应用场景？

😎：在子线程处理完耗时操作，可以通过 Handler 发送消息，在主线程处理界面刷新；另外一点就是可以发送延时消息，用来做一些延时操作。

**面试官**：你知道延时消息的原理吗？

😎：不知道，会用不就行了？

**面试官**：好的，回去等通知吧



---

>😨 业余时间经常打游戏、追剧、熬夜，目前月薪 15k

**面试官**：说一说 Android 的消息机制

😨：Android 的消息机制主要是指 Handler 的运行机制以及 Handler 所附带的 MessageQueue 和 Looper 的工作流程。消息的处理流程是这样的：

1. 通过 Handler 的 sendXXX 或 postXXX 系列方法发送一条消息
2. 这条消息插入到 MessageQueue 中的指定位置
3. Looper.loop() 不断的调用 MessageQueue.next() 取出当前需要处理的消息，若当前无消息则阻塞到有新消息
4. 取出来的消息，通过 Handler.dispatchMessage 进行分发处理

**面试官**：你知道延时消息的原理吗？

😨：发送的消息 Message 有一个属性 when 用来记录这个消息需要处理的时间，when 的值：普通消息的处理时间是当前时间；而延时消息的处理时间是当前时间 + delay 的时间。Message 会按 Message.when 递增插入到 MessageQueue ，也就是越早时间的排在越前面。

接下来就是消息的处理了，Looper.loop() 不断调用 MessageQueue.next() 取出当前需要处理的消息，而 next() 方法会判断队头消息的 Message.when ，如果时间还没到，就休眠到指定时间；如果当前时间已经到了，就返回这个消息，交给 Handler 去分发。这样就实现处理延时消息了。

**面试官**：Handler 是怎么分发消息的？

😨：简单的只需要看源码

``` java
public void dispatchMessage(Message msg) {
    if (msg.callback != null) {
    // callback 的值为通过 Handler.post(Runnable) 方法发送消息时传入的 Runnable
        message.callback.run();
    } else {
        if (mCallback != null) {
          // 这个 mCallback 是调用在 Handler 构造函数时可选传入的
          // 传入 CallBack 就省得为了重载 handleMessage 而新写一个 Handler 的子类
            if (mCallback.handleMessage(msg)) {
                return;
            }
        }
      // 最后就是交给 Handler 自己处理 Message 啦
        handleMessage(msg);
    }
}
```

很有意思的一个点，mCallback 处理完如果返回 false，还是会继续往下走，再交给 Handler.handleMessage 处理的。所以这边可以通过反射去 hook 一个 Handler ，可以监听 Handler 处理的每个消息，也可以改 msg 里面的值。

**面试官**：主线程的 Looper.loop() 在死循环，为什么没有 ANR 呢？

😨：这个还真没想过

**面试官**：好的，回去等通知吧

---

>🤔️ 坚持每天学习、不断的提升自己，目前月薪 30k

**面试官**：主线程的 Looper.loop() 在死循环，为什么没有 ANR 呢？

🤔️：首先 ANR 的根本原因是：应用未在规定的时间内处理 AMS 指定的任务才会 ANR。AMS 调用到应用端的 Binder 线程，应用再将任务封装成 Message 发送到主线程 Handler ，Looper.loop()  通过 MessageQueue.next() 拿到这个消息进行处理。如果不能及时处理这个消息呢，肯定是因为在它前面有耗时的消息处理，或者因为这个任务本身就很耗时。所以 ANR 不是因为 loop 循环，而是因为主线程中有耗时任务。

**面试官**：主线程的 Looper.loop() 在死循环，会很消耗资源吗？

🤔️： Looper.loop()  通过 MessageQueue.next() 取当前需要处理的消息，如果当前没有需要处理的消息呢，会调用 nativePollOnce 方法让线程进入休眠。当消息队列没有消息时，无限休眠；当队列的第一个消息还没到需要处理的时间时，则休眠时间为 Message.when - 当前的时间。这样在空闲的时候主线程也不会消耗额外的资源了。而当有新消息入队时 enqueueMessage 里会判断是否需要通过 nativeWake 方法唤醒主线程来处理新消息。唤醒最终是通过往 eventfd 发起一个写操作，这样主线程就会收到一个可读事件进而从休眠状态被唤醒。

**面试官**：你知道 IdleHandler 吗？

🤔️：IdleHandler 是通过 MessageQueue.addIdleHandler() 来添加到 MessageQueue 的，前面提到当 MessageQueue.next() 当前没有需要处理消息的时候就会进入休眠，而在进入休眠之前呢，就会调用 IdleHandler 接口里的 boolean queueIdle() 方法。这个方法的返回 true 则调用后保留，下次队列空闲时还会继续调用；而如果返回 false 调用完就被 remove 了。可以用来做延时加载，而且是在空闲时加载，不像 Handler.postDelayed 需要指定延时的时间。

**面试官**：可以，我们再来聊聊别的。

---

> 看完了这三位同学的面试表现，你有什么感想呢？欢迎关注 “Android 面试官” 在公众号后台留言讨论


