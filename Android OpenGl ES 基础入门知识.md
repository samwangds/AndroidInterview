相信大家都听过大名鼎鼎的 OpenGL，但可能大多数人没有实践使用过，本文就来介绍一下 Android OpenGl ES 的基础入门知识。

或许你在工作中不会用到，*但为你个人成长着想一下*，扩展自己的知识广度，总归是有利无弊的，你说对吧？

本文分为以下四部分介绍：
- OpenGL 基础概念
- OpenGL 坐标系理解
- OpenGL 渲染管线
- OpenGL 着色语言

*建议收藏本文哦～*

## OpenGL 基础概念

#### OpenGL
OpenGL 即 Open Graphics Library，是一个功能强大、调用方便的底层图形库，它定义了跨编程语言、跨平台的专业图形程序接口，可用于二维或三维图像的处理与渲染。

OpenGL 是跨平台的，除了它纯粹专注的渲染外，其他内容在每个平台上都要有它的具体实现，比如上下文环境和窗口的管理就交由各个设备自己来完成。
#### OpenGL ES
OpenGL ES （OpenGL for Embedded Systems）是三维图形 API OpenGL 的子集，针对手机、PDA 和游戏主机等嵌入式设备而设计。

Android 对应 OpenGL ES 的版本支持如下：
- Android 1.0 开始支持 OpenGL ES 1.0 及 1.1
- Android 2.2 开始支持 OpenGL ES 2.0
- Android 4.3 开始支持 OpenGL ES 3.0
- Android 5.0 开始支持 OpenGL ES 3.1

其中 OpenGL ES 1.0 是以 OpenGL 1.3 规范为基础的，OpenGL ES 1.1 是以 OpenGL 1.5 规范为基础的，而 OpenGL ES 2.0 基于 OpenGL 2.0 实现。

2.x 版本相比 1.x 版本有较大差异，1.x 版本为 fixed function pipeline，即固定管线硬件，而 2.x 版本为 programmable pipeline，可编程管线硬件。

固定管线中原本由系统做的一部分工作，在可编程管线中必须需要自己写程序实现，具体程序为 vertex shader（顶点着色器）和 fragment shader（片元着色器）。

#### OpenGL 上下文
OpenGL 是一个仅仅关注图像渲染的图像接口库，在渲染过程中它需要将顶点信息、纹理信息、编译好的着色器等渲染状态信息存储起来，而存储这些信息的数据结构就可以看作 OpenGL 的上下文。

调用任何 OpenGL 函数前，必须已经创建了 OpenGL Context，GL Context 存储了 OpenGL 的状态变量以及其他渲染有关的信息。

OpenGL 是个状态机，有很多状态变量，是个标准的过程式操作过程，改变状态会影响后续所有操作，这和面向对象的解耦原则不符，毕竟渲染本身就是个复杂的过程。

OpenGL 采用 Client-Server 模型来解释 OpenGL 程序，即 Server 存储 GL Context（可能不止一个），Client 提出渲染请求，Server 给予响应，一般 Server 和 Client 都在我们的 PC 上，但 Server 和 Client 也可以是通过网络连接。

之后的渲染工作就要依赖这些渲染状态信息来完成，当一个上下文被销毁时，它所对应的 OpenGL 渲染工作也将结束。

#### EGL
在 OpenGL 的设计中，OpenGL 是不负责管理窗口的，窗口的管理交由各个设备自己来完成，具体来讲，IOS 平台上使用 EAGL 提供本地平台对 OpenGL 的实现，在 Android 平台上使用 EGL 提供本地平台对 OpenGL 的实现。

EGL 是 OpenGL ES 和 Android 底层平台视窗系统之间的接口，在 OpenGL 的输出与设备屏幕之间架接起一个桥梁，承担了为 OpenGL 提供上下文环境以及管理窗口的职责。

