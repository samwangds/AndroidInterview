 # Activity 的 LaunchMode 及使用场景

`android:launchMode` 有关应如何启动 Activity 的指令。共有四种模式可与 `Intent` 对象中的 Activity 标记（`FLAG_ACTIVITY_*` 常量）协同工作，以确定在调用 Activity 处理 Intent 时应执行的操作。启动模式有以下四种：

#### standard 标准模式

这是默认模式，每次激活 Activity 时都会创建 Activity 实例，并放入任务栈中。使用场景：大多数 Activity。

* 本应用打开 activity 实例放入本应用，即发送 intent 的 task 栈的顶部。Activity A 启动了 Activity B (standard), 则 B 运行在 A 所在的栈中。
* 用 ApplicationContext 去启动 standard 模式的 Activity 会报错，由于非 Activity 的 Context 并没有任务栈，所以新开的 Activity 不知道应该放哪，解决方法是指定 FLAG_ACTIVITY_NEW_TASK，这样启动的时候就会为它创建一个新的 Task，实际是以 singleTask 启动的
* 跨应用打开仍在同一个栈中，但是 recent app 是两个页面； 如果 intent flag 带有`FLAG_ACTIVITY_RESET_TASK_IF_NEEDED`,则跨应用会在同一个 recent app

#### singleTop 栈顶复用

如果在当前任务的栈顶正好存在该 Activity 的实例，就重用该实例( 会调用实例的 onNewIntent() )，否则就会创建新的实例并放入栈顶，即使栈中已经存在该 Activity 的实例，只要不在栈顶，都会创建新的实例。使用场景如新闻类或者阅读类 App 的内容页面。

#### singleTask 栈内复用

会在系统中查找属性值 affinity 等于它的属性值 taskAffinity 的任务栈，如果存在则在该这个任务栈中启动，否则就在建新任务栈（affinity 值为它的 taskAffinity）启动（FLAG_ACTIVITY_NEW_TASK 同样的情况）
如果在任务栈中已经有该 Activity 的实例，就重用该实例(会调用实例的 onNewIntent() )。重用时，会让该实例回到栈顶，因此在它上面的实例将会被移出栈(即 singleTask 有 clearTop 的效果)。如果栈中不存在该实例，将会创建新的实例放入栈中。

适用于项目首页。

会使 flag`FLAG_ACTIVITY_RESET_TASK_IF_NEEDED` 失效

例子：

* 目前任务栈 S1 中情况为 ABC，此时 Activity D 以 singleTask 请求启动，需要的任务栈 S2，由于 S2, D 均不存在，则系统先创建任务栈 S2，再创建 D 放入 S2 中。
* 若 D 需要的是 S1，其它情况同上述，则系统直接创建 D，放入栈 S1 中
* 若 D 需要的是 S1，且 S1 的情况 ADBC，此时 D 不会重新创建，而是移除 BC，D 即为栈顶，调用 onNewIntent

#### singleInstance 单实例模式

加强版的 singleTask, 具有所有的特性，与 singleTask 唯一的区别 系统不会在 singleInstance activity 的 task 栈中放入任何其他的 activity 实例，**单独位于一个 task 中** 

singleInstance 不要用于中间页面，如果用于中间页面，跳转会有问题，比如：A -> B (singleInstance) -> C，由于 B 是自己一个任务栈，而 AC 默认情况下是同一个栈，这样回退的时候就是 C -> A -> B 可能会让用户懵逼。

应用场景：来电界面、Launcher 页面


#### 启动模式的使用方式

1. Manifest.xml 静态指定，在代码中跳转时会依照指定的模式来创建 Activity

```
<activity
  android:name="..activity.MultiportActivity"
  android:launchMode="singleTask"/>
```

2. 启动时在 Intent 中动态指定

```
Intent intent = new Intent();
intent.setClass(context, MainActivity.class);
intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
context.startActivity(intent);
```

差别：

* 优先级方法 2 动态指定较高，同时存在时以 2 为准；
* 范围：方法 1 无法指定 FLAG_ACTIVITY_CLEAR_TOP，方法 2 无法指定 singleInstance 模式

#### Activity 的 Flags

* FLAG_ACTIVITY_NEW_TASK 如果 taskAffinity 一样则与标准模式一样新启动一个 Activity,如果不一样则新建一个 task 放该 Activity
* FLAG_ACTIVITY_SINGLE_TOP 与 SingleTop 效果一致
* FLAG_ACTIVITY_CLEAR_TOP 销毁目标 Activity 和它之上的所有 Activity，重新创建目标 Activity，+ FLAG_ACTIVITY_SINGLE_TOP 效果与 SingleTask 效果一致
* FLAG_ACTIVITY_EXCLUDE_FROM_RECENTS 具有此标记位的 Activity 不会出如今历史 Activity 的列表中

其它 flag 查文档

#### activity stack 、 task

* stack 用栈的方式管理 Activity,这样后入的 activity 会被先移除
* Task 是指将相关的 Activity 组合到一起，以 Activity Stack 的方式进行管理，使得多个应用的 Activity 可以组合共同完成一个任务(Task)。

#### Taskaffinity

> 值为字符串，必须包含分隔符.

用来控制 Activity 所属的任务栈，可以翻译为任务相关性。设置 activity 的启动模式为 singleTask 或 singleInstance 才能生效(其实 singleInstance 本来就会在新 Task 中)。    
不同的 TaskStack, Affinity 可以一样。

1. 通过在 Manifest.xml 配置

```
<activity android:name=".bActivity"
            android:launchMode="singleTask"
            android:taskAffinity="task.name"/>
```
2. Intent 动态设置

如果在 xml 设未设置 launchMode，只设置了 taskAffinity 则需要在 intent 时指定相关的 launchMode 的 flag

```
Intent intent = new Intent(aAvtivity.this, bActivity.class);
intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
startActivity(intent);
```

当 TaskAffinity 生效时，如已经存在相应名称的任务栈，则不会新建栈，而是在该栈的栈顶建立相应 activity；如果没有相应名称的任务栈，就会建立对应名称的新的任务栈。

#### 与 allowTaskReparenting 混用

在应用 A 中启动了应用 B 的 ActivityC，若 allowTaskReparenting 为 true，则 C 会从 A 的栈中移到 B 的任务栈。此时从 home，打开应用 B 显示的不是主界面，而是 ActivityC


最后说一下，查看当前任务栈的 adb 命令 adb shell dumpsys activity activities 

