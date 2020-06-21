ServiceManager 是 Android 系统中重要的组成部分，我们有必要理解它的工作原理，本文从三个方面介绍：

- 1.ServiceManager 概述
- 2.ServiceManager 启动原理
- 3.服务的注册与查询

## ServiceManager 概述

Binder 是 Android 中使用最广泛的 IPC 机制，正因为有了 Binder，Android 系统中形形色色的进程与组件才能真正统一成有机的整体。Binder 通信机制与 TCP/IP 有共通之处，其组成元素可以这样来类比：

- binder 驱动  ->  路由器
- ServiceManager  -> DNS
- Binder Client  ->  客户端
- Binder Server  ->  服务器

ServiceManager 是为了完成 Binder Server 的 Name（域名）和 Handle（IP 地址）之间对应关系的查询而存在的，它主要包含的功能：

**注册**：当一个 Binder Server 创建后，应该将这个 Server 的 Name 和 Handle 对应关系记录到 ServiceManager 中

**查询**：其他应用可以根据 Server 的 Name 查询到对应的 Service Handle

但 ServiceManager 自身也是一个 Binder Server（服务器），怎么找到它的 "IP 地址"呢？Binder 机制对此做了特别规定：ServiceManager 在 Binder 通信过程中的 Handle 永远是 0。

## ServiceManager 启动原理

Android 系统第一个启动的 init 进程解析 init.rc 脚本时构建出系统的初始运行状态，Android 系统服务大多是在这个脚本中描述并被相继启动的，包括 zygote、mediaserver、surfaceflinger 以及 servicemanager 等，其中 servicemanager 描述如下：

```objectivec
#init.rc
service servicemanager /system/bin/servicemanager
    class core
    user system
    group system
    critical
    onrestart restart healthd
    onrestart restart zygote
    onrestart restart media
    onrestart restart surfaceflinger
    onrestart restart drm
```

可以看到，当 ServiceManager 发生问题重启时，其他 healthd、zygote、media 等服务也会被重启。ServiceManager 服务启动后会执行 service_manager.c 的 main 函数，关键代码如下：

```objectivec
//frameworks/native/cmds/servicemanager/service_manager.c
int main(){
    bs = binder_open(128*1024);
    if (binder_become_context_manager(bs)) {
        ...
    }
    ...
    binder_loop(bs, svcmgr_handler);
    return 0;
}
```

其中三个函数对应了 ServiceManager 初始化的三个关键工作：

1. binder_open()：打开 binder 驱动并映射内存块大小为 128KB
2. binder_become_context_manager()：将自己设置为 Binder "DNS" 管理者
3. binder_loop()：进入循环，等待 binder 驱动发来消息

下面分别来分析这三个函数，首先来看 binder_open() 是怎么打开 binder 驱动并映射内存的：
```objectivec
struct binder_state *binder_open(size_t mapsize){
    struct binder_state *bs;
    struct binder_version vers;
    bs = malloc(sizeof(*bs));
    ...
    //打开 binder 驱动，最终调用 binder_open() 函数
    bs->fd = open("/dev/binder", O_RDWR | O_CLOEXEC);
    ...
    //获取 Binder 版本，最终调用 binder_ioctl() 函数
    ioctl(bs->fd, BINDER_VERSION, &vers)
    ...
    //将虚拟内存映射到 Binder，最终调用 binder_mmap() 函数
    bs->mapped = mmap(NULL, mapsize, PROT_READ, MAP_PRIVATE, bs->fd, 0);
    ...
    return bs;
}
```

再来看 binder_become_context_manager() 是怎么将自己设置为 Binder "DNS 管理者的"：

```objectivec
int binder_become_context_manager(struct binder_state *bs){
    //发送 BINDER_SET_CONTEXT_MGR 命令，最终调用 binder_ioctl() 函数
    return ioctl(bs->fd, BINDER_SET_CONTEXT_MGR, 0);
}
```

最后来看 binder_loop() 是怎么循环等待并处理 binder 驱动发来的消息：