EGL 为双缓冲工作模式，即有一个 Back Frame Buffer 和一个 Front Frame Buffer，正常绘制的目标都是 Back Frame Buffer，绘制完成后再调用 eglSwapBuffer API，将绘制完毕的 FrameBuffer 交换到 Front Frame Buffer 并显示出来。

从代码层面来看，OpenGL ES 的 opengles 包下定义了平台无关的绘图指令，EGL（javax.microedition.khronos.egl）
则定义了控制 displays，contexts 以及 surfaces 的统一的平台接口。

- Display（EGLDisplay） 是对实际显示设备的抽象
- Surface（EGLSurface）是对用来存储图像的内存区域 FrameBuffer 的抽象，包括 Color Buffer、Stencil Buffer、Depth Buffer
- Context（EGLContext）存储 OpenGL ES 绘图的一些状态信息

![](/img/1.jpeg)

*使用 EGL 绘图的一般步骤：*

1. 获取 EGLDisplay 对象
2. 初始化与 EGLDisplay 之间的连接
3. 获取 EGLConfig 对象
4. 创建 EGLContext 实例
5. 创建 EGLSurface 实例
6. 连接 EGLContext 和 EGLSurface
7. 使用 GL 指令绘制图形
8. 断开并释放与 EGLSurface 关联的 EGLContext 对象
9. 删除 EGLSurface 对象
10. 删除 EGLContext 对象
11. 终止与 EGLDisplay 之间的连接


一般来说在 Android 平台上开发 OpenGL ES 应用，无需按照上述步骤来绘制图形，可以直接使用 GLSurfaceView 控件，该控件提供了对 Display、Surface 以及 Context 的管理，大大简化了开发流程。

#### OpenGL 纹理

纹理（Texture）是一个 2D 图片（甚至也有 1D 和 3D 的纹理），它可以用来添加物体的细节；你可以想象纹理是一张绘有砖块的纸，无缝折叠贴合到你的 3D 的房子上，这样你的房子看起来就像有砖墙外表了。

因为我们可以在一张图片上插入非常多的细节，这样就可以让物体非常精细而不用指定额外的顶点。

## OpenGL 坐标系理解

OpenGL 要求输入的顶点坐标都是标准化设备坐标，即每个顶点的 x、y、z 都在 -1 到 1 之间，由标准化设备坐标转换为屏幕坐标的过程中会经历变换多个坐标系统，在这些特定的坐标系中，一些操作和计算可以更加方便。

![](/img/15.jpg)

#### 局部坐标
顶点坐标起始于局部空间（Local Space），在这里称为局部坐标，是以物体某一点为原点而建立的，该坐标系仅对该物体适用，用来简化对物体各部分坐标的描述。物体放到场景中时，各部分经历的坐标变换相同，相对位置不变。

#### 世界坐标
局部坐标通过模型矩阵进行位移、缩放、旋转，将物体从局部变换到世界空间，并和其他物体一起相对于世界的原点摆放。

#### 观察坐标
将世界空间坐标转化为用户视野前方的坐标，通常是由一系列的位移和旋转的组合（观察矩阵）来完成。

#### 裁剪坐标
坐标到达观察空间之后，通过投影矩阵会将指定范围内的坐标变换为标准化设备坐标的范围(-1.0, 1.0)，所有在范围外的坐标会被裁剪掉。

#### 屏幕坐标
将裁剪坐标位于(-1.0, 1.0)范围的坐标变换到由 glViewport 函数所定义的坐标范围内，最后变换出来的坐标将会送到光栅器，将其转化为片段。


## OpenGL 渲染管线
OpenGL 渲染管线流程为：顶点数据 -> 顶点着色器 -> 图元装配 -> 几何着色器 -> 光栅化 -> 片段着色器 -> 逐片段处理 -> 帧缓冲

![](/img/16.jpg)

OpenGL 渲染管线的流程其实就是 OpenGL 引擎渲染图像的流程，也就是说 OpenGL 引擎一步一步的将图片渲染到屏幕上的过程，渲染管线可以分为以下几个阶段：

