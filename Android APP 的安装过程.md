今天我们一起来看一下，系统是怎么把应用安装到你的手机上的。

## 入口

当我们点击一个 APK 进行安装时，会弹出以下界面

![](img/image-20200907203547588.png)

这是 Android Framework 提供的软件包安装程序，页面为： PackageInstallerActivity

```JAVA
package com.android.packageinstaller;

public class PackageInstallerActivity extends Activity {
    public void onClick(View v) {
        if (v == mOk) {
            if (mOk.isEnabled()) {
                //...省略一些细节
                startInstall(); // 开始安装
            }
        } else if (v == mCancel) {
            // Cancel and finish
        }
    }
  
    private void startInstall() {
        // Start subactivity to actually install the application
        Intent newIntent = new Intent();
        newIntent.putExtra(PackageUtil.INTENT_ATTR_APPLICATION_INFO,
                mPkgInfo.applicationInfo);
        newIntent.setData(mPackageURI);
        newIntent.setClass(this, InstallInstalling.class);
        ...// newIntent.putExtra 其它参数 
        startActivity(newIntent);
        finish();
    }
}
```

在这个页面点击安装时，会把安装包的信息通过 Intent 传递到 InstallInstalling 这个 Activity.  InstallInstalling 的作用主要是向 PMS 发送包信息以及处理回调。

#### InstallInstalling.onCreate

进来新页面，当然是先从 onCreate 开始了

```java
protected void onCreate(@Nullable Bundle savedInstanceState) {
    ApplicationInfo appInfo = getIntent()
                .getParcelableExtra(PackageUtil.INTENT_ATTR_APPLICATION_INFO);
    mPackageURI = getIntent().getData();
    ...  
    // 根据 mPackageURI 创建一个对应的 File
    final File sourceFile = new File(mPackageURI.getPath());
    // 显示应用信息 icon, 应用名或包名
    PackageUtil.initSnippetForNewApp(this, PackageUtil.getAppSnippet(this, appInfo,
          sourceFile), R.id.app_snippet); 
    
    // 创建、组装 SessionParams，它用来携带会话的参数
    PackageInstaller.SessionParams params = new PackageInstaller.SessionParams(
                        PackageInstaller.SessionParams.MODE_FULL_INSTALL);
    params.installFlags = PackageManager.INSTALL_FULL_APP;
    ...
    params.installerPackageName =
            getIntent().getStringExtra(Intent.EXTRA_INSTALLER_PACKAGE_NAME);

    File file = new File(mPackageURI.getPath());
    // 对 APK 进行轻量级的解析，并将解析的结果赋值给 SessionParams 相关字段 
    PackageParser.PackageLite pkg = PackageParser.parsePackageLite(file, 0);
    params.setAppPackageName(pkg.packageName);
    params.setInstallLocation(pkg.installLocation);
    params.setSize(
            PackageHelper.calculateInstalledSize(pkg, false, params.abiOverride));
    // 向 InstallEventReceiver 注册一个观察者返回一个新的 mInstallId
    // InstallEventReceiver 是一个 BroadcastReceiver，可以通过 EventResultPersister 接收到所有的安装事件
    // 这里事件会回调给 this::launchFinishBasedOnResult
    mInstallId = InstallEventReceiver
                .addObserver(this, EventResultPersister.GENERATE_NEW_ID,
                        this::launchFinishBasedOnResult);

    try {
        // PackageInstaller 的 createSession 
        // 方法内部会通过 IPackageInstaller 与 PackageInstallerService进行进程间通信，
        // 最终调用的是 PackageInstallerService 的 createSession 方法来创建并返回 mSessionId
        mSessionId = getPackageManager().getPackageInstaller().createSession(params);
    } catch (IOException e) {
        launchFailure(PackageManager.INSTALL_FAILED_INTERNAL_ERROR, null);
    }
   
  
}
```

#### InstallInstalling.onResume

接下来是 onResume, 通过 InstallingAsyncTask 做一些异步工作

```JAVA
protected void onResume() {
    super.onResume();

    // This is the first onResume in a single life of the activity
    if (mInstallingTask == null) {
        PackageInstaller installer = getPackageManager().getPackageInstaller();
        PackageInstaller.SessionInfo sessionInfo = installer.getSessionInfo(mSessionId);

        if (sessionInfo != null && !sessionInfo.isActive()) {
            mInstallingTask = new InstallingAsyncTask();
            mInstallingTask.execute();
        } else {
            // we will receive a broadcast when the install is finished
            mCancelButton.setEnabled(false);
            setFinishOnTouchOutside(false);
        }
    }
}
```

