*本文将深入源码详细介绍 binder 机制中的以下关键类：*
1. ProcessState
2. IPCThreadState
3. BpBinder
4. BinderProxy

## binder 架构

介绍之前，先简单回顾下 binder 的整体架构，大致了解这些类的角色。

对于一个比较典型的、两个应用之间的 IPC 通信流程而言：

![](https://imgkr.cn-bj.ufileos.com/cb23955f-f76b-4ea2-9b9c-d07c6257df6c.png)

Client 通过 ServiceManager 或 AMS 获取到的远程 binder 实体，一般会用 Proxy 做一层封装，比如 ServiceManagerProxy，而被封装的远程 binder 实体是一个 **BinderProxy**。

**BpBinder** 和 BinderProxy 其实是一个东西：远程 binder 实体，只不过一个 Native 层、一个 Java 层，BpBinder 内部持有了一个 binder 句柄值 handle。

**ProcessState** 是进程单例，负责打开 Binder 驱动设备及 mmap；**IPCThreadState** 为线程单例，负责与 binder 驱动进行具体的命令通信。

由 Proxy 发起 transact() 调用，会将数据打包到 Parcel 中，层层向下调用到 BpBinder ，在 BpBinder 中调用 IPCThreadState 的 transact() 方法并传入 handle 句柄值，IPCThreadState 再去执行具体的 binder 命令。

由 binder 驱动到 Server 的大概流程就是：Server 通过 IPCThreadState 接收到 Client 的请求后，层层向上，最后回调到 Stub 的 onTransact() 方法。

下面结合源码详细分析这几个类，彻底搞懂它们！*（这可能需要你较多的注意力及时间，如果暂不满足，先收藏本文咯～）*

## ProcessState

ProcessState 专门管理每个应用进程的 Binder 操作，同一个进程中只有一个 ProcessState 实例存在，且只在 ProcessState 对象创建时才打开 Binder 设备以及内存映射。相关代码如下：
```objectivec
///frameworks/native/libs/binder/ProcessState.cpp
sp<ProcessState> ProcessState::self(){
    Mutex::Autolock _l(gProcessMutex);
    if (gProcess != NULL) { //如果创建过 ProcessState 就直接返回
        return gProcess;
    }
    gProcess = new ProcessState;
    return gProcess;
}
```
外部统一通过 ProcessState::self() 方法获取 ProcessState，以此保证 ProcessState 的进程单例，ProcessState 的构造函数如下：
```objectivec
#define BINDER_VM_SIZE ((1*1024*1024) - (4096 *2))
#define DEFAULT_MAX_BINDER_THREADS 15

ProcessState::ProcessState()
    : mDriverFD(open_driver()) //打开 binder 设备
    , mVMStart(MAP_FAILED) //初始化为 MAP_FAILED，映射成功后会变更
    , mThreadCountLock(PTHREAD_MUTEX_INITIALIZER)
    , mThreadCountDecrement(PTHREAD_COND_INITIALIZER)
    , mExecutingThreadsCount(0)
    , mMaxThreads(DEFAULT_MAX_BINDER_THREADS) //binder 线程最大数量
    , mStarvationStartTimeMs(0)
    , mManagesContexts(false)
    , mBinderContextCheckFunc(NULL)
    , mBinderContextUserData(NULL)
    , mThreadPoolStarted(false)
    , mThreadPoolSeq(1){
       if (mDriverFD >= 0) { //已经成功打开 binder 驱动设备
           // 将应用进程一块虚拟内存空间与 binder 驱动映射，在此内存块上进行数据通信
           mVMStart = mmap(0, BINDER_VM_SIZE, PROT_READ, MAP_PRIVATE | MAP_NORESERVE, mDriverFD, 0);
           if (mVMStart == MAP_FAILED) { //映射失败处理
               ALOGE("Using /dev/binder failed: unable to mmap transaction memory.\n");
               close(mDriverFD);
               mDriverFD = -1;
           }
       }
}
```
ProcessState 的构造函数初始化了一些重要的变量，包括调用 open_driver() 打开 binder 设备，初始化 binder 线程最大数量，将 BINDER_VM_SIZE (接近 1M ) 的内存与 binder 驱动 mmap.

除了 ProcessState 的初始化，ProcessState 中还有一些比较重要的方法，比如 getStrongProxyForHandle()、getWeakProxyForHandle() 等，可以通过 handle 值获取对应 IBinder 对象，getWeakProxyForHandle() 方法如下：
```objectivec
wp<IBinder> ProcessState::getWeakProxyForHandle(int32_t handle){
    wp<IBinder> result;
    AutoMutex _l(mLock);
    //查找 IBinder 是否已经创建过
    handle_entry* e = lookupHandleLocked(handle);
    if (e != NULL) {
        IBinder* b = e->binder;
        if (b == NULL || !e->refs->attemptIncWeak(this)) {
            b = new BpBinder(handle); //没创建过就新建 BpBinder
            result = b;
            e->binder = b;
            if (b) e->refs = b->getWeakRefs();
        } else {
            result = b;
            e->refs->decWeak(this);
        }
    }
    return result;
}
```
lookupHandleLocked() 方法用于查找本进程中是否已经创建过要获取的 IBinder，如果没有获取到，就创建一个，lookupHandleLocked() 内部通过一个 Vector 来存放创建过的 IBinder：
```objectivec
Vector<handle_entry> mHandleToObject;

struct handle_entry{
    IBinder* binder;
    RefBase::weakref_type* refs;
}
```
如上代码所示，每个 IBinder 对象通过一个 handle_entry 结构体存放，也就是说，ProcessState 中有一个全局列表来记录所有的 IBinder 对象。

## IPCThreadState

ProcessState 对应于一个进程，是进程内单例，而 IPCThreadState 对应于一个线程，是线程单例(Thread Local)。

ProcessState 中打开了 binder 驱动、进行 mmap 映射，虽然调用了 ioctl() 函数，但主要是一些初始化配置。而具体的 BR_TRANSACTION 等命令都是由 IPCThreadState 负责执行的，当上层传来一个命令，会调用它的 transact 函数，该函数精简后如下：
```objectivec
status_t IPCThreadState::transact(int32_t handle,
                                  uint32_t code, const Parcel& data,
                                  Parcel* reply, uint32_t flags){
    //检查数据是否有效
    status_t err = data.errorCheck();
    if (err == NO_ERROR) {
        //将数据打包塞到 mOut 里
        err = writeTransactionData(BC_TRANSACTION, flags, handle, code, data, NULL);
    }
    if ((flags & TF_ONE_WAY) == 0) { //不是 one way 调用，需要等待回复
        if (reply) {
            err = waitForResponse(reply);
        } else {
            Parcel fakeReply;
            err = waitForResponse(&fakeReply);
        }
    } else { //one way 调用，不用等待回复
        err = waitForResponse(NULL, NULL);
    }
    return err;
}
```
IPCThreadState 中有 mIn、mOut 两个 Parcel 数据，mIn 用来存放从别处读取而来的数据，mOut 存放要写入到别处的数据，在 writeTransactionData() 方法中将数据存放到 mOut，准备写入到 binder 驱动。

waitForResponse() 方法去实际执行写入到 binder 驱动，简化后的 waitForResponse() 方法如下：
```objectivec
status_t IPCThreadState::waitForResponse(Parcel *reply, status_t *acquireResult){
    uint32_t cmd;
    int32_t err;
    while (1) {
        //进一步调用 talkWithDriver 去执行写入数据到 binder 驱动
        if ((err=talkWithDriver()) < NO_ERROR) break;
        err = mIn.errorCheck(); //检查数据有效性
        if (err < NO_ERROR) break;
        if (mIn.dataAvail() == 0) continue; //检查数据有效性
        cmd = (uint32_t)mIn.readInt32(); //拿到 binder 驱动发过来的命令
        switch (cmd) {
            //处理命令
            case BR_TRANSACTION_COMPLETE:{...}
            case BR_DEAD_REPLY:{...}
            case BR_FAILED_REPLY:{...}
            case BR_ACQUIRE_RESULT:{...}
            case BR_REPLY:{...}
            default:
                //其他命令在 executeCommand 方法中处理
                err = executeCommand(cmd);
                if (err != NO_ERROR) goto finish;
                break;
            }
    }
    return err;
}
```
可以看到 waitForResponse() 中并没有直接执行写入数据到 binder，而是进一步调用 talkWithDriver 去处理，随后 waitForResponse() 方法处理了由 binder 驱动发送过来的命令，比如 BR_TRANSACTION_COMPLETE ：
```objectivec
case BR_TRANSACTION_COMPLETE:
       if (!reply && !acquireResult) goto finish;
       break;
```
在 transact() 方法中判断如果是 one way 调用，reply 及 acquireResult 都传入 NULL，所以上面条件成立，直接退出循环，不用再等待 binder 驱动的回复。

到目前为止，由 transact() 到 waitForResponse()，已经将要发送的数据准备好，并对后续 binder 驱动的回复也做了处理，但还没看到真正写入数据给 binder 驱动的代码，但已经知道就在 talkWithDriver() 方法中，此方法中主要做了三个工作：

1. 准备 binder_write_read 数据
2. 写入 binder 驱动
3. 处理驱动回复

以此将 talkWithDriver() 代码简化分为对应的三部分来看，首先是准备 binder_write_read 数据：
```objectivec
status_t IPCThreadState::talkWithDriver(bool doReceive){
    binder_write_read bwr; //binder 驱动接受的数据格式
    const bool needRead = mIn.dataPosition() >= mIn.dataSize();
    const size_t outAvail = (!doReceive || needRead) ? mOut.dataSize() : 0;
    bwr.write_size = outAvail; //要写入的数据量
    bwr.write_buffer = (uintptr_t)mOut.data(); //要写入的数据
    // This is what we'll read.
    if (doReceive && needRead) {
        bwr.read_size = mIn.dataCapacity(); //要读取的数据量
        bwr.read_buffer = (uintptr_t)mIn.data(); //存放读取数据的内存空间
    } else {
        bwr.read_size = 0;
        bwr.read_buffer = 0;
    }
    // 如果不需要读也不需要写，那就直接返回
    if ((bwr.write_size == 0) && (bwr.read_size == 0)) return NO_ERROR;
```
在 IPCThreadState.h 中声明了 talkWithDriver() 方法的参数 doReceive 默认为 true，waitForResponse() 中没有传入参数，所以这里的 doReceive 为 true。

binder_write_read 是 binder 驱动与用户态共用的、存储读写操作的数据，在 binder 驱动内部依赖 binder_write_read 决定是要读取还是写入数据：其内部变量 read_size>0 则代表要读取数据，write_size>0 代表要写入数据，若都大于 0 则先写入，后读取。

准备好 binder_write_read 后，再来看是怎么写入 binder 驱动的，其实很简单，真正执行写入的操作就一行代码：
```objectivec
    ioctl(mProcess->mDriverFD, BINDER_WRITE_READ, &bwr)
```
对应的会调用到 binder 驱动的 binder_ioctl() 函数，这里不延伸此函数，接着看 talkWithDriver() 方法的第三个工作，处理驱动的回复：
```objectivec
     if (bwr.write_consumed > 0) { //成功写入了数据
         if (bwr.write_consumed < mOut.dataSize())
             mOut.remove(0, bwr.write_consumed);
         else
             mOut.setDataSize(0);
     }
     if (bwr.read_consumed > 0) { //成功读取到了数据
         mIn.setDataSize(bwr.read_consumed);
         mIn.setDataPosition(0);
     }
     return NO_ERROR;
}
```
bwr.write_consumed > 0 代表 binder 驱动消耗了 mOut 中的数据，所以要把这部分已经处理过的数据移除调；bwr.read_consumed > 0 代表 binder 驱动成功的返回了数据给我们，并写入了上面通过 bwr.read_buffer 指定的内存地址，即 mIn 中，所以要对 mIn 对相关的修正。

到这里 talkWithDriver 执行完毕，读取到的数据放到了 mIn 中，也正好对应于上面 waitForResponse() 方法中从 mIn 中取数据的逻辑。

## BpBinder

上文介绍 ProcessState 中的 getWeakProxyForHandle() 方法时，构造了一个 BpBinder 对象返回：
```objectivec
new BpBinder(handle)
```
IPCThreadState 作为主要与 binder 驱动交互的对象，它的 transact 方法第一个参数就是 handle 值：
```objectivec
status_t IPCThreadState::transact(int32_t handle,
                                  uint32_t code, const Parcel& data,
                                  Parcel* reply, uint32_t flags)
```
注意这两个线索：一是将 handle 交给 BpBinder 持有，二是在调用 IPCThreadState transact 方法时需要传入 handle，这意味着什么呢？*一个 BpBinder 对象就是关联了一个远程 handle 的操作封装，其内部是通过 IPCThreadState 来实现的* 。但这个仅是猜想，下面通过 BpBinder 源码来验证是否属实，首先是构造函数：
```objectivec
BpBinder::BpBinder(int32_t handle)
   : mHandle(handle)
   , mAlive(1)
   , mObitsSent(0)
   , mObituaries(NULL)

   ALOGV("Creating BpBinder %p handle %d\n", this, mHandle);

   extendObjectLifetime(OBJECT_LIFETIME_WEAK);
   IPCThreadState::self()->incWeakHandle(handle);
}
```
智能指针的 OBJECT_LIFETIME_WEAK 配置，代表 BpBinder 对象的强计数器和弱计数器的值都为 0 时才会被销毁。另外可以看到通过内部变量 mHandle 持有 handle 值，在 BpBinder 的 transact 方法中使用了 mHandle：
```objectivec
status_t BpBinder::transact(
    uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags){
    if (mAlive) {
        status_t status = IPCThreadState::self()->transact(
            mHandle, code, data, reply, flags);
        if (status == DEAD_OBJECT) mAlive = 0;
        return status;
    }
    return DEAD_OBJECT;
}
```
其内部确实是调用了 IPCThreadState 的 transact 方法，这便验证了 *一个 BpBinder 对象就是关联了一个远程 handle 的操作封装，其内部是通过 IPCThreadState 来实现的* 的描述是正确的。

## BinderProxy

先给出结论：BinderProxy 就是 BpBinder，"BpBinder" 中的 "p" 即 Proxy，只不过 BpBinder 是 Native 层的，BinderProxy 是 Java 层的。BinderProxy 和 BpBinder 分别继承自 Java 和 Native 层的 IBinder 接口，即 IBinder.h 和 IBinder.java，它们可以看作同一个接口，都定义了 transact 等方法。

下面根据源码来验证这个结论，ServiceManager.java 中获取 Service Manager 的代码如下：
```java
    private static IServiceManager getIServiceManager() {
        if (sServiceManager != null) {
            return sServiceManager;
        }
        // Find the service manager
        sServiceManager = ServiceManagerNative.asInterface(BinderInternal.getContextObject());
        return sServiceManager;
    }
```
其中 asInterface 方法接收的参数就是一个 IBinder 类型对象，可想而知，BinderInternal 的 getContextObject() 方法返回的是一个 BinderProxy 对象：
```java
    /**
     * Return the global "context object" of the system.  This is usually
     * an implementation of IServiceManager, which you can use to find
     * other services.
     */
    public static final native IBinder getContextObject();
```
这里保留了注释，为什么将 Service Manager 命名为 Context Object 呢？因为每个进程都需要 IPC 操作，IPC 是作为进程的基础配置存在的。上面代码的注释更直接的描述了此方法，有助于我们理解。

接着看这个方法，对应的 native 实现在 android_util_Binder.cpp 中：
```objectivec
static jobject android_os_BinderInternal_getContextObject(JNIEnv* env, jobject clazz){
    sp<IBinder> b = ProcessState::self()->getContextObject(NULL);
    return javaObjectForIBinder(env, b);
}
```
ProcessState 的 getContextObject() 方法如下：
```objectivec
sp<IBinder> ProcessState::getContextObject(const sp<IBinder>& /*caller*/){
    return getStrongProxyForHandle(0);
}
```
上文介绍过 ProcessState 的 getWeakProxyForHandle() 方法，其内部构造了一个 BpBinder 对象返回，getStrongProxyForHandle() 方法跟 getWeakProxyForHandle() 一样，也是返回了一个 BpBinder 对象，只不过是强引用类型。

javaObjectForIBinder() 方法不再展开，它根据 BpBinder 对象构造了一个 BinderProxy 对象，并且记录了 BpBinder 的内存地址，以便后续从 Java->Native 时，可以根据 BinderProxy 获取到对应的 BpBinder 对象。

与 javaObjectForIBinder() 对应， 由 BinderProxy -> BpBinder 调用的是  android_util_Binder.cpp 的 ibinderForJavaObject() 方法。

# 最后

不要畏难，只要我们耐心不懈的对其单点突破、反复琢磨，再复杂的机制也只是纸老虎！
