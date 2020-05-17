面试官提了一个问题，我们来看看 😎、😨 和 🤔️ 三位同学的表现如何吧

---
>😎 自认为无所不知，水平已达应用开发天花板，目前月薪 10k

**面试官**️：你了解 Android 系统启动流程吗？

😎：系统第一个启动的是 init 进程，然后 init 进程会再启动 Zygote、service manager、system_server 这些比较关键的服务进程。

**面试官**：system_server 是由 init 进程启动的吗？

😎：是的，但不用在意这些细节，没啥用。

**面试官**：好的，回去等通知吧

---

>😨 业余时间经常打游戏、追剧、熬夜，目前月薪 15k

**面试官**：你了解 Android 系统启动流程吗？

😨：系统首先会启动 init 进程，然后 init 进程会通过 init.rc 脚本做一些初始化工作，启动一些比较重要的服务进程，包括 Zygote、service manager 等。

**面试官**：system_server 进程是什么时候启动的？

😨：system_server 是在 Zygote 进程中启动的。

**面试官**：为什么要在 Zygote 中启动，而不是由 init 直接启动呢？

😨：嗯... 这个不清楚了，我只是大概了解关键服务进程的启动顺序，再深入的就没有去学过了

**面试官**：好的，回去等通知吧

---

>🤔️ 坚持每天学习、不断的提升自己，目前月薪 30k

**面试官**：你了解 Android 系统启动流程吗？

🤔️：当按电源键触发开机，首先会从 ROM 中预定义的地方加载引导程序 BootLoader 到 RAM 中，并执行 BootLoader 程序启动 Linux Kernel， 然后启动用户级别的第一个进程： init 进程。

init 进程会解析 init.rc 脚本做一些初始化工作，包括挂载文件系统、创建工作目录以及启动系统服务进程等，其中系统服务进程包括 Zygote、service manager、media 等。

在 Zygote 中会进一步去启动 system_server 进程，然后在 system_server 进程中会启动 AMS、WMS、PMS 等服务，等这些服务启动之后，AMS 中就会打开 Launcher 应用的 home Activity，最终就看到了手机的 "桌面"。

**面试官**：system_server 为什么要在 Zygote 中启动，而不是由 init 直接启动呢？

🤔️：Zygote 作为一个孵化器，可以提前加载一些资源，这样 fork() 时基于 Copy-On-Write 机制创建的其他进程就能直接使用这些资源，而不用重新加载。比如 system_server 就可以直接使用 Zygote 中的 JNI 函数、共享库、常用的类、以及主题资源。

**面试官**：为什么要专门使用 Zygote 进程去孵化应用进程，而不是让 system_server 去孵化呢？

🤔️：首先 system_server 相比 Zygote 多运行了 AMS、WMS 等服务，这些对一个应用程序来说是不需要的。另外进程的 fork() 对多线程不友好，仅会将发起调用的线程拷贝到子进程，这可能会导致死锁，而 system_server 中肯定是有很多线程的。

**面试官**：能说说具体是怎么导致死锁的吗？

fork() 时只会把调用线程拷贝到子进程、其他线程都会立即停止，那如果一个线程在 fork() 前占用了某个互斥量，fork() 后被立即停止，这个互斥量就得不到释放，再去请求该互斥量就会发生死锁了。

**面试官**：Zygote 为什么不采用 Binder 机制进行 IPC 通信？

🤔️：Binder 机制中存在 Binder 线程池，是多线程的，如果 Zygote 采用 Binder 的话就存在上面说的 fork() 与 多线程的问题了。

其实严格来说，Binder 机制不一定要多线程，所谓的 Binder 线程只不过是在循环读取 Binder 驱动的消息而已，只注册一个 Binder 线程也是可以工作的，比如 service manager 就是这样的。

实际上 Zygote 尽管没有采取 Binder 机制，它也不是单线程的，但它在 fork() 前主动停止了其他线程，fork() 后重新启动了。

**面试官**：可以，我们再来聊聊别的。