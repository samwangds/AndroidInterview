## 背景

在做应用启动速度优化时，需先了解启动阶段做了哪些耗时任务，分析 Application 的 attachBaseContext、onCreate 等关键方法，统计它们内部调用到的其他方法耗时。

分析要结合 systrace 工具，因为不仅要知道方法的 wall time，还要知道 cpu time，这样才能知道是否属于 cpu 密集型任务，然后针对任务类型进行调整或线程调度。

需求很清晰，在要统计的方法调用前插桩加入 TraceCompat.beginSection()，调用后加入 TraceCompat.endSection()。需求也很简单，我们可以很快的使用 aspect、javassist 或 asm 实现。

但是，这次是方法插桩 + systrace，需为此开发一个插件；下次是方法插桩 + 耗时统计，就得再开发一个插件。为什么插件一定要与插桩逻辑绑定呢？

**为什么没有一款插件，只提供方法插桩能力，不写死插桩逻辑，而是由使用者自由的定制插桩逻辑呢？**

基于这个痛点，我们来开发一款可自由定制插桩逻辑的插件。

## AOP 方案选择

首先是 AOP 技术方案的选择，aspect、javassist 还是 asm ？ 在思考了一秒钟后，我决定选择 asm，理由很简单：性能高、逼格高。

那要基于 gradle 及 asm 原生 API，从零开发吗？ NO，有些轮子不能造，我们要当坐在马车上驰骋的人！

所以我最终选择基于 ByteX 开发。

ByteX 与 Jetpack StartUp 有异曲同工之妙。

Startup 针对多个三方库各自使用 ContentProvider 初始化导致拖慢启动速度的问题，提供了一个 ContentProvider 来集中运行所有依赖项的初始化；ByteX 针对多个功能插件各自进行 transform 导致拖慢编译速度的问题，提供了一个宿主插件 transform，集中处理所有的 transform。

ByteX 对 Transform 及 ASM 相关 API 做了封装，大大节省了插件开发的工作量，我们无需处理 class/jar 的 IO 操作，只需关注想要进行的 hook 逻辑即可。

所以，通过性能及开发成本两个维度的考量，基于 ByteX 开发一些有意义的插件，是一个不错的选择。

## trace-plugin 插件

成果先行，目前 trace-plugin 已开发完并发布，见：https://github.com/yhaolpz/ByteXPlugin/tree/master/trace，可查看插件源码及接入方式。

使用姿势很简单，仅 @TraceClass、@TraceMethod 两个注解而已：

#### @TraceClass 为类注解，可配置：

```java
//指定方法插桩实现类
Class methodTrace() default TimeTrace.class;
//是否要追踪此类中所有方法
boolean traceAllMethod() default false;
//是否要追踪方法内部调用到的方法
boolean traceInnerMethod() default false;
```

#### @TraceMethod 为方法注解，可配置：

```java
//是否要追踪此方法
boolean trace() default true;
//是否要追踪方法内部调用到的方法
boolean traceInnerMethod() default false;
```

**举个例子**

Test 类中有 m1()、m2()、m3() 三个方法：

```java
public class Test{
    public static void m1() {
        m2();
        OtherClass.m4();
    }
    public static void m2() {
    }
    public static void m3() {
    }
}
```

*追踪 m1() 耗时：*

```java
@TraceClass
public class Test{
    @TraceMethod
    public static void m1() {...
```

*追踪类中所有方法耗时：*

```java
@TraceClass(traceAllMethod = true)
public class Test{...
```

*追踪类中所有方法耗时，但不包括 m1()：*

```java
@TraceClass(traceAllMethod = true)
public class Test{
    @TraceMethod(trace = false)
    public static void m1() {...
```

*追踪 m1() 方法内部调用到的方法，即 m2()、OtherClass.m4() 的耗时：*

```java
@TraceClass
public class Test{
    @TraceMethod(trace = false,traceInnerMethod = true)
    public static void m1() {
        m2();
        OtherClass.m4();
    }
    ...
```

*自定义追踪插桩处理：*

```java
//继承自 IMethodTrace 方法实现自己的插桩处理，例如 systrace 追踪处理：
public class CustomSysTrace implements IMethodTrace {
    @Override
    public void onMethodEnter(String className, String methodName, String methodDesc, String outerMethod) {
        TraceCompat.beginSection(className + "#" + methodName);
    }
    @Override
    public void onMethodEnd(String className, String methodName, String methodDesc, String outerMethod) {
        TraceCompat.endSection();
    }
}

//在类注解中指定即可
@TraceClass(methodTrace = CustomSysTrace.class)
public class Test{...}
```

## 自定义插桩处理实现原理

其实非常简单，插件内部对于需要插桩的方法会统一调用到 TraceRecord 类进行处理：

```java
public class TraceRecord {
    //插桩方法执行前会统一调到这里
    public static void onMethodEnter(String traceImplClass,
                                     String className,
                                     String methodName, String methodDesc,
                                     String outerMethod) {
        getMethodTrace(traceImplClass).onMethodEnter(className,
                                methodName, methodDesc, outerMethod);
    }
    //插桩方法执行完后会统一调到这里
    public static void onMethodEnd(String traceImplClass,
                                   String className,
                                   String methodName, String methodDesc,
                                   String outerMethod) {
        getMethodTrace(traceImplClass).onMethodEnd(className,
                                methodName, methodDesc, outerMethod);
    }
}
```

traceImplClass 就是我们在类注解中指定的自定义插桩逻辑实现类，比如 CustomSysTrace ，然后在 getMethodTrace() 中实例化：

```java
private static IMethodTrace getMethodTrace(String traceImplClass) {
    IMethodTrace methodTrace = sMethodTraceMap.get(traceImplClass);
    if (methodTrace == null) {
        try {
            methodTrace = (IMethodTrace) Class.forName(traceImplClass).newInstance();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
    sMethodTraceMap.put(traceImplClass, methodTrace);
    return methodTrace;
}
```

## 最后

有了简洁、优雅、易用、功能强大的 trace-plugin 插件，以后再也不怕方法插桩了，你想插什么就插什么。