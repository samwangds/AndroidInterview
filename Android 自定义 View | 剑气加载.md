> 看到一个非常炫的加载效果，文章中大佬是通过 css 实现的，今天咱们来用 Android 的自定义 View 来实现一下
>
> https://juejin.cn/post/7001779766852321287

#### 这是理想中的效果

![src=http___image.17173.com_bbs_v1_2012_12_01_1354372326576.gif&refer=http___image.17173.gif](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c61adacf68a542b5889a532205d6a552~tplv-k3u1fbpfcp-watermark.awebp)

#### 这是现实的效果

![circle 11](img/circle 11.gif)

不能说百分百还原吧，只能说精髓

<img src="https://gimg2.baidu.com/image_search/src=http%3A%2F%2Fimg9.doubanio.com%2Fview%2Fgroup_topic%2Fl%2Fpublic%2Fp202403164.jpg&refer=http%3A%2F%2Fimg9.doubanio.com&app=2002&size=f9999,10000&q=a80&n=0&g=0n&fmt=jpeg?sec=1634745833&t=88662a8ded11a3b527dafb1296c3a0a2" alt="img" style="zoom:50%;" />

## 开工

这个效果仔细看，就是有三个类似月牙形状的元素进行循环转动，我们只需要拆解出一个月牙来做效果即可，最后再将三个月牙组合起来就可以达到最终效果了

## 月牙

#### 先画一个圆

<img src="img/image-20210921002044181.png" alt="image-20210921002044181" style="zoom:50%;" />

#### 再画个大一丢丢的

![image-20210921002129703](img/image-20210921002129703.png)

#### 再把这个大圆往右移一丢丢,裁切出来的左右两个都是月牙

![image-20210921002239797](img/image-20210921002239797.png)

#### 实现

老司机应该一眼就能看出来，只要在两次绘制圆中只要使用一个叠加模式就能达到裁剪出一个月牙的效果了。那么是什么模式呢，我去搜索一下~

当当当，就是它~ `PorterDuff.Mode.DST_OUT`

![image-20210921002857005](img/image-20210921002857005.png)

相关源码如下：

```kotlin
canvas.drawColor(Color.BLACK)

val layerId =
    canvas.saveLayer(0f, 0f, width.toFloat(), height.toFloat(), null, Canvas.ALL_SAVE_FLAG)
val halfW = width / 2f
val halfH = height / 2f
val radius = min(width, height) / 3f

paint.color = Color.WHITE
canvas.drawCircle(halfW, halfH, radius, paint)
paint.color = Color.BLACK
paint.xfermode = xfermode
canvas.drawCircle(halfW, halfH - 0.05f * radius, radius * 1.01f, paint)
canvas.restoreToCount(layerId)
paint.xfermode = null
```

#### 运行起来我们就得到了一弯浅浅的月牙

![image-20210921004428266](img/image-20210921004428266.png)

## 立体空间变化

我们可以看出效果图里的每一个月牙并不是那么方正，而是有一定的空间旋转，再加上绕着 Z 轴旋转。这里需要利用 Camera 与 Matrix 实现3D效果（相关知识可参考：https://www.jianshu.com/p/34e0fe5f9e31）

#### 我们先给它在 x 轴转 35 度 ，y 轴转 -45 度（参考开头文章的数据）

```kotlin
rotateMatrix.reset()
camera.save()
camera.rotateX(35F)
camera.rotateY(-45F)
camera.getMatrix(rotateMatrix)
camera.restore()
val halfW = width / 2f
val halfH = height / 2f

rotateMatrix.preTranslate(-halfW, -halfH)
rotateMatrix.postTranslate(halfW, halfH)
canvas.concat(rotateMatrix)
```

运行效果如下，从普通的月牙变成了帅气的剑气

![image-20210921010008348](img/image-20210921010008348.png)



## 动画

我们上面做了固定角度的 X, Y 轴的旋转，这个时候我们只要加上一个 Z 轴的调转动画，这个剑气就动起来了。

```kotlin
val anim = ValueAnimator.ofFloat(0f, -360f).apply {
    // Z 轴是逆时针，取负数，得到顺时针的旋转
    interpolator = null
    repeatCount = RotateAnimation.INFINITE
    duration = 1000

    addUpdateListener {
        invalidate()
    }
}
// 省略前面已写代码...
camera.rotateZ(anim.animatedValue as Float)

// 在合适的地方启动动画
view.anim.start()
```

运行效果：

![circle 12](img/circle 12.gif)

## 举一反三

有了这一个完整的剑气旋转，只要再来两道，组成完整的剑气加载就可以了。

#### 将前面的代码抽象成一个方法

```KOTLIN
private fun drawSword(canvas: Canvas, rotateX: Float, rotateY: Float) {
    val layerId =
        canvas.saveLayer(0f, 0f, width.toFloat(), height.toFloat(), null, Canvas.ALL_SAVE_FLAG)
    rotateMatrix.reset()
    camera.save()
    camera.rotateX(rotateX)
    camera.rotateY(rotateY)
    camera.rotateZ(anim.animatedValue as Float)
    camera.getMatrix(rotateMatrix)
    camera.restore()

    val halfW = width / 2f
    val halfH = height / 2f

    rotateMatrix.preTranslate(-halfW, -halfH)
    rotateMatrix.postTranslate(halfW, halfH)
    canvas.concat(rotateMatrix)
    canvas.drawCircle(halfW, halfH, radius, paint)
    paint.xfermode = xfermode
    canvas.drawCircle(halfW, halfH - 0.05f * radius, radius * 1.01f, paint)
    canvas.restoreToCount(layerId)
    paint.xfermode = null
}
```

#### 绘制三道剑气

``` kotlin
verride fun onDraw(canvas: Canvas) {
   super.onDraw(canvas)
   canvas.drawColor(Color.BLACK)
   // 偏移角度来源开关文章
   drawSword(canvas,35f, -45f)
   drawSword(canvas,50f, 10f)
   drawSword(canvas,35f, 55f)
}
```

#### 跑起来看看

![circle 13](img/circle 13.gif)

Emm... 这动画也太整齐划一了

#### 错开三道剑气

在 Z 轴旋转上，我们给每道剑气一个初始值的旋转值（360/3 = 120），这样它们就能均匀的错开了。

相关实现如下：

```kotlin
private fun drawSword(canvas: Canvas, rotateX: Float, rotateY: Float, startValue: Float) {
   //... 省略未改动代码
   camera.rotateZ(anim.animatedValue as Float + startValue)
   //... 省略未改动代码
}

override fun onDraw(canvas: Canvas) {
    super.onDraw(canvas)
    canvas.drawColor(Color.BLACK)
    drawSword(canvas,35f, -45f, 0f)
    drawSword(canvas,50f, 10f, 120f)
    drawSword(canvas,35f, 55f, 240f)
}
```

#### 最终效果

![circle 14](img/circle 14.gif)

和我们开头预期的效果图一模一样

完整代码：https://github.com/samwangds/DemoFactory/blob/master/app/src/main/java/demo/com/sam/demofactory/view/SwordLoadingView.kt

## End 吹欺汀

值此中秋佳节，祝各位有知识又有头发~

<img src="https://pic3.zhimg.com/80/v2-253613b5dd4140a046b5b24bf8f3c5d7_1440w.jpg?source=1940ef5c" alt="img" style="zoom:50%;" />