>本文设计源码及逻辑较多，完全理解可能要耗一定精力，已尽量画图辅助，建议收藏。

应用程序中执行 getService() 需与 ServiceManager 通过 binder 跨进程通信，此过程中会贯穿 Framework、Natve 层以及 Linux 内核驱动。

![](https://imgkr2.cn-bj.ufileos.com/96d7bdb3-99de-4363-a117-fa971023d60f.png?UCloudPublicKey=TOKEN_8d8b72be-579a-4e83-bfd0-5f6ce1546f13&Signature=zhlNjpY74%252FcWuex0nGqjF9TMVDI%253D&Expires=1599481477)

binder 驱动的整体分层如上图，下面先来宏观的了解下 getService() 在整个 Android 系统中的调用栈，ServiceManager 本身的获取：
![](https://imgkr2.cn-bj.ufileos.com/dde0d1f1-cd61-4d3e-ada1-ad895a78e847.png?UCloudPublicKey=TOKEN_8d8b72be-579a-4e83-bfd0-5f6ce1546f13&Signature=8gL8sgibHfGMFKnO9VWknJpKTDQ%253D&Expires=1599481681)

与 ServiceManager 进行 IPC 通信：
![](https://imgkr2.cn-bj.ufileos.com/8f7afabf-4e51-45ca-b760-e6a18dc74227.png?UCloudPublicKey=TOKEN_8d8b72be-579a-4e83-bfd0-5f6ce1546f13&Signature=3HWfOifWnLuXs5UwNk79xCUZl9o%253D&Expires=1599481738)

**本文将主要分析此过程中 binder 驱动具体承担了哪些工作**，也就是上图中 IPCThreadState 与 binder 驱动的 ioctl 调用。

binder 驱动中做的工作可以总结为以下几步：

1. 准备数据，根据命令分发给具体的方法去处理
2. 找到目标进程的相关信息
3. 将数据一次拷贝到目标进程所映射的物理内存块
4. 记录待处理的任务，唤醒目标线程
5. 调用线程进入休眠
6. 目标进程直接拿到数据进行处理，处理完后唤醒调用线程
7. 调用线程返回处理结果

在源码中实际会执行到的函数主要包括：

1. binder_ioctl()
2. binder_get_thread()
3. binder_ioctl_write_read()
4. binder_thread_write()
5. binder_transaction()
6. binder_thread_read()

下面按照这些 binder 驱动中的函数，以工作步骤为脉络，深入分析驱动中的源码执行逻辑，彻底搞定 binder 驱动！

## 1.binder_ioctl()

在 IPCThreadState 中通过系统调用 ioctl 陷入系统内核，调用到 binder_ioctl() 方法：
```objectivec
ioctl(mProcess->mDriverFD, BINDER_WRITE_READ, &bwr)
```

binder_ioctl() 方法中会根据 BINDER_WRITE_READ、BINDER_SET_MAX_THREADS 等不同 cmd 转调到不同的方法去执行，这里我们只关注 BINDER_WRITE_READ，代码如下：

```objectivec
static long binder_ioctl(struct file *filp, unsigned int cmd, unsigned long arg){
    int ret;
    //拿到调用进程在 binder_open() 中记录的 binder_proc
    struct binder_proc *proc = filp->private_data;
    struct binder_thread *thread;
    binder_lock(__func__);
    //获取调用线程 binder_thread
    thread = binder_get_thread(proc);
    switch (cmd) {
    case BINDER_WRITE_READ:
      //处理 binder 数据读写,binder IPC 通信的核心逻辑
    	ret = binder_ioctl_write_read(filp, cmd, arg, thread);
    	if (ret)
    		goto err;
    	break;
    ...
}
```

之前文章介绍过 binder_open() 方法， binder_open() 方法主要做了两个工作：
1. 创建及初始化每个进程独有一份的、用来存放 binder 相关数据的 binder_proc 结构体
2. **将 binder_proc 记录起来，方便后续使用**。

正是通过 file 的 private_data 来记录的：

```objectivec
static int binder_open(struct inode *nodp, struct file *filp){
    ...
    filp->private_data = proc;
    ...
}
```

拿到调用进程后，进一步通过 binder_get_thread() 方法拿到调用线程，然后就交给 binder_ioctl_write_read() 方法去执行具体的 binder 数据读写了。

可见 binder_ioctl() 方法本身的逻辑非常简单，将数据 arg 透传了出去。

下面分别来看 binder_get_thread()、binder_ioctl_write_read() 这两个方法。

## 2.binder_get_thread()

```objectivec
static struct binder_thread *binder_get_thread(
                            struct binder_proc *proc){
    struct binder_thread *thread = NULL;
    struct rb_node *parent = NULL;
    //从 proc 中获取红黑树根节点
    struct rb_node **p = &proc->threads.rb_node;
    //查找 pid 等于当前线程 id 的thread，该红黑树以 pid 大小为序存放
    while (*p) {
        parent = *p;
        thread = rb_entry(parent, struct binder_thread, rb_node);
        //current->pid 是当前调用线程的 id
        if (current->pid < thread->pid)
            p = &(*p)->rb_left;
        else if (current->pid > thread->pid)
            p = &(*p)->rb_right;
        else
            break;
    }

    if (*p == NULL) {//如果没有找到，则新创建一个
        thread = kzalloc(sizeof(*thread), GFP_KERNEL);
        if (thread == NULL)
            return NULL;
        binder_stats_created(BINDER_STAT_THREAD);
        thread->proc = proc;
        thread->pid = current->pid;
        init_waitqueue_head(&thread->wait); //初始化等待队列
        INIT_LIST_HEAD(&thread->todo); //初始化待处理队列
        //加入到 proc 的 threads 红黑树中
        rb_link_node(&thread->rb_node, parent, p);
        rb_insert_color(&thread->rb_node, &proc->threads);
        thread->looper |= BINDER_LOOPER_STATE_NEED_RETURN;
        thread->return_error = BR_OK;
        thread->return_error2 = BR_OK;
    }
    return thread;
}
```
binder_thread 是用来描述线程的结构体，binder_get_thread() 方法中逻辑也很简单，首先从调用进程 proc 中查找当前线程是否已被记录，如果找到就直接返回，否则新建一个返回，并记录到 proc 中。

也就是说所有调用 binder_ioctl() 的线程，都会被记录起来。

## 3.binder_ioctl_write_read

此方法分为两部分来看，首先是整体逻辑：

```objectivec
static int binder_ioctl_write_read(struct file *filp,
				unsigned int cmd, unsigned long arg,
				struct binder_thread *thread){
    int ret = 0;
    struct binder_proc *proc = filp->private_data;
    unsigned int size = _IOC_SIZE(cmd);
    //用户传下来的数据赋值给 ubuf
    void __user *ubuf = (void __user *)arg;
    struct binder_write_read bwr;
    //把用户空间数据 ubuf 拷贝到 bwr
    if (copy_from_user(&bwr, ubuf, sizeof(bwr))) {
    	ret = -EFAULT;
    	goto out;
    }
    暂时忽略处理数据逻辑...
    //将读写后的数据写回给用户空间
    if (copy_to_user(ubuf, &bwr, sizeof(bwr))) {
    	ret = -EFAULT;
    	goto out;
    }
out:
	return ret;
}

```
起初看到 copy_from_user() 方法时难以理解，因为它看起来是将我们要传输的数据拷贝到内核空间了，但目前还没有看到 server 端的任何线索，bwr 跟 server 端没有映射关系，那后续再将 bwr 传输给 server 端的时候又要拷贝，这样岂不是多次拷贝了？

其实这里的 copy_from_user() 方法并没有拷贝要传输的数据，而仅是拷贝了持有传输数据内存地址的 bwr。后续处理数据时会根据 bwr 信息真正的去拷贝要传输的数据。

处理完数据后，会将处理结果体现在 bwr 中，然后返回给用户空间处理。那是如何处理数据的呢？所谓的处理数据，就是对数据的读写而已：

```objectivec
    if (bwr.write_size > 0) {//写数据
    	ret = binder_thread_write(proc,
             thread,
             bwr.write_buffer, bwr.write_size,
             &bwr.write_consumed);
        trace_binder_write_done(ret);
        if (ret < 0) { //写失败
            bwr.read_consumed = 0;
            if (copy_to_user(ubuf, &bwr, sizeof(bwr)))
                ret = -EFAULT;
            goto out;
        }
    }
    if (bwr.read_size > 0) {//读数据
    	ret = binder_thread_read(proc, thread, bwr.read_buffer,
            bwr.read_size,
            &bwr.read_consumed,
            filp->f_flags & O_NONBLOCK);
        trace_binder_read_done(ret);
        if (!list_empty(&proc->todo))
            //唤醒等待状态的线程
            wake_up_interruptible(&proc->wait);
        if (ret < 0) { //读失败
            if (copy_to_user(ubuf, &bwr, sizeof(bwr)))
                ret = -EFAULT;
            goto out;
        }
    }
```

可见 binder 驱动内部依赖用户空间的 binder_write_read 决定是要读取还是写入数据：其内部变量 read_size>0 则代表要读取数据，write_size>0 代表要写入数据，若都大于 0 则先写入，后读取。

至此焦点应该集中在 binder_thread_write() 和 binder_thread_read()，下面分析这两个方法。

## 4.binder_thread_write

在上面的 binder_ioctl_write_read() 方法中调用 binder_thread_write() 时传入了 bwr.write_buffer、bwr.write_size 等，先搞清楚这些参数是什么。

最开始是在用户空间 IPCThreadState 的 transact() 中通过 writeTransactionData() 方法创建数据并写入 mOut 的，writeTransactionData 方法代码如下：

```objectivec
status_t IPCThreadState::writeTransactionData(int32_t cmd, uint32_t binderFlags,
    int32_t handle, uint32_t code, const Parcel& data, status_t* statusBuffer){
    binder_transaction_data tr; //到驱动内部后会取出此结构体进行处理
    tr.target.ptr = 0;
    tr.target.handle = handle; //目标 server 的 binder 的句柄
    //请求码，getService() 服务对应的是 GET_SERVICE_TRANSACTION
    tr.code = code;
    tr.flags = binderFlags;
    tr.cookie = 0;
    tr.sender_pid = 0;
    tr.sender_euid = 0;
    const status_t err = data.errorCheck(); //验证数据合理性
    if (err == NO_ERROR) {
        tr.data_size = data.ipcDataSize(); //传输数据大小
        tr.data.ptr.buffer = data.ipcData(); //传输数据
        tr.offsets_size = data.ipcObjectsCount()*sizeof(binder_size_t);
        tr.data.ptr.offsets = data.ipcObjects();
    } else {...}
    mOut.writeInt32(cmd); // transact 传入的 cmd 是 BC_TRANSACTION
    mOut.write(&tr, sizeof(tr)); //打包成 binder_transaction_data
    return NO_ERROR;
}
```
然后在 IPCThreadState 的 talkWithDriver() 方法中对 write_buffer 赋值：
```objectivec
    bwr.write_buffer = (uintptr_t)mOut.data();
```

搞清楚了数据的来源，再来看 binder_thread_write() 方法，binder_thread_write() 方法中处理了大量的 BC_XXX 命令，代码很长，这里我们只关注当前正在处理的 BC_TRANSACTION 命令，简化后代码如下：

```objectivec
static int binder_thread_write(struct binder_proc *proc,
        struct binder_thread *thread,
        binder_uintptr_t binder_buffer, size_t size,
        binder_size_t *consumed){
    uint32_t cmd;
    void __user *buffer = (void __user *)(uintptr_t)binder_buffer;
    void __user *ptr = buffer + *consumed; //数据起始地址
    void __user *end = buffer + size; //数据结束地址
    //可能有多个命令及对应数据要处理，所以要循环
    while (ptr < end && thread->return_error == BR_OK) {
        if (get_user(cmd, (uint32_t __user *)ptr)) // 读取一个 cmd
            return -EFAULT;
        //跳过 cmd 所占的空间，指向要处理的数据
        ptr += sizeof(uint32_t);
        switch (cmd) {
            case BC_TRANSACTION:
            case BC_REPLY: {
                 //与 writeTransactionData 中准备的数据结构体对应
                 struct binder_transaction_data tr;
                 //拷贝到内核空间 tr 中
                 if (copy_from_user(&tr, ptr, sizeof(tr)))
                    return -EFAULT;
                 //跳过数据所占空间，指向下一个 cmd
                 ptr += sizeof(tr);
                 //处理数据
                 binder_transaction(proc, thread, &tr, cmd == BC_REPLY);
                 break;
            }
            处理其他 BC_XX 命令...
        }
    //被写入处理消耗的数据量，对应于用户空间的 bwr.write_consumed
    *consumed = ptr - buffer;
```
binder_thread_write() 中从 bwr.write_buffer 中取出了 cmd 和 cmd 对应的数据，进一步交给 binder_transaction() 处理，需要注意的是，BC_TRANSACTION、BC_REPLY 这两个命令都是由 binder_transaction() 处理的。

简单梳理一下，由 binder_ioctl -> binder_ioctl_write_read -> binder_thread_write ，到目前为止还只是在准备数据，没有看到跟目标进程相关的任何处理，都属于 "准备数据，根据命令分发给具体的方法去处理" 第 1 个工作。

而到此为止，第 1 个工作便结束，下一步的 binder_transaction() 方法终于要开始后面的工作了。

## 5.binder_transaction

binder_transaction() 方法中代码较长，先总结它干了哪些事：对应开头列出的工作，此方法中做了非常关键的 2-4 步：

2. 找到目标进程的相关信息
3. 将数据一次拷贝到目标进程所映射的物理内存块
4. 记录待处理的任务，唤醒目标线程

以这些工作为线索，将代码分为对应的部分来看，首先是**找到目标进程的相关信息**，简化后代码如下：

```objectivec
static void binder_transaction(struct binder_proc *proc,
			       struct binder_thread *thread,
			       struct binder_transaction_data *tr, int reply){
    struct binder_transaction *t; //用于描述本次 server 端要进行的 transaction
    struct binder_work *tcomplete; //用于描述当前调用线程未完成的 transaction
    binder_size_t *offp, *off_end;
    struct binder_proc *target_proc; //目标进程
    struct binder_thread *target_thread = NULL; //目标线程
    struct binder_node *target_node = NULL; //目标 binder 节点
    struct list_head *target_list; //目标 TODO 队列
    wait_queue_head_t *target_wait; //目标等待队列
    if(reply){
        in_reply_to = thread->transaction_stack;
        ...处理 BC_REPLY，暂不关注
    }else{
        //处理 BC_TRANSACTION
        if (tr->target.handle) { //handle 不为 0
            struct binder_ref *ref;
            //根据 handle 找到目标 binder 实体节点的引用
            ref = binder_get_ref(proc, tr->target.handle);
            target_node = ref->node; //拿到目标 binder 节点
        } else {
            // handle 为 0 则代表目标 binder 是 service manager
            // 对于本次调用来说目标就是 service manager
            target_node = binder_context_mgr_node;
        }
    }
    target_proc = target_node->proc; //拿到目标进程
    if (!(tr->flags & TF_ONE_WAY) && thread->transaction_stack) {
    	struct binder_transaction *tmp;
    	tmp = thread->transaction_stack;
    	while (tmp) {
            if (tmp->from && tmp->from->proc == target_proc)
                target_thread = tmp->from; //拿到目标线程
            tmp = tmp->from_parent;
    	}
    }
    target_list = &target_thread->todo; //拿到目标 TODO 队列
    target_wait = &target_thread->wait; //拿到目标等待队列
```
binder_transaction、binder_work 等结构体在上一篇中有介绍，上面代码中也详细注释了它们的含义。比较关键的是 binder_get_ref() 方法，它是如何找到目标 binder 的呢？这里暂不延伸，下文再做分析。

继续看 binder_transaction() 方法的第 2 个工作，**将数据一次拷贝到目标进程所映射的物理内存块**：

```objectivec
    t = kzalloc(sizeof(*t), GFP_KERNEL); //创建用于描述本次 server 端要进行的 transaction
    tcomplete = kzalloc(sizeof(*tcomplete), GFP_KERNEL); //创建用于描述当前调用线程未完成的 transaction
    if (!reply && !(tr->flags & TF_ONE_WAY)) //将信息记录到 t 中：
        t->from = thread; //记录调用线程
    else
        t->from = NULL;
    t->sender_euid = task_euid(proc->tsk);
    t->to_proc = target_proc; //记录目标进程
    t->to_thread = target_thread; //记录目标线程
    t->code = tr->code; //记录请求码，getService() 对应的是 GET_SERVICE_TRANSACTION
    t->flags = tr->flags;
    //实际申请目标进程所映射的物理内存，准备接收要传输的数据
    t->buffer = binder_alloc_buf(target_proc, tr->data_size,
        tr->offsets_size, !reply && (t->flags & TF_ONE_WAY));
    //申请到 t->buffer 后，从用户空间将数据拷贝进来，这里就是一次拷贝数据的地方！！
    if (copy_from_user(t->buffer->data, (const void __user *)(uintptr_t)
        tr->data.ptr.buffer, tr->data_size)) {
        return_error = BR_FAILED_REPLY;
        goto err_copy_data_failed;
    }
```

为什么在拷贝之前要先申请物理内存呢？之前介绍 binder_mmap() 方法时详细分析过，虽然 binder_mmap() 直接映射了 (1M-8K) 的虚拟内存，但却只申请了 1 页的物理页面，等到实际使用时再动态申请。也就是说，在 binder_ioctl() 实际传输数据的时候，再通过 binder_alloc_buf() 方法去申请物理内存。

至此已经将要传输的数据拷贝到目标进程，目标进程可以直接读取到了，接下来要做的就是将目标进程要处理的任务记录起来，然后唤醒目标进程，这样在目标进程被唤醒后，才能知道要处理什么任务。

最后来看 binder_transaction() 方法的第 3 个工作，**记录待处理的任务，唤醒目标线程**：

```objectivec
	if (reply) { //如果是处理 BC_REPLY，pop 出来栈顶记录的 transaction(实际上是删除链表头元素)
		binder_pop_transaction(target_thread, in_reply_to);
	} else if (!(t->flags & TF_ONE_WAY)) {
        //如果不是 oneway，将 server 端要处理的 transaction 记录到当前调用线程
		t->need_reply = 1;
		t->from_parent = thread->transaction_stack;
		thread->transaction_stack = t;
	} else {
		...暂不关注 oneway 的情况
	}
	t->work.type = BINDER_WORK_TRANSACTION;
    list_add_tail(&t->work.entry, target_list); //加入目标的处理队列中
    tcomplete->type = BINDER_WORK_TRANSACTION_COMPLETE; //设置调用线程待处理的任务类型
    list_add_tail(&tcomplete->entry, &thread->todo); //记录调用线程待处理的任务
    if (target_wait)
        wake_up_interruptible(target_wait); //唤醒目标线程
```

再次梳理一下，至此已经完成了前四个工作：

1. 准备数据，根据命令分发给具体的方法去处理
2. 找到目标进程的相关信息
3. 将数据一次拷贝到目标进程所映射的物理内存块
4. 记录待处理的任务，唤醒目标线程

其中第 1 个工作涉及到的方法为 binder_ioctl() -> binder_get_thread() -> binder_ioctl_write_read()  -> binder_thread_write() ，主要是一些数据的准备和方法转跳，没做什么实质的事情。而 binder_transaction()  方法中做了非常重要的 2-4 工作。

剩下的工作还有：

5. 调用线程进入休眠
6. 目标进程直接拿到数据进行处理，处理完后唤醒调用线程
7. 调用线程返回处理结果

可以想到，5 和 6 其实没有时序上的限制，而是并行处理的。下面先来看第 5 个工作：调用线程是如何进入休眠等待服务端执行结果的。

## 6.binder_thread_read

在唤醒目标线程后，调用线程就执行完 binder_thread_write() 写完了数据，返回到 binder_ioctl_write_read() 方法中，接着执行 binder_thread_read() 方法。

而调用线程的休眠就是在此方法中触发的，下面将 binder_thread_read() 分为两部分来看，首先是是否阻塞当前线程的判断逻辑：

```objectivec
static int binder_thread_read(struct binder_proc *proc,
            struct binder_thread *thread,
            binder_uintptr_t binder_buffer, size_t size,
            binder_size_t *consumed, int non_block){
    void __user *buffer = (void __user *)(uintptr_t)binder_buffer; //bwr.read_buffer
    void __user *ptr = buffer + *consumed; //数据起始地址
    void __user *end = buffer + size; //数据结束地址
    if (*consumed == 0) {
        if (put_user(BR_NOOP, (uint32_t __user *)ptr))
            return -EFAULT;
        ptr += sizeof(uint32_t);
    }
    //是否要准备睡眠当前线程
    wait_for_proc_work = thread->transaction_stack == NULL &&
            list_empty(&thread->todo);
    if (wait_for_proc_work) {
        if (non_block) { //non_block 为 false
            if (!binder_has_proc_work(proc, thread))
                ret = -EAGAIN;
        } else
    	    ret = wait_event_freezable_exclusive(proc->wait,
                        binder_has_proc_work(proc, thread));
    } else {
        if (non_block) { //non_block 为 false
            if (!binder_has_thread_work(thread))
                ret = -EAGAIN;
        } else
            ret = wait_event_freezable(thread->wait,
                        binder_has_thread_work(thread));
    }
```
consumed 即用户空间的 bwr.read_consumed，这里是 0 ，所以将一个 BR_NOOP 加到了 ptr 中。

怎么理解 wait_for_proc_work 条件呢？在 binder_transaction() 方法中将 server 端要处理的 transaction 记录到了当前调用线程 thread->transaction_stack 中；将当前调用线程待处理的任务记录到了 thread->todo 中。

所以这里的 thread->transaction_stack 和 thread->todo 都不为空，wait_for_proc_work 为 false，代表不准备阻塞当前线程。

但 wait_for_proc_work 并不是决定是否睡眠的最终条件，接着往下看，其中 non_block 恒为 false，那是否要睡眠当前线程就取决于 binder_has_thread_work() 的返回值，binder_has_thread_work() 方法如下：

```objectivec
static int binder_has_thread_work(struct binder_thread *thread){
    return !list_empty(&thread->todo) || thread->return_error != BR_OK ||
        (thread->looper & BINDER_LOOPER_STATE_NEED_RETURN);
}
```
thread->todo 不为空，所以 binder_has_thread_work() 返回 true，当前调用线程不进入休眠，继续往下执行。你可能会有疑问，不是说调用线程的休眠就是在 binder_thread_read() 方法中触发的吗？确实是，只不过不是本次，先接着分析 binder_thread_read() 继续往下要执行的逻辑：

```objectivec
struct binder_work *w;
w = list_first_entry(&thread->todo, struct binder_work,entry);
switch (w->type) {
    case BINDER_WORK_TRANSACTION_COMPLETE: {
        cmd = BR_TRANSACTION_COMPLETE;
        if (put_user(cmd, (uint32_t __user *)ptr))
            return -EFAULT;
        ptr += sizeof(uint32_t);
        binder_stat_br(proc, thread, cmd);
        list_del(&w->entry); //删除 binder_work 在 thread->todo 中的引用
        kfree(w);
    }
    case BINDER_WORK_NODE{...}
    case BINDER_WORK_DEAD_BINDER{...}
    ...
```
在上面 binder_transaction() 方法最后，将 BINDER_WORK_TRANSACTION_COMPLETE 类型的 binder_work 加入到 thread->todo 中。而这里就是对这个 binder_work 进行处理，将一个 BR_TRANSACTION_COMPLETE 命令加到了 ptr 中。

梳理一下目前的逻辑，至此已经顺序执行完 binder_thread_write()、binder_thread_read() 方法，并且在 binder_thread_read() 中往用户空间传输了两个命令：BR_NOOP 和 BR_TRANSACTION_COMPLETE。

本次 binder_ioctl() 调用就执行完了，然后会回到 IPCThreadState 中，在 [掌握 binder 机制？先搞懂这几个关键类！](https://mp.weixin.qq.com/s/gHtZ9pjMJ-jXA12rvXA4cg) 详细分析过 IPCThreadState 中的代码，这里就不再展开，简单概括一下后续执行的逻辑：

mIn 中有 BR_NOOP 和 BR_TRANSACTION_COMPLETE 两个命令，首先处理 BR_NOOP 命令，此命令什么也没做，由于 talkWithDriver() 处于 while 循环中，会再一次进入 talkWithDriver()，但因为此时 mIn 中还有数据没读完，不会调用 binder_ioctl()。

然后处理 BR_TRANSACTION_COMPLETE 命令，如果是 oneway 就直接结束本次 IPC 调用，否则再一次进入 talkWithDriver()，第二次进入 talkWithDriver 时，bwr.write_size = 0，bwr.read_size > 0，所以会第二次调用 binder_ioctl() 方法。第二次执行 binder_ioctl() 时，bwr.write_size = 0，bwr.read_size > 0，所以不会再执行 binder_thread_write() 方法，而只执行 binder_thread_read() 方法。

第二次执行 binder_thread_read() 时，thread->todo 已经被处理为空，但是 thread->transaction_stack 还不为空，wait_for_proc_work 仍然为 false，但最终决定是否要休眠的条件成立了： binder_has_thread_work(thread) 返回 false，由此当前调用线程通过 wait_event_freezable() 进入休眠。

## 最后

至此还剩下两个工作：

6. 目标进程直接拿到数据进行处理，处理完后唤醒调用线程
7. 调用线程返回处理结果

但是已经不用再看代码了，因为上述方法已经覆盖了剩下的工作。对于 getService() 来说，目标进程就是 Service Manager，其与 binder 驱动交互相关代码在 [ServiceManager 的工作原理](https://mp.weixin.qq.com/s/1d_LHbzfp4l8qQEu7tfKZg) 也已经做过详细的分析。

最后仍然补上图来概括 binder 驱动所承担的工作。调用进程逻辑：

![](https://upload-images.jianshu.io/upload_images/4679478-86f75e5d9482f12a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


Service Manager 端逻辑：

![](https://upload-images.jianshu.io/upload_images/4679478-c9344e7ee100ba83.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


本节完整的分析了一次 IPC 调用中 binder 驱动内部具体的执行逻辑，此部分也是 binder 机制中最难的，而将最难的部分掌握后，可以极大的提高信心。