```objectivec
void binder_loop(struct binder_state *bs, binder_handler func){
    int res;
    //执行 BINDER_WRITE_READ 命令所需的数据格式：
    struct binder_write_read bwr;
    uint32_t readbuf[32]; //每次读取数据的大小
    readbuf[0] = BC_ENTER_LOOPER; 
    //先将 binder 驱动的进入循环命令发送给 binder 驱动：
    binder_write(bs, readbuf, sizeof(uint32_t));
    for (;;) { //进入循环
        bwr.read_size = sizeof(readbuf);
        //读取到的消息数据存储在 readbuf
        bwr.read_buffer = (uintptr_t) readbuf; 
        //执行 BINDER_WRITE_READ 命令读取消息数据
        res = ioctl(bs->fd, BINDER_WRITE_READ, &bwr);
        if (res < 0) {
            ALOGE("binder_loop: ioctl failed (%s)\n", strerror(errno));
            break;
        }
        //处理读取到的消息数据
        res = binder_parse(bs, 0, (uintptr_t) readbuf, bwr.read_consumed, func);
        ...
    }
}
```
BINDER_WRITE_READ 命令既可以用来读取数据也可以写入数据，具体是写入还是读取依赖 binder_write_read 结构体的 write_size 和 read_size 哪个大于 0，上面代码通过 bwr.read_size = sizeof(readbuf) 赋值，所以是读取消息。

binder_parse() 方法内部处理由 binder 驱动主动发出的、一系列 BR_ 开头的命令，包括上面提到过的 BR_TRANSACTION、BR_REPLY 等，简化后的代码如下：

```objectivec
int binder_parse(struct binder_state *bs, struct binder_io *bio,
                 uintptr_t ptr, size_t size, binder_handler func){
    switch(cmd) {
        case BR_TRANSACTION: {
            ...
            res = func(bs, txn, &msg, &reply); //处理消息
            //返回处理结果
            inder_send_reply(bs, &reply, txn->data.ptr.buffer, res); 
            ...
            break;
        }
        case BR_REPLY: {...}
        case BR_DEAD_BINDER: {...}
        ...
    }
}
```
对于 BR_TRANSACTION 命令主要做了两个工作，一是调用 func() 具体处理消息；二是调用 inder_send_reply() 将消息处理结果告知给 binder 驱动，注意这里的 func 是由 service_manager.c main 函数中传过来的方法指针，也就是 svcmgr_handler() 方法。

## 服务注册与查询

经过上面 ServiceManager 服务启动的过程分析，已经知道由 binder 驱动主动发过来的 BR_TRANSACTION 命令最终在 service_manager.c 的 svcmgr_handler() 方法中处理，那服务的注册与查询请求想必就是在这个方法中实现的了，确实如此，简化后的关键代码如下：

```objectivec
int svcmgr_handler(struct binder_state *bs,
                   struct binder_transaction_data *txn,
                   struct binder_io *msg,
                   struct binder_io *reply){
    switch(txn->code) {
         case SVC_MGR_GET_SERVICE:
         case SVC_MGR_CHECK_SERVICE:
              //查询服务，根据 name 查询 Server Handle
              handle = do_find_service(s, len, txn->sender_euid, txn->sender_pid);
              return 0;
         case SVC_MGR_ADD_SERVICE:
             //注册服务，记录服务的 name(下面的参数 s) 与 handle
             if (do_add_service(bs, s, len, handle, txn->sender_euid,
                 allow_isolated, txn->sender_pid))
                 return -1;
             break;
         case SVC_MGR_LIST_SERVICES: {
             //查询所有服务，返回存储所有服务的链表 svclist
             si = svclist;
             while ((n-- > 0) && si)
                 si = si->next;
             if (si) {
                 bio_put_string16(reply, si->name);
                 return 0;
             }
             return -1;
    }
    bio_put_uint32(reply, 0);
    return 0;
}
```
注册的服务都会存储在 svclist 链表上，do_add_service() 就是将服务插入到 svclist 链表上记录下来，do_find_service() 方法则遍历 svclist 查找对应的服务。

svcmgr_handler() 方法执行完后会进一步调用 inder_send_reply() 将执行结果回复给 binder 驱动，然后进入下一轮循环继续等待处理消息。

## 总结

ServiceManager 在 init.rc 中描述，由 init 进程启动，运行在一个单独的进程。

ServiceManager 启动后主要做了三件事：1. 打开 binder 驱动并映射内存；2. 将自己设置为 "服务大管家"；3. 循环等待 binder 驱动发来的消息。

ServiceManager 通过记录服务 Name 和 Handler 的关系，提供服务注册与查询功能。

对 Client 来说，ServiceManager 也是一个 Service，Client 可以直接获取到这个 Service，因为它的句柄值固定为 0。

![](/img/binder_code.png)

>如上图，这里有一份按模块分好的 Binder 源码，并有关键步骤注释。关注公众号 **Android 面试官** 留言：binder，即可获得此 Binder 学习必备源码～