> Zygote 单词意思是“受精卵”，寓意着它可以“孕育”出一个“新生命”。在 Android 中大多数的进程和系统进程都是通过 Zygote 来生成的。

## Zygote 的作用

其实这个作用前面简介就已经概括了，Zygote 就两个作用

- 启动 SystemServer
- 孵化应用进程 

通过 Zygote fork 出这些进程，这些进程就可以直接继承 Zygote 准备好的资源，比如：

- 常用类
- JNI 函数
- 主题资源
- 共享库

直接继承资源，避免重新去加载这些资源，从而提高了系统的性能。

## Zygote 的启动流程

init 进程是安卓启动的第一个进程，而 Zygote 是由 init 进程解析 rc 脚本时启动的。早期的 Android 版本中 Zygote 的启动命令直接被写在 init.rc 中。但随着硬件升级，Android 系统同时面对 32 位和 64 位的机器，因而对 Zygote 的启动也需要根据不同的情况区分对待：

```c++
/* system/core/rootdir/init.rc */
import /init.${ro.hardware}.rc
import /init.${ro.zygote}.rc
```

在 init.rc 文件中，根据系统属性 ro.zygote 的具体值，加载不同的描述 Zygote 的 rc 脚本，比如：

- init.zygote32.rc -> 32 位的机器 
- init.zygote32_64.rc -> Primary Arch
- init.zygote64_32.rc -> Second Arch
- init.zygote64.rc -> 64 位的机器 

#### init.zygote64.rc

以 64 位的 rc 文件为例，我们来看下里面的内容：

```
service zygote /system/bin/app_process64 -Xzygote /system/bin --zygote --start-system-server
    class main
    priority -20
    user root
    group root readproc reserved_disk
    socket zygote stream 660 root system
    onrestart write /sys/android_power/request_state wake
    onrestart write /sys/power/state on
    onrestart restart audioserver
    onrestart restart cameraserver
    onrestart restart media
    onrestart restart netd
    onrestart restart wificond
    writepid /dev/cpuset/foreground/tasks
```

zygote 进程的启动方式是 fork + execve 系统调用：

```c++
pid_t pid = fork();
// 调用fork 启动子进程
// 会返回两次
if(pid == 0) { 
  // fork 出来的子进程
  // 加载新的二进程程序，来替换继承的父进程资源 
  execve(path, argv, env)
} else {
  // 父进程
}
```

这个脚本里最核心的就是第一行了，拆分一下得到信息（也就是启动进程后需要的 execve 参数 ）：

- ServiceName: zygote
- Path: /system/bin/app_process64
- Arguments:  -Xzygote /system/bin --zygote --start-system-server

从 zygote 的 path 路径可以看出，它所在的程序名叫“app_process64”，不像 ServiceManager 是在一个独立的程序中。通过指定 --zygote 参数 ，app_process 可以识别出用户是否需要启动 zygote。

在我们这个场景中，init.rc 指定了 --zygotę 选项, 因而 app_process 接下来将启动 ZygoteInit，并传入 start-system-server 作为参数之一标记需要启动 System Server, 接下来 ZygoteInit 会运行于 Java 虚拟机之上（在此之前都是在 native 层运行, app_process 会去启动虚拟机）。我们来看下相关代码：

```JAVA
//frameworks/base/core/java/com/android/internal/os/ZygoteInit.java
public static void main(String argv[]) {
    //....
    for (int i = 1; i < argv.length; i++) {
      // 读取参数 
         if ("start-system-server".equals(argv[i])) {
             startSystemServer = true; // 需要启动 SystemServer
         } else if ("--enable-lazy-preload".equals(argv[i])) {
             enableLazyPreload = true;
         } else if (argv[i].startsWith(SOCKET_NAME_ARG)) {
             socketName = argv[i].substring(SOCKET_NAME_ARG.length());
         } else ...
     }
     // 注册 socket
     zygoteServer.registerServerSocketFromEnv(socketName);
     // 预加载种类资源
     preload(bootTimingsTraceLog);
     if (startSystemServer) {
         // 去启动 systemServer
         Runnable r = forkSystemServer(abiList, socketName, zygoteServer);
         // {@code r == null} in the parent (zygote) process, and {@code r != null} in the
         // child (system_server) process.
         if (r != null) {
             r.run();
             return;
         }
     }
     // loops forever in the zygote.
     caller = zygoteServer.runSelectLoop(abiList);
     // We're in the child process and have exited the select loop. 
     // Proceed to execute the command.
     if (caller != null) {
         caller.run();
     }
 }
```

精简下这段代码并不复杂，它主要完成以下三项工作：

- 注册一个 Socket：<br>
  Zygote 是孵化器，一时有新程序需要运行时，系统会通过这个 Soket (名称：ANDROID_SOCKET_zygote) 通知 Zygote，并由它负责实际的进程孵化过程。
- 预加载各种资源, preload 方法里包含：<br>
  - preloadClasses();
  - preloadResources();
  - preloadOpenGL();
  - preloadSharedLibraries();
  - preloadTextResources();
- 启动一个 Loop 循环，去处理 Socket 接收到的事件

至此，Zygote 便启动完成了。至于 Zygotę 具体是怎么启动 SystemServer 和孵化应用进程的，感兴趣的不妨点赞在看，我们后面再谈。