#### InstallingAsyncTask

```JAVA
private final class InstallingAsyncTask extends AsyncTask<Void, Void,
            PackageInstaller.Session> {

        @Override
        protected PackageInstaller.Session doInBackground(Void... params) {
            PackageInstaller.Session session;
            session = getPackageManager().getPackageInstaller().openSession(mSessionId);
            session.setStagingProgress(0);

            File file = new File(mPackageURI.getPath());
 	          OutputStream out = session.openWrite("PackageInstaller", 0, sizeBytes)
            InputStream in = new FileInputStream(file)
            long sizeBytes = file.length();
            byte[] buffer = new byte[1024 * 1024];

            while (true) {
                int numRead = in.read(buffer);
                if (numRead == -1) {
                    session.fsync(out);
                    break;
                }
               // 将 APK 文件通过 IO 流的形式写入到 PackageInstaller.Session 中
                out.write(buffer, 0, numRead);
                if (sizeBytes > 0) {
                    float fraction = ((float) numRead / (float) sizeBytes);
                    session.addProgress(fraction);
                }
             }
             return session; 
        }

        @Override
        protected void onPostExecute(PackageInstaller.Session session) { 
            Intent broadcastIntent = new Intent(BROADCAST_ACTION);
            broadcastIntent.setFlags(Intent.FLAG_RECEIVER_FOREGROUND);
            broadcastIntent.setPackage(
                    getPackageManager().getPermissionControllerPackageName());
            broadcastIntent.putExtra(EventResultPersister.EXTRA_ID, mInstallId);

            PendingIntent pendingIntent = PendingIntent.getBroadcast(
                    InstallInstalling.this,
                    mInstallId,
                    broadcastIntent,
                    PendingIntent.FLAG_UPDATE_CURRENT);
            // 调用 PackageInstaller.Session 的 commit 方法，进行安装
            session.commit(pendingIntent.getIntentSender()); 
        }
    }
```

来看下 PackageInstaller.Session 里的实现

```java
public static class Session implements Closeable {
    private IPackageInstallerSession mSession;
    public void commit(@NonNull IntentSender statusReceiver) {
        try {
            mSession.commit(statusReceiver, false);
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }
}
```

mSession 的类型为 IPackageInstallerSession，这说明要通过 IPackageInstallerSession 来进行进程间的通信，最终会调用PackageInstallerSession 的 commit 方法，剧透一下在这个类执行完后，就会进入鼎鼎大名的 PMS 去真正的执行安装了 ：

```java
public class PackageInstallerSession extends IPackageInstallerSession.Stub {
    public void commit(@NonNull IntentSender statusReceiver, boolean forTransfer) {
        // 将包的信息封装为 PackageInstallObserverAdapter
        final PackageInstallObserverAdapter adapter = new PackageInstallObserverAdapter(
          mContext, statusReceiver, sessionId,
          isInstallerDeviceOwnerOrAffiliatedProfileOwnerLocked(), userId);
        mRemoteObserver = adapter.getBinder();
        // 通过 Handler 处理消息事件
        mHandler.obtainMessage(MSG_COMMIT).sendToTarget();
    }
  
    private final Handler.Callback mHandlerCallback = new Handler.Callback() {
        @Override
        public boolean handleMessage(Message msg) {
            switch (msg.what) {
                case MSG_COMMIT:
                    commitLocked();
                    break;
            }
        }
    };
  
    private final PackageManagerService mPm;
    private void commitLocked() throws PackageManagerException {
        mPm.installStage(mPackageName, stageDir, ...);
    }
  
}
```
mPm 就是系统服务 PackageManagerService。installStage 方法就是正式开始 apk 的安装过程。这个过程包括两大步：拷贝安装包、装载代码

## 拷贝安装包

继续看 installStage 的代码
```JAVA
// PackageManagerService.java
void installStage(String packageName, File stagedDir,...) {
    final Message msg = mHandler.obtainMessage(INIT_COPY);
    // 把之前传入的 sessionParams 安装信息，及其它信息封装成 InstallParams
    final InstallParams params = new InstallParams(origin, null, observer,
        sessionParams.installFlags, installerPackageName, sessionParams.volumeUuid,
        verificationInfo, user, sessionParams.abiOverride,
        sessionParams.grantedRuntimePermissions, signingDetails, installReason);
    mHandler.sendMessage(msg);
}
```

发送的消息 INIT_COPY 从名字上就知道是去初始化复制