#### 1.指定几何对象
首先要了解几何图元的概念，几何图元就是点、直线、三角线等几何对象，在提供了顶点坐标后，还要确定具体要画的是点、线段还是三角形，这就要确定具体执行的绘制指令。比如 OpenGL 提供给开发者的绘制方法 glDrawArrays，这个方法的第一个参数就是指定绘制方式，可选值有：

**GL_POINTS**：以点的形式进行绘制，通常用在绘制粒子效果的场景。
**GL_LINES**：以线的形式进行绘制，通常用于绘制直线的场景。
**GL_TRIANGLE_STRIP**：以三角形的形式进行绘制，所有二维图像的渲染都会使用这种方式。

具体选用哪一种绘制方式决定了 OpenGL 渲染管线的第一阶段应如何去绘制几何图元，这就是第一阶段指定几何对象。

#### 2.顶点处理
不论上面的几何图元是如何指定的，所有的几何数据都将会通过这个阶段。这个阶段的操作内容有：根据模型视图（即根据几何图元创建的物体）和投影矩阵进行变换来改变顶点的位置，根据纹理坐标与纹理矩阵来改变纹理坐标的位置，如果设计三维的渲染，还要处理光照计算和法线变换。

关键的操作就是顶点坐标变换及光照处理，每个顶点是分别单独处理的。这个阶段所接受的数据是每个顶点的属性特征，输出的则是变换后的顶点数据。

#### 3.图元组装
在顶点处理之后，顶点的全部属性都已经被确定。在这个阶段顶点将会根据应用程序设定的图元规则如 GL_POINTS 、GL_TRIANGLES(三角形) 等被组装成图元。

#### 4.珊格化操作
在图元组装后会传递过来图元数据，到目前为止，这些图元信息还只是顶点而已：顶点处都还没有“像素点”、直线段端点之间是空的、多边形的边和内部也是空的，光栅化的任务就是构造这些。

这个阶段会将图元数据分解成更小的单元并对应于帧缓冲区的各个像素，这些单元称为片元，一个片元可能包含窗口颜色、纹理坐标等属性。

片元的属性则是图元上的顶点数据等经过插值而确定的，这就是珊格化操作，也就是确定好每一个片元是什么。

#### 5.片元处理
珊格化操作构造了像素点，这个阶段就是处理这些像素点，根据自己的业务处理（比如提亮、饱和度调节、对比度调节、高斯模糊等）来变换这个片元的颜色。

#### 6.逐片段处理
进行剪切、Alpha 测试、 模版测试、深度测试、混合等处理，这些操作将会最后影响其在帧缓冲区的颜色值。

#### 7.帧缓冲操作
此阶段主要执行帧缓冲的写入操作，也是渲染管线的最后一步，负责将最终的像素点写到帧缓冲区。

上面提到 OpenGL ES 2.0 版本相比之前版本，提供了可编程的着色器来代替 1.x 版本渲染管线的某些阶段，具体为：

- Vertex Shader（顶点着色器）用于替换顶点处理阶段
- Fragment Shader（片元着色器）用于替换片元处理阶段

## OpenGL 着色语言
OpenGL 着色语言 GLSL 全称为 OpenGL Shading Language，是为了实现着色器的功能而向开发人员提供的一种开发语言，语法与 C 语言类似，下面分为以下几点来学习 GLSL：

#### 1.基本数据类型

- void：空类型，即不返回任何值
- bool：布尔类型，true/false
- int：带符号的整数，signed integer
- float：带符号的浮点数，signed scalar
- vec2、vec3、vec4：n-维浮点数向量
- bvec2、bvec3、bvec4：n-维布尔向量
- ivec2、ivec3、ivec4：n-维整数向量
- mat2、mat3、mat4：2x2、3x3、4x4 浮点数矩阵
- sampler2D：2D 纹理
- samplerCube：盒纹理

