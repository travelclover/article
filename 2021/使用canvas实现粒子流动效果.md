# 使用canvas实现粒子流动效果
首先我们先来看下实现的效果吧~  
<video id="video" controls="controls"  src="https://travelclover.github.io/img/2021/07/canvas实现粒子流动效果.mp4">
</video>
![canvas实现粒子流动效果.gif](https://travelclover.github.io/img/2021/07/canvas实现粒子流动效果.gif)  
这是一个简单的粒子流向效果，实现起来也非常简单，该效果可以用在管道流向、风场变化、以及其它需要表达方向、速度的可视化效果展示场景中。  

该效果中的难点只有一个，就是粒子的拖尾效果。  
![粒子拖尾效果](https://travelclover.github.io/img/2021/07/粒子拖尾效果.png)

## 粒子拖尾效果实现
实现粒子拖尾效果需要用到canvas中的一个知识点，`globalCompositeOperation`字段。该属性用来设置如何将一个源（新的）图像绘制到目标（已有）图像上。  
> ***注意***：*源图像 = 您打算放置到画布上的绘图；目标图像 = 您已经放置在画布上的绘图。*  

`globalCompositeOperation`字段有很多属性值，默认为'`source-over`'，表示在目标图像上显示源图像。在本示例中我们会用到'`destination-in`'和'`lighter`'这两个属性值。
> destination-in: 在源图像中显示目标图像。只有源图像内的目标图像部分会被显示，源图像是透明的。  
> lighter: 显示源图像 + 目标图像。  

每次调用绘制方法，都要执行以下代码:
```javascript
this.ctx.globalCompositeOperation = 'destination-in'; // 设置模式为：目标图形和源图形重叠的部分会被保留（源图形），其余显示为透明
this.ctx.fillRect(0, 0, this.canvasWidth, this.canvasHeight); // 绘制矩形
this.ctx.globalCompositeOperation = 'lighter'; // 设置模式为: 源图像 + 目标图像。重叠部分的颜色会重新计算
this.ctx.globalAlpha = 0.8; // 设置画布上绘制图形的不透明度
```
实现半透明的拖尾效果原理就好比是在一张纸上画了一幅画，然后又盖了一张半透明的纸上去继续作画，被压在下面的画就会慢慢变淡直到看不清，但最上面的画就很清晰。在该示例中，每画一帧，就相当于在原直线的前方又画了一条新的直线，因为是新画的，所以也是最清楚的，所以也是最亮的，旧的直线就像蒙上了一层纸，慢慢变暗，所以越旧的线段越暗淡。画面连贯起来，给人的感觉就像是粒子在前进，后面的尾巴在慢慢消失。上面代码中的'`fillRect`'方法就类似于覆盖一张纸的操作，'globalAlpha = 0.8'就类似于将覆盖的纸设置为半透明的纸。

## 粒子向前移动
在创建粒子类的时候，我们需要设置粒子当前位置坐标、以及下次绘制时的位置坐标。在每次绘制前，更新当前位置坐标以及计算下次位置坐标。绘制时就只用绘制当前位置坐标到下次位置坐标这条线段。  
为了让粒子运动起来显得更加有“灵性”，我们可以增加速度增量字段，并在更新粒子状态的方法中随机修改增量速度，让粒子看起来是在做无规则运动，而不是一条直线~  

```javascript
// 粒子类
class Particle {
  constructor(x, y) {
    this.x = x; // 坐标x值
    this.y = y; // 坐标y值
    this.speedRate = 0.4; // 速率，用来控制粒子流动的快慢
    this.speedX = 0; // 速度在x方向上的增量
    this.speedY = 0; // 速度在y方向上的增量
    this.lifetime = 1 + Math.random() * 800; // 粒子生命周期，每次更新都会减小
    this.nextX = x + this.speedX; // 接下来粒子的x坐标
    this.nextY = y + this.speedY; // 接下来粒子的y坐标
  }

  // 更新粒子的方法
  update() {
    this.x = this.nextX; // 更新x坐标
    this.y = this.nextY; // 更新y坐标
    this.speedX += (Math.random() * 2 - 1) * this.speedRate; // x方向增量
    this.speedY += (Math.random() * 2 - 1) * this.speedRate; // y方向增量
    this.nextX = this.x + this.speedX; // 计算接下来粒子的x坐标
    this.nextY = this.y + this.speedY; // 计算接下来粒子的y坐标
    this.lifetime--; // 生命周期减1
  }
}
```

## 完整代码
如需查看示例效果可点击下载[完整代码](https://download.csdn.net/download/qq_37155408/20032384)。代码中包含详细的注释，方便大家的理解。  

---
如果该文章对您有所帮助，请您一定不要吝啬您的鼓励。点赞、评论、分享、收藏、打赏都是您对我的鼓励和支持。  
如果您有`GitHub`账号，还可以[关注我~](https://github.com/travelclover)  
最后，感谢大家的阅读，如有错误，还请各位批评指正。