```java
class PackageHandler extends Handler {
    void doHandleMessage(Message msg) {
        switch (msg.what) {
            case INIT_COPY: {
                HandlerParams params = (HandlerParams) msg.obj;
                // 调用 connectToService 方法连接安装 apk 的 Service 服务。
                if (!connectToService()) {
                    return;
                } else {
                     // Once we bind to the service, the first
                     // pending request will be processed.
                     mPendingInstalls.add(idx, params);
                }
            }
        }
    }

    private boolean connectToService() {
         // 通过隐式 Intent 绑定 Service，实际绑定的 Service 是 DefaultContainerService 
         Intent service = new Intent().setComponent(DEFAULT_CONTAINER_COMPONENT);
         if (mContext.bindServiceAsUser(service, mDefContainerConn,
                 Context.BIND_AUTO_CREATE, UserHandle.SYSTEM)) {
             mBound = true;
             return true;
         }
         return false;
   }
}
```

当绑定 Service 成功之后，会在 mDefContainerConn 的 onServiceConnection 方法中发送一个绑定操作的 Message，如下所示：

```JAVA
class DefaultContainerConnection implements ServiceConnection {
    public void onServiceConnected(ComponentName name, IBinder service) {
        final IMediaContainerService imcs = IMediaContainerService.Stub
                .asInterface(Binder.allowBlocking(service));
        mHandler.sendMessage(mHandler.obtainMessage(MCS_BOUND, imcs));
    }
}

// MCS_BOUND 还是在前面的 PackageHandler 处理，直接截取相关代码
{
    HandlerParams params = mPendingInstalls.get(0);
    if (params.startCopy()) {
        if (mPendingInstalls.size() > 0) {
            mPendingInstalls.remove(0);
        }
    }
}
```

mPendingInstalls 是一个等待队列，里面保存所有需要安装的 apk 解析出来的 HandlerParams 参数（前面在 INIT_COPY 处理时 add），从 mPendingInstalls 中取出第一个需要安装的 HandlerParams 对象，并调用其 startCopy 方法，在 startCopy 方法中会继续调用一个抽象方法 handleStartCopy 处理安装请求。通过之前的分析，我们知道 HandlerParams 实际类型是 InstallParams 类型，因此最终调用的是 InstallParams 的 handlerStartCopy 方法，这是整个安装包拷贝的核心。

```JAVA
class InstallParams extends HandlerParams {
  
    public void handleStartCopy() throws RemoteException {
        if (origin.staged) {
            // 设置安装标志位，决定是安装在手机内部存储空间还是 sdcard 中
            if (origin.file != null) {
                installFlags |= PackageManager.INSTALL_INTERNAL;
                installFlags &= ~PackageManager.INSTALL_EXTERNAL;
            } 
        }
        
        // 判断安装位置
        final boolean onSd = (installFlags & PackageManager.INSTALL_EXTERNAL) != 0;
        final boolean onInt = (installFlags & PackageManager.INSTALL_INTERNAL) != 0;
        final boolean ephemeral = (installFlags & PackageManager.INSTALL_INSTANT_APP) != 0;
      
        final InstallArgs args = createInstallArgs(this); 
        // ...  
        ret = args.copyApk(mContainerService, true);
    }
  
    private InstallArgs createInstallArgs(InstallParams params) {
        if (params.move != null) {
            return new MoveInstallArgs(params);
        } else {
            return new FileInstallArgs(params);
        }
    }
}
```

正常的流程下，createInstallArgs 返回的是 FileInstallArgs 对象

#### FileInstallArgs 的 copyApk 方法

````JAVA
int copyApk(IMediaContainerService imcs, boolean temp) throws RemoteException {
    return doCopyApk(imcs, temp);
}

private int doCopyApk(IMediaContainerService imcs, boolean temp) throws RemoteException {
    // 创建存储安装包的目标路径，实际上是 /data/app/ 应用包名目录
    final File tempDir = mInstallerService.allocateStageDirLegacy(volumeUuid, isEphemeral);

    final IParcelFileDescriptorFactory target = new IParcelFileDescriptorFactory.Stub() {
        @Override
        public ParcelFileDescriptor open(String name, int mode) throws RemoteException {
            final File file = new File(codeFile, name);
            final FileDescriptor fd = Os.open(file.getAbsolutePath(),
                    O_RDWR | O_CREAT, 0644);
            Os.chmod(file.getAbsolutePath(), 0644);
            return new ParcelFileDescriptor(fd);
        }
    };
    // 调用服务的 copyPackage 方法将安装包 apk 拷贝到目标路径中；
    ret = imcs.copyPackage(origin.file.getAbsolutePath(), target);
    // 将 apk 中的动态库 .so 文件也拷贝到目标路径中。
    ret = NativeLibraryHelper.copyNativeBinariesWithOverride(handle, libraryRoot,
                        abiOverride);
}
````