*其中 float 可指定精度：*

- high：32bit，一般用于顶点坐标
- medium：16bit，一般用于纹理坐标
- low：8bit，一般用于颜色表示

#### 2.变量修饰符

- none：(默认的可省略)本地变量，可读可写，函数的输入参数既是这种类型
- const：声明变量或函数的参数为只读类型
- attribute：用于保存顶点或法线数据,它可以在数据缓冲区中读取数据，仅能用于顶点着色器
- uniform：在运行时 shader 无法改变 uniform 变量，一般用来放置程序传递给 shader 的变换矩阵，材质，光照参数等等，可用于顶点着色器和片元着色器
- varying：用于修饰从顶点着色器向片元着色器传递的变量


要注意全局变量限制符只能为 const、attribute、uniform 和 varying 中的某一个，不可复合。

#### 3.内置变量

GLSL 程序使用一些特殊的内置变量与硬件进行沟通，他们大致分成两种，一种是 input 类型,他负责向硬件(渲染管线)发送数据；另一种是 output 类型，负责向程序回传数据，以便编程时需要。

*顶点着色器中 output 类型的内置变量如下：*
- highp vec4  gl_Position：放置顶点坐标信息
- mediump float gl_PointSize：需要绘制点的大小,(只在gl.POINTS模式下有效)

*片元着色器中 input 类型的内置变量如下：*
- mediump vec4 gl_FragCoord;：片元在 framebuffer 画面的相对位置
- bool gl_FrontFacing：标志当前图元是不是正面图元的一部分
- mediump vec2 gl_PointCoord：经过插值计算后的纹理坐标,点的范围是0.0到1.0

*片元着色器中 output 类型的内置变量如下：*
- mediump vec4 gl_FragColor：设置当前片点的颜色
- mediump vec4 gl_FragData[n]：设置当前片点的颜色,使用glDrawBuffers数据数组

#### 4.内置常量

GLSL 提供了一些内置的常量，用来说明当前系统的一些特性。有时我们需要针对这些特性，对 shader 程序进行优化，让程序兼容度更好。

*顶点着色器中的内置常量如下：*
- const mediump int gl_MaxVertexAttribs >= 8：顶点着色器中可用的最大 attributes 数
- const mediump int gl_MaxVertexUniformVectors >= 128：顶点着色器中可用的最大 uniform vectors 数
- const mediump int gl_MaxVaryingVectors >= 8：顶点着色器中可用的最大 varying vectors 数
- const mediump int gl_MaxVertexTextureImageUnits >= 0：顶点着色器中可用的最大纹理单元数
- const mediump int gl_MaxCombinedTextureImageUnits >= 8：表示最多支持多少个纹理单元

*片元着色器中的内置常量如下：*
- const mediump int gl_MaxTextureImageUnits >= 8：片元着色器中能访问的最大纹理单元数
- const mediump int gl_MaxFragmentUniformVectors >= 16：片元着色器中可用的最大 uniform vectors 数
- const mediump int gl_MaxDrawBuffers = 1：表示可用的 drawBuffers 数,在 OpenGL ES 2.0 中这个值为 1, 在将来的版本可能会有所变化

上面这些值的大小取决于 OpenGL ES 在某设备上的具体实现。

#### 5.内置函数

- 通用函数：abs、floor、min、max 等，参数可传入 float/vec2/vec3/vec4 类型
- 角度函数：sin、cos 等，参数可传入 float/vec2/vec3/vec4 类型
- 指数函数：pow、log 等，参数可传入 float/vec2/vec3/vec4 类型
- 几何函数：distance、dot 等，参数可传入 float/vec2/vec3/vec4 类型
- 矩阵函数：matrixCompMult，参数传入 mat 类型
- 向量函数：lessThan、equal 等，参数可传入 vec2/vec3/vec4 类型
- 纹理函数：texture2D、texture2DProj 等


