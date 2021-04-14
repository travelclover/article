# Three.js实现抛物线动态流向效果
老规矩，首先来看下最终实现的效果。  
![抛物线流向动态效果图](https://travelclover.github.io/img/2021/04/抛物线动态流向效果.gif)
该效果可能会用在飞机航线起点到终点的演示、导弹发射轨迹效果演示等场景。掌握思路以后，效果实现起来非常简单，总结下来要点其实就两个：1.绘制抛物线；2.实现流向动态效果。下面就来看看具体实现过程吧。  

## 创建三维场景
首先我们要创建一个三维场景，创建场景非常简单，仅仅一句代码就能实现。
```javascript
const scene = new THREE.Scene();
```
创建了场景后，我们还不能显示任何东西，因为还需要用到渲染器和相机。
```javascript
// 创建相机
const camera = new THREE.PerspectiveCamera(
  75,
  window.innerWidth / window.innerHeight,
  0.1,
  1000
);
camera.position.z = 50; // 设置相机位置
```
three.js中相机有很多种，这里用到的是`PerspectiveCamera`相机。该类型是透视相机，可以用来模拟人眼所看到的景象，所以大多数三维场景渲染中都是使用的该相机。点击查看[文档详情](https://threejs.org/docs/index.html#api/en/cameras/PerspectiveCamera)。

```javascript
// 创建渲染器
const renderer = new THREE.WebGLRenderer();
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);
renderer.render(scene, camera);
```
创建渲染器后，设置输出画布大小。具体属性和方法可点击[查看文档](https://threejs.org/docs/index.html#api/en/renderers/WebGLRenderer)。  

然后需要添加一个方法，每隔一段时间调用该方法去渲染场景。
```javascript
function animate() {
  renderer.render(scene, camera);
  requestAnimationFrame(animate);
}
// 必须调用一次
animate();
```
该方法中会用到一个全局方法`requestAnimationFrame`，不熟悉的请点击[查看文档](https://developer.mozilla.org/zh-CN/docs/Web/API/Window/requestAnimationFrame)。

```javascript
// 添加坐标轴辅助工具
const axesHelper = new THREE.AxesHelper(100);
scene.add(axesHelper);
```
为了能够更直观的感受三维空间感，我们还可以在场景中添加一个坐标轴辅助工具。红、绿、蓝分别表示x轴、y轴、z轴。如下图所示。  
![坐标轴辅助工具.png](https://travelclover.github.io/img/2021/04/坐标轴辅助工具.png)

到现在为止，可以看到场景中的坐标轴了，但是我们通过拖动鼠标发现并不能移动和旋转场景，那是因为我们还没有添加控制器。控制器模块需要单独引入，可以在[three.js代码仓库](https://github.com/mrdoob/three.js)中找到相关代码。
```javascript
// 创建控制器
const controls = new OrbitControls(camera, renderer.domElement);
controls.enableDamping = true; // 启用阻尼（惯性），这将给控制器带来重量感，如果该值被启用，必须在动画循环里调用.update()
controls.dampingFactor = 0.05; // 阻尼惯性大小
```
**注意：** 当`controls.enableDamping`为`true`时，必须在动画循环里调用`controls.update()`方法。  
```javascript
function animate() {
  // controls.enableDamping设为true时（启用阻尼），必须在动画循环里调用.update()
  controls.update();

  renderer.render(scene, camera);
  requestAnimationFrame(animate);
}
```

## 生成抛物线
生成一条抛物线需要用到three.js中的`QuadraticBezierCurve3`类。该类可以根据三个点坐标（起点、终点和一个控制点）创建一条平滑的三维二次贝塞尔曲线。点击[查看文档](https://threejs.org/docs/index.html#api/en/extras/curves/QuadraticBezierCurve3)。  
```javascript
const point1 = [50, 0, 0]; // 点1坐标
const point2 = [-50, 0, 0]; // 点2坐标
const controlPoint = [0, 50, 0]; // 控制点坐标

// 创建三维二次贝塞尔曲线
const curve = new THREE.QuadraticBezierCurve3(
  new THREE.Vector3(point1[0], point1[1], point1[2]),
  new THREE.Vector3(controlPoint[0], controlPoint[1], controlPoint[2]),
  new THREE.Vector3(point2[0], point2[1], point2[2])
);
```
得到抛物线后，我们不能直接就用这个生成的曲线，因为我们还要实现流动的效果，该效果实现的原理是把曲线分割成很多条小线段，其中一条线段的颜色为高亮色，其余的线段颜色为暗色，每隔一定的时间就将高亮线段的前一线段颜色改为高亮色，原高亮线段改为暗色，这样通过设置合理的间隔时间，高亮线段的变化就连贯起来了，看起来就是流动的效果。  

`QuadraticBezierCurve3`类继承于`Curve`类，上面有个`getPoints()`方法，该方法接收一个`number`类型的参数，作用是将曲线分割成指定的数量，并返回一个包含全部分割点的数组，数组长度为传入的参数大小加`1`。比如传入的参数是10，意思是将曲线分割成10段，那么返回的分割点数量就为11（1个起点、中间的9个分割点、1个终点）。具体可点击查看`Curve`类的[文档详情](https://threejs.org/docs/index.html#api/en/extras/core/Curve)。  

```javascript
const divisions = 30; // 曲线的分段数量
const points = curve.getPoints(divisions); // 返回 分段数量 + 1 个点，例如这里的points.length就为31
```
现在我们得到了31个点坐标，用这些点作为顶点创建`Geometry`。
```javascript
// 创建Geometry
const geometry = new THREE.Geometry();
geometry.vertices = points; // 将上一步得到的点列表赋值给geometry的vertices属性
```
创建了geometry我们还要给它赋值一个顶点颜色数组，因为几何体的材质需要设置成顶点着色，后面要实现改变线段颜色的需求时，我们就只用改变顶点颜色数组就行。
```javascript
// 设置顶点 colors 数组，与顶点数量和顺序保持一致。
geometry.colors = new Array(points.length).fill(
  new THREE.Color('#333300')
);
```
**注意：** `colors`的数量和顺序是和`vertices`一一对应的。colors中的第一个就是对应vertices中第一个顶点的颜色。  
有了几何体，我们还需要材质才能创建网格。需要注意的是，我们需要将材质设置为顶点着色。
```javascript
// 生成材质
const material = new THREE.LineBasicMaterial({
  vertexColors: THREE.VertexColors, // 顶点着色
  transparent: true, // 定义此材质是否透明
  side: THREE.DoubleSide,
});
const mesh = new THREE.Line(geometry, material);
```
到现在我们就已经创建好了抛物线，一定不要忘了将它添加进场景中去，这样才能渲染出来！
```javascript
scene.add(mesh);
```

## 实现动态流向效果
上面提到过，我们要实现动态的流向效果，只需要每隔一段时间更新顶点颜色数组就可以了。所以我们需要在`animate`方法中添加更新顶点颜色的代码。
```javascript
let colorIndex = 0; // 高亮颜色流动的索引值
let timestamp = 0; // 时间戳

function animate() {
  controls.update();
  // 时间间隔
  let now = new Date().getTime();
  if (now - timestamp > 30) {
    geometry.colors = new Array(divisions + 1)
      .fill(new THREE.Color('#333300'))
      .map((color, index) => {
        if (index === colorIndex) {
          return new THREE.Color('#ffff00');
        }
        return color;
      });
    // 如果geometry.colors数据发生变化，colorsNeedUpdate值需要被设置为true
    geometry.colorsNeedUpdate = true;
    timestamp = now;
    colorIndex++;
    if (colorIndex > divisions) {
      colorIndex = 0;
    }
  }
  renderer.render(scene, camera);
  requestAnimationFrame(animate);
}
```
**注意：** 改变了顶点颜色数组后一定要将`geometry.colorsNeedUpdate`属性设置为`true`，这样three.js才知道顶点颜色数组发生了变化，非常重要！！！

现在，我们就完成了所有代码，实现了抛物线动态流向效果。  
![抛物线流向动态效果图](https://travelclover.github.io/img/2021/04/抛物线动态流向效果.gif)

点击查看[在线示例](https://travelclover.github.io/demo/example/Three.js实现抛物线动态流向效果)。  
点击查看完整[示例代码](https://github.com/travelclover/demo/blob/gh-pages/example/Three.js实现抛物线动态流向效果.html)。  

如果该文章对您有所帮助，请您一定不要吝啬您的鼓励。点赞、评论、分享、收藏、打赏都是您对我的鼓励和支持。  
如果您有`GitHub`账号，还可以[关注我~](https://github.com/travelclover)  
最后，感谢大家的阅读，如有错误，还请各位批评指正。