这里的 IMediaContainerService imcs 就是之前连接上的 DefaultContainerService

#### DefaultContainerService

 copyPackage 方法本质上就是执行 IO 流操作，具体如下：

````JAVA
// new IMediaContainerService.Stub()
public int copyPackage(String packagePath, IParcelFileDescriptorFactory target) {
    PackageLite pkg = null;
    final File packageFile = new File(packagePath);
    pkg = PackageParser.parsePackageLite(packageFile, 0);
    return copyPackageInner(pkg, target);
}

// DefaultContainerService
private int copyPackageInner(PackageLite pkg, IParcelFileDescriptorFactory target){
    copyFile(pkg.baseCodePath, target, "base.apk");
    if (!ArrayUtils.isEmpty(pkg.splitNames)) {
        for (int i = 0; i < pkg.splitNames.length; i++) {
            copyFile(pkg.splitCodePaths[i], target, "split_" + pkg.splitNames[i] + ".apk");
        }
    }

    return PackageManager.INSTALL_SUCCEEDED;
}

private void copyFile(String sourcePath, IParcelFileDescriptorFactory target, String targetName){
    InputStream in = null;
    OutputStream out = null;
    try {
        in = new FileInputStream(sourcePath);
        out = new ParcelFileDescriptor.AutoCloseOutputStream(
                target.open(targetName, ParcelFileDescriptor.MODE_READ_WRITE));
        FileUtils.copy(in, out);
    } finally {
        IoUtils.closeQuietly(out);
        IoUtils.closeQuietly(in);
    }
}
````

最终安装包在 data/app 目录下以 base.apk 的方式保存，至此安装包拷贝工作就已经完成。

## 装载代码

安装包拷贝完成，就要开始真正安装了。代码回到上述的 HandlerParams 中的 startCopy 方法：

```java
private abstract class HandlerParams {
    final boolean startCopy() {
        ...
        handleStartCopy();
        handleReturnCode();
    }
}

class InstallParams extends HandlerParams {
    @Override
    void handleReturnCode() {
        // If mArgs is null, then MCS couldn't be reached. When it
        // reconnects, it will try again to install. At that point, this
        // will succeed.
        if (mArgs != null) {
            processPendingInstall(mArgs, mRet);
        }
    }  
  
    private void processPendingInstall(final InstallArgs args, final int currentStatus) {
         mHandler.post(new Runnable() {
             public void run() {
                 PackageInstalledInfo res = new PackageInstalledInfo();
                 if (res.returnCode == PackageManager.INSTALL_SUCCEEDED) {
                    // 预安装操作，主要是检查安装包的状态，确保安装环境正常，如果安装环境有问题会清理拷贝文件
                    args.doPreInstall(res.returnCode);
                    synchronized (mInstallLock) {
                        // 安装阶段
                        installPackageTracedLI(args, res);
                    }
                    args.doPostInstall(res.returnCode, res.uid);
                }
                ...
             }
         }
    }
}

```

#### installPackageLI

installPackageTracedLI 方法中添加跟踪 Trace，然后调用 installPackageLI 方法进行安装。这个方法有 600 行，取部分关键代码：

