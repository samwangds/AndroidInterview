GUI（Graphical User Interface）即图形用户界面，官方架构图如下：

![](/img/ape_fwk_graphics.png)

下面分别介绍上图中的几个重要角色：

#### IMAGE STREAM PRODUCERS 图像流生产方

生成图形缓冲区以供消耗的任何内容，例如 OpenGL ES、Canvas 2D 和 mediaserver 视频解码器都是图像流生产方。

#### IMAGE STREAM CONSUMERS 图像流消耗方

图像流最常见的消耗方是 SurfaceFlinger，该系统服务接收来自于多个源的数据缓冲区，组合它们，并将它们发送给显示设备。

除了 SurfaceFlinger，OpenGL ES 应用也可以消耗图像流，例如相机应用会消耗相机预览图像流，另外非 GL 应用也可以消耗图像流，例如 ImageReader 类。

SurfaceFlinger 使用 OpenGL 和 Hardware Composer 来合成一组 Surface。

#### WindowManager

WindowManager 会控制 window 对象，window 是用于容纳视图对象的容器。

window 对象由 Surface 对象提供支持。WindowManager 会监督生命周期、输入和聚焦事件、屏幕方向、转换、动画、位置、变形、Z 轴顺序等窗口事件。

WindowManager 会将所有窗口元数据发送到 SurfaceFlinger，以便 SurfaceFlinger 可以使用这些数据合成 Surface。

#### Surface

无论开发者使用什么渲染 API，一切内容都会渲染到 Surface 上，Surface 即供 UI 应用程序绘制图像的 "画板"，承载应用要渲染的图像数据。

应用端可以使用 OpenGL ES 、Vulkan 或 Canvas API 渲染到 Surface 上。

#### Hardware Composer 硬件混合渲染器(HWC)

用于确定组合缓冲区的最有效方式，作为 HAL 硬件抽象层，其实现是基于特定设备的，通常由屏幕硬件设备制造商 (OEM) 完成。

SurfaceFlinger 在收集可见层的所有缓冲区后，便会询问 HWC 应如何进行合成。如果 HWC 将层合成类型标记为客户端合成，则 SurfaceFlinger 会合成这些层，然后 SurfaceFlinger 会将输出缓冲区传递给 HWC。

#### Gralloc

包括 fb 和 gralloc 两个设备，fb 负责打开内核中的 FrameBuffer、初始化配置，并提供了 post、setSwapInterval 等操作接口；gralloc 负责管理帧缓冲区的分配和释放。

作为 HAL，上层都会通过 Gralloc 来访问内核显示设备的帧缓冲区。

### BufferQueue 与图像数据流

图像流由生产方流向消耗方，这种典型的生产者-消费者模型都是需要一个缓冲区队列的，BufferQueue 就是这个队列，它将图像流生产方与消耗方结合在一起，并且可以调解图像流从生产方到消耗方的固定周期。

![](/img/bufferqueue.png)

如图，生产方通过 dequeue 向 BufferQueue 申请空闲的缓冲区，将图像数据存放进去，然后通过 queue 移交给 BufferQueue。

消耗方通过 acquire 从 BufferQueue 中获取图像数据缓冲区，然后进行合成显示或处理后，再将缓冲区交还给 BufferQueue。

对应到显示场景，应用程序作为生产方将图像数据交给 BufferQueue；SurfaceFlinger 则作为消耗方从 BufferQueue 中取出来，然后合成图像数据。