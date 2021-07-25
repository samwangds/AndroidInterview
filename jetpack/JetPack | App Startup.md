# JetPack 开篇 | App Startup 的使用及原理

## JetPack

Jetpack 包含一系列 Android 库，它们都采用最佳做法并在 Android 应用中提供向后兼容性。

另外 jet pack 词组意思是：喷气发动机组件。谷歌起这个名字，应该想说用了这些库我们的应用就能起飞吧~

今天挖个新坑，后面我们会补全 JetPack 系列的各个库的使用及原理

## Startup

Startup 启动；作为我们新坑的开篇再适合不过了。([旧坑-设计模式](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzIzOTkwMDY5Nw==&action=getalbum&album_id=1398049733876514816&scene=173&from_msgid=2247488242&from_itemidx=1&count=3&nolastread=1#wechat_redirect))

顾名思义，Startup 是一个可在应用启动时简单、高效地初始化的组件。

## How

[App Startup](https://developer.android.google.cn/topic/libraries/app-startup) 在应用启动时提供一种简单、高效的方法来初始化组件。库开发者和应用开发者都可以使用 App Startup 简化启动流程，并显式指定初始化顺序。

我们来看下如何使用 Startup 组件

#### 1. 添加依赖

```groovy
// build.gradle (project 根目录下)
buildscript {
    repositories {
        google()
        jcenter()
    }
}

// app/build.gradle
dependencies {
    implementation "androidx.startup:startup-runtime:1.0.0"
}
```

#### 2. 定义 Initalizer

通过创建接口 `androidx.startup.Initializer<T>` 的实现类来为每个组件定一个初始化器（Initalizer）。这个接口有两个方法：

```JAVA
public interface Initializer<T> {
		// create 方法包含初始化这个组件需要的所有操作并返回一个 T 的实例
    @NonNull
    T create(@NonNull Context context);

    // 返回这个初始化器所依赖的所有初始化器列表
    // 可以通过这个方法来控制组件的初始化顺序
    @NonNull
    List<Class<? extends Initializer<?>>> dependencies();
}
```

我们来写个 Hello World，我们有两个组件 HelloComponent,  WorldComponent.  

代码如下：

```kotlin
class WorldComponent private constructor() {
    val msg = "World"
    companion object{
        lateinit var instance: WorldComponent
        fun create() :WorldComponent{
            instance = WorldComponent()
            return instance
        }
    }
}

// 依赖于 WorldComponent
class HelloComponent private constructor() {
    private lateinit var world: WorldComponent
    private val msg = "Hello"
    fun print() {
        Log.i("Startup","$msg ${world.msg}")
    }
    companion object {
        lateinit var instance: HelloComponent
        fun create() :HelloComponent{
            instance = HelloComponent()
            instance.world = WorldComponent.instance
            return instance
        }
    }
}
```

要先发现 World 才能 Say Hello，所以 HelloComponent 依赖于 WorldComponent. 为这两个组件创建初始化器

``` kotlin
class WorldInitializer : Initializer<WorldComponent> {
    override fun create(context: Context): WorldComponent {
        return WorldComponent()
    }

    override fun dependencies(): MutableList<Class<out Initializer<*>>> {
        return mutableListOf() // 无任何依赖
    }
}

class HelloInitializer : Initializer<HelloComponent> {
    override fun create(context: Context): HelloComponent {
        return HelloComponent.create()
    }

    override fun dependencies(): MutableList<Class<out Initializer<*>>> {
        // 依赖 WorldInitializer
        return mutableListOf(WorldInitializer::class.java)
    }
}
```

这样 startup 就会先初始化 WorldComponent 再初始化 HelloComponent

#### 3. 在 Manifest 中列出初始化器

```xml
 <provider
     android:name="androidx.startup.InitializationProvider"
     android:authorities="${applicationId}.androidx-startup"
     android:exported="false"
     tools:node="merge">
     <!-- This entry makes HelloInitializer discoverable. -->
     <meta-data  android:name="com.sam.hellojetpack.startup.HelloInitializer"
         android:value="androidx.startup" />
 </provider>
```

此处我们不需要增加 `WorldInitializer` 的 `<meta-data>` 项, 因为 HelloInitializer 依赖于 WorldInitializer，这就表示如果 HelloInitializer 被发现，那同时 WorldInitializer 也会被发现。

Startup 库里有一条 lint rule, 用来检查是否正确的配置了全部的 `Initializer`. 直接在命令行执行`./gradlew :app:lintDebug`即可。

## 手动初始化/延迟初始化

一般来讲，使用 App Startup 库时，`InitializationProvider` 会调用 `AppInitializer` 自动发现和执行 Initializer。

然而，我们也可以直接使用`AppInitializer` 来手动初始化那些你不需要在应用一启动就初始化的组件。这就是延迟初始化，有利于缩短应用的启动时间。

#### 禁用个别组件的自动初始化

禁用一个组件的自动初始化，需要在 Manifest将这个组件的`<meta-data>`条目删除，如果是自己编写的 Initializer，那不要加到 manifest 里即可。

如果是第三方库里的 Initializer 可以使用如下方式：

```XML
<provider
    android:name="androidx.startup.InitializationProvider"
    android:authorities="${applicationId}.androidx-startup"
    android:exported="false"
    tools:node="merge">
    <meta-data android:name="com.example.ExampleLoggerInitializer"
              tools:node="remove" />
</provider>
```

使用 ` tools:node="remove"`，可以保证 merger tool 也从其它合并的 Manifest 中移除这个条目。

注意：移除一个 Initializer 那它依赖的 Initializer 同时也会被移除。这也很好理解，原来它依赖的 Initializer 没列在 Manifest 里面，而是通过依赖关系被发现的。

#### 禁用所有组件自动初始化

因为自动初始化是通过 `InitializationProvider`  来调用的，所以只需要在 Manifest 中移除这个 ContentProvider 条目即可

```XML
<provider
    android:name="androidx.startup.InitializationProvider"
    android:authorities="${applicationId}.androidx-startup"
    tools:node="remove" />
```

#### 手动初始化

我们禁用了自动初始化后，需要在合适的位置手动调用 `AppInitializer` 来初始化 Initializer 以及它依赖的 Initializer 

我们前面的 HelloWorld 手动初始化的代码如下：

````kotlin
AppInitializer.getInstance(applicationContext)
    .initializeComponent(HelloInitializer::class.java)
````

在自动初始化或手动初始化之后，我们就可以使用 HelloWorld 组件了。调用相关的测试方法：

```kotLin
HelloComponent.instance.print()

// 输出结果如下：
.../com.sam.hellojetpack I/Startup: Hello World
```

## Why

得益于共享一个 Content Provider 来初始化各个组件，在需要初始化较多组件时，App Startup 可以显著减少应用的启动时间。我们来看下这是怎么实现的。

#### 初始化流程

App Startup 使用了一个名为 InitializationProvider 的 ContentProvider。

相关源码：

```JAVA
// androidx.startup.InitializationProvider
public final class InitializationProvider extends ContentProvider {
    @Override
    public boolean onCreate() {
        Context context = getContext();
        if (context != null) {
            AppInitializer.getInstance(context).discoverAndInitialize();
        } else {
            throw new StartupException("Context cannot be null");
        }
        return true;
    }
}
```

该 ContentProvider 在合并后的 AndroidManifest.xml 文件中查找条目来发现 Initializer。

```JAVA
// androidx.startup.AppInitializer
void discoverAndInitialize() {
    ComponentName provider = new ComponentName(mContext.getPackageName(),
            InitializationProvider.class.getName());
    ProviderInfo providerInfo = mContext.getPackageManager()
            .getProviderInfo(provider, GET_META_DATA);
    Bundle metadata = providerInfo.metaData;
    String startup = mContext.getString(R.string.androidx_startup);
    if (metadata != null) {
        Set<Class<?>> initializing = new HashSet<>();
        Set<String> keys = metadata.keySet();
        for (String key : keys) {
            String value = metadata.getString(key, null);
            if (startup.equals(value)) {
                Class<?> clazz = Class.forName(key);
                // clazz is a Initializer.class
                if (Initializer.class.isAssignableFrom(clazz)) {
                    Class<? extends Initializer<?>> component =
                            (Class<? extends Initializer<?>>) clazz;
                    mDiscovered.add(component);
                    doInitialize(component, initializing);
                }
            }
        }
    }
}
```

找到各个 Initializer 就可以执行初始化方法了：

```java
// androidx.startup.AppInitializer
<T> T doInitialize(
       Class<? extends Initializer<?>> component,
       Set<Class<?>> initializing) {
     Object result;
     if (!mInitialized.containsKey(component)) {
         initializing.add(component);
         Object instance = component.getDeclaredConstructor().newInstance();
         Initializer<?> initializer = (Initializer<?>) instance;
         // 找到依赖的 Initializer
         List<Class<? extends Initializer<?>>> dependencies =
                 initializer.dependencies();

         if (!dependencies.isEmpty()) {
             for (Class<? extends Initializer<?>> clazz : dependencies) {
                 if (!mInitialized.containsKey(clazz)) {
                     // 优先初始化依赖的 Initializer
                     doInitialize(clazz, initializing);
                 }
             }
         }
         
       	 // 执行初始化
         result = initializer.create(mContext);
         // 避免重复加载
         initializing.remove(component);
         mInitialized.put(component, result);
     } else {
         result = mInitialized.get(component);
     }
     return (T) result;
}
```



#### 更进一步

ContentProvider.onCreate 什么时候被调用的？

 应用创建 Application 是通过调用 ActivityThread.handleBindApplication 方法，这个方法的相关流程有：

- 创建 Application
- 初始化 Application 的 Context
- 调用 installContentProviders 并传入刚创建好的 Application 来创建 ContentProvider

我们就从这边往下看：

```JAVA
// android.app.ActivityThread
private void installContentProviders(
        Context context, List<ProviderInfo> providers) {
  ...
  for (ProviderInfo cpi : providers) {
      ContentProviderHolder cph = installProvider(context, null, cpi,
              false /*noisy*/, true /*noReleaseNeeded*/, true /*stable*/);
      if (cph != null) {
          cph.noReleaseNeeded = true;
      }
  }
  ...
}
```

遍历调用 installProvider 初始化单个 ContentProvider

```java
private ContentProviderHolder installProvider(Context context,
        ContentProviderHolder holder, ProviderInfo info,
        ...) {
  ...
  ContentProvider localProvider = null;
  final java.lang.ClassLoader cl = c.getClassLoader();
  localProvider = packageInfo.getAppFactory()
          .instantiateProvider(cl, info.name);
  provider = localProvider.getIContentProvider();
  localProvider.attachInfo(c, info);
  ...
}
```

实例化 ContentProvider 调用 attachInfo 来绑定相关信息

```JAVA
// android.content.ContentProvider
private void attachInfo(Context context, ProviderInfo info, boolean testing) {
    /*
     * Only allow it to be set once, so after the content service gives
     * this to us clients can't change it.
     */
    if (mContext == null) {
        mContext = context;
        if (info != null) {
            setReadPermission(info.readPermission);
            setWritePermission(info.writePermission);
            setPathPermissions(info.pathPermissions);
            mExported = info.exported;
            mSingleUser = (info.flags & ProviderInfo.FLAG_SINGLE_USER) != 0;
            setAuthorities(info.authority);
        }
        ContentProvider.this.onCreate();
    }
}
```

最后调用了 ContentProvider.onCreate(), 它的子类 InitializationProvider 在 onCreate 的时候便开始了 startup 组件的流程

[源码](https://github.com/samwangds/HelloJetPack)

参考链接 [App Startup](https://developer.android.google.cn/topic/libraries/app-startup)