````java
private void installPackageLI(InstallArgs args, PackageInstalledInfo res) {
    ...
    PackageParser pp = new PackageParser();
    final PackageParser.Package pkg;
    // 1. parsePackage
    pkg = pp.parsePackage(tmpPackageFile, parseFlags);
    
    // 2. 校验安装包签名
    final KeySetManagerService ksms = mSettings.mKeySetManagerService;
    if (ksms.shouldCheckUpgradeKeySetLocked(signatureCheckPs, scanFlags)) {
        if (!ksms.checkUpgradeKeySetLocked(signatureCheckPs, pkg)) {
            res.setError(INSTALL_FAILED_UPDATE_INCOMPATIBLE, "Package "
                    + pkg.packageName + " upgrade keys do not match the "
                    + "previously installed version");
            return;
        }
    }
    
    // 3. 设置相关权限，生成、移植权限 
    int N = pkg.permissions.size();
    for (int i = N-1; i >= 0; i--) {
        final PackageParser.Permission perm = pkg.permissions.get(i);
        ...
    }
  
    // 4. 生成安装包Abi(Application binary interface，应用二进制接口)
    try {
        String abiOverride = (TextUtils.isEmpty(pkg.cpuAbiOverride) ?
            args.abiOverride : pkg.cpuAbiOverride);
        final boolean extractNativeLibs = !pkg.isLibrary();
        derivePackageAbi(pkg, abiOverride, extractNativeLibs);
    } catch (PackageManagerException pme) {
        res.setError(INSTALL_FAILED_INTERNAL_ERROR, "Error deriving application ABI");
        return;
    }
  
    // 5. 冻结 APK，执行替换安装 或 新安装， 
    try (PackageFreezer freezer = freezePackageForInstall(pkgName, installFlags,
            "installPackageLI")) {
        if (replace) {
            replacePackageLIF(pkg, parseFlags, scanFlags, args.user,
                    installerPackageName, res, args.installReason);
        } else {
            private void installNewPackageLIF((pkg, parseFlags, scanFlags | SCAN_DELETE_DATA_ON_FAILURES,
                    args.user, installerPackageName, volumeUuid, res, args.installReason);
        }
    }
  
    // 5. 优化dex文件（实际为 dex2oat 操作，用来将 apk 中的 dex 文件转换为 oat 文件）
    if (performDexopt) { 
          mPackageDexOptimizer.performDexOpt(pkg, pkg.usesLibraryFiles,
                  null /* instructionSets */,
                  getOrCreateCompilerPackageStats(pkg),
                  mDexManager.getPackageUseInfoOrDefault(pkg.packageName),
                  dexoptOptions); 
    }
    ...
}
````

最后我们来看一下 installNewPackageLIF

```JAVA
private void installNewPackageLIF(PackageParser.Package pkg, final @ParseFlags int parseFlags,
            final @ScanFlags int scanFlags,...) {
    // 继续扫描解析 apk 安装包文件，保存 apk 相关信息到 PMS 中，并创建 apk 的 data 目录，具体路径为 /data/data/应用包名
    PackageParser.Package newPackage = scanPackageTracedLI(pkg, parseFlags, scanFlags,
        System.currentTimeMillis(), user);
    // 更新系统设置中的应用信息，比如应用的权限信息
    updateSettingsLI(newPackage, installerPackageName, null, res, user, installReason);
    if (res.returnCode == PackageManager.INSTALL_SUCCEEDED) {
         // 安装然后准备 APP 数据 
         prepareAppDataAfterInstallLIF(newPackage);
     } else {
         // 如果安装失败，则将安装包以及各种缓存文件删除
         deletePackageLIF(pkgName, UserHandle.ALL, false, null,
                 PackageManager.DELETE_KEEP_DATA, res.removedInfo, true, null);
     }
}
```

prepareAppDataAfterInstallLIF 还会有一系列的调用

```JAVA
   prepareAppDataAfterInstallLIF()
-> prepareAppDataLIF()
-> prepareAppDataLeafLIF()
-> mInstaller.createAppData(...)

final Installer mInstaller;
private void prepareAppDataLeafLIF(...) {
   // 最终调用 系统服务 Installer 安装
   ceDataInode = mInstaller.createAppData(volumeUuid, packageName, userId, flags,
                    appId, seInfo, app.targetSdkVersion);     
}   

public class Installer extends SystemService {
   ...
}

```

至此整个 apk 的安装过程结束，实际上安装成功之后，还会发送一个 App 安装成功的广播 ACTION_PACKAGE_ADDED。手机桌面应用注册了这个广播，当接收到应用安装成功之后，就将 apk 的启动 icon 显示在桌面上。

## 总结

在手机上仅仅是点一下安装按钮而已，背后却有着这么繁琐的流程，相信通过今天的学习大家应该能对系统的应用安装流程有一个完整的认知。回顾一下安装的流程如下：

- 点击 APK 安装，会启动 PackageInstallerActivity，再进入 InstallInstalling 这两个 Activity 显示应用信息
- 点击页面上的安装，将 APK 信息存入 PackageInstaller.Session 传到 PMS
- PMS会做两件事，拷贝安装包和装载代码
- 在拷贝安装包过程中会开启 Service 来 copyAPK 、检查apk安装路径，包的状态
- 拷贝完成以 base.apk 形式存在/data/app包名下
- 装载代码过程中，会继续解析 APK，把清单文件内容存放于 PMS
- 对 apk 进行签名校验
- 安装成功后，更新应用设置权限，发送广播通知桌面显示APP图标，安装失败则删除安装包和各种缓存文件
- 执行 dex2oat 优化

**本文源码基于 Android 28**


