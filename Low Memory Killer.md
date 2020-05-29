你一定听说过 Android 的应用保活，可能也知道几种保活方案，但这是一件没有门槛的事，任何人都可以轻易的从网上搜到，我们的目标应该是成为方案的创造者或改进者，而不仅是搬运工。

## LMK 是什么？

Linux Kernel 有自己的内存监控机制 OOMKiller。当系统的可用内存达到临界值时，OOMKiller 就会按照优先级从低到高杀掉进程。

优先级该如何衡量呢？OOMKiller 会综合进程当前消耗内存、进程占用 CPU 时间、进程类型等因素，对进程实时评分。分值存储在 /proc/{PID}/oom_score 中，可通过 cat 命令查看。分值越低的进程，优先级越高，被杀死的概率越小。

基于 Linux Kernel 的 OOMKiller 思想，Android 系统拓展出了自己的内存监控体系，相比 Linux 达到临界值才触发，Android 实现了**不同梯级**的 Killer，并为此开发了专门的驱动，名为 Low Memory Killer。

## LMK 的梯级规则

LMK 的源码位于内核的 /drivers/staging/android/Lowmemorykiller.c，Lowmemorykiller.c 中有如下定义：
```objectivec
static int lowmem_adj[6] = {0, 1, 6, 12};
static int lowmem_adj_size = 4; //页大小

//元素使用时以 lowmem_adj_size 为单位，即乘以 lowmem_adj_size
static size_t lowmem_minfree[6] = { 
    3 * 512, //6MB
    2 * 1024, //8MB
    4 * 1024, //16MB
    16 * 1024，//64MB
};
```
lowmem_minfree 定义了可用内存容量对应的不同梯级。lowmem_adj 与 lowmem_minfree 中的梯级一一对应，表示处于某梯级时需要被处理的 adj 值。adj 值用来描述进程的优先级，取值范围为 -17~15，数字越小表示进程优先级越高，被杀死的概率越小。

比如当可用内存低于 64MB 时，即 lowmem_minfree 第 4 梯级，对应于 lowmem_adj 的 12，那就会清理掉优先级低于 12（即 adj>12）的进程。

## 自定义梯级

上面这两个数组中梯级的定义只是系统的预定义值，Android 系统还提供了相应的文件供我们修改这两组值，路径为：

```objectivec
/sys/module/lowmemorykiller/parameters/adj
/sys/module/lowmemorykiller/parameters/minfree
```
可以在 init.rc (系统启动时由 init 进程解析的一个脚本) 中，这样修改：
```objectivec
write /sys/module/lowmemorykiller/parameters/adj        0, 8
write /sys/module/lowmemorykiller/parameters/minfree 1024, 4096
```

除了启动时机，也能在系统运行时改变。ActivityManagerService 在运行时会根据当前的系统配置自动调整 adj 和 minfree，以尽可能适配不同的硬件设备，它的 updateOomLevels 方法也是通过修改上面两个文件来实现的。 

## adj

了解了 Low Memory Killer 的梯级规则后，来看下 Android 进程的 adj（Adjustment） 值含义：

| ADJ | 说明 |
|--|--|
| HIDDEN_APP_MAX_AD = 15 | 只运行了不可见 Activity 的进程 |
| HIDDEN_APP_MIN_ADJ = 9  | 只运行了不可见 Activity 的进程 |
| SERVICE_B_ADJ = 8  | B list of Service |
| PREVIOUS_APP_ADJ = 7  | 用户的上一个产生交互的进程 |
| HOME_APP_ADJ = 6  | Launcher 进程 |
| SERVICE_ADJ = 5  | 当前运行了 application service 的进程 |
| BACKUP_APP_ADJ = 4  | 用于承载 backup 相关操作的进程 |
| HEAVY_WEIGHT_APP_ADJ = 3  | 重量级应用程序进程 |
| PERCEPTIBLE_APP_ADJ = 2  | 能被用户感觉但不可见，如后台运行的音乐播放器 |
| VISIBLE_APP_ADJ = 1  | 有前台可见的 Activity |
| FOREGROUND_APP_ADJ = 0  | 当前正在前台运行与用户交互的进程 |
| PERSISTENT_PROC_ADJ = -12  | Persistent 性质的进程，如 telephony |
| SYSTEM_ADJ = -16  | 系统进程 |

除了表格中系统的评定规则，我们有没有办法改变某一进程的 adj 值呢？和修改上面的 adj、minfree 梯级类似，进程的 adj 值也可以通过写文件的方式来修改的，路径为 /proc/{PID}/oom_adj，比如在 init.rc 中是这样修改的：
```objectivec
write /proc/1/oom_adj -16
```