# Three.js实现简单开门动画
先来看一下简单的开门动画实现效果，如下图所示：
![](https://github.com/travelclover/img/blob/master/2020/03/doorOpen.gif)  
可以看得出这个开门动画还是比较简单的，主要关键点有2个地方：
1.  实现门围绕右门框旋转。
2.  利用关键帧实现动画。
接下来主要讲一下这两个关键点。

## 1.实现门围绕右门框旋转
Three.js中物体的默认旋转中心为物体自身的中心，例如有一扇门（实际上是一个宽为10，高为20，深度为0.5的立方体），如果让它绕Y轴旋转90度，也就是设置`rotation.y`属性为`Math.PI/2`，结果如下图所示：
![](https://github.com/travelclover/img/blob/master/2020/03/door.rotation.y.jpg)  
很显然，大多数门不应该这样旋转。那么我们应该怎么做才能让门绕着指定轴旋转呢？这时候我们可以在`门`这个对象外面包一层`Object3D`对象，让外层物体的旋转中心位置在原本想要旋转的指定位置上，再调整门在外层物体中的相对位置，使门的右边缘在外层物体的中心位置上。
```javascript
// 创建门
const door = new THREE.BoxBufferGeometry(10, 20, 0.5);
const doorMaterial = new THREE.MeshLambertMaterial({ color: 0xd88c00 });
const doorMesh = new THREE.Mesh(door, doorMaterial);
// 实现门围绕特定轴旋转
const group = new THREE.Group(); // 外层对象
group.position.set(5, 10, 0); // 设置外层对象的中心为原本想要旋转的位置
group.add(doorMesh); // 把'门'添加进外层对象中
doorMesh.position.set(-5, 0, 0); // 调整门在外层对象中的相对位置
```
> `Group`对象和`Object3D`对象几乎是相同的，其目的是使得组中对象在语法上的结构更加清晰。

## 2.利用关键帧实现动画
先给用于关键帧动画的网格模型命名。
```javascript
group.name = 'door'; // 外层网格模型命名为door
```
> 动画将作用于外层对象`group`上，因为`group`包含`doorMesh`对象，所以`group`旋转时`doorMesh`也会旋转。

接着需要编辑关键帧，需要用到关键帧轨道[`(KeyframeTrack)`](https://threejs.org/docs/index.html#api/en/animation/KeyframeTrack) API。
```javascript
const times = [0, 3]; // 关键帧时间数组，单位'秒'
const rotationValues = [0, -Math.PI / 2]; // 需要转动的角度
 // 创建关键帧轨道
const rotationTrack = new THREE.KeyframeTrack(
    'door.rotation[y]', // 指定对象中的变形目标为Y轴旋转属性
    times, // 关键帧的时间数组
    rotationValues // 与时间数组中的时间点相关的值组成的数组
);
```
然后使用[`AnimationClip`](https://threejs.org/docs/index.html#api/en/animation/AnimationClip) API剪辑动画。
```javascript
const duration = 3; // 持续时间，单位'秒'
// 动画剪辑
const clip = new THREE.AnimationClip(
    'open', // 此剪辑的名称
    duration, // 如果传入负数，持续时间将会从传入的数组中计算得到
    [rotationTrack] // 一个由关键帧轨道（KeyframeTracks）组成的数组。
);
```
> `duration`参数决定了动画的播放时间，偏小则编辑的帧动画将不能完全播放，偏大则帧动画播放完后会继续空播放，所以一般设置为关键帧时间数组的最大值。  
> 第三个参数为数组，可以剪辑多个关键帧轨道，意思是可以同时进行多个不同的动画，比如在物体旋转的同时可以改变材质的颜色或者改变几何体的大小等。

最后就是播放编辑好的关键帧数据，需要用到动画混合器[`AnimationMixer`](https://threejs.org/docs/index.html#api/en/animation/AnimationMixer) API。
```javascript
const mixer = new THREE.AnimationMixer(group); // 动画混合器
const AnimationAction = mixer.clipAction(clip); // 返回所传入的剪辑参数的AnimationAction
AnimationAction.timeScale = 1; // 可以调节播放速度，默认是1。为0时动画暂停。值为负数时动画会反向执行。
AnimationAction.play(); // 开始播放
```
> `AnimationMixer`需要传入的参数为混合器播放的动画所属的对象，在这里也就是外层对象`group`。
> [`clipAction`](https://threejs.org/docs/index.html#api/en/animation/AnimationMixer.clipAction)方法返回所传入的剪辑参数的 [AnimationAction](https://threejs.org/docs/index.html#api/en/animation/AnimationAction)，第一个参数可以是动画剪辑(AnimationClip)对象或者动画剪辑的名称，所以这里也可以传入剪辑的名称“`open`”。

设置完以上所有内容后，你会发现动画还是没有变化，那是因为还有一个很重要的步骤没有做。我们需要在渲染函数中`render()`调用动画混合器`mixer`的[`update`](https://threejs.org/docs/index.html#api/en/animation/AnimationMixer.update)方法，`update`方法需要传入两帧渲染的间隔时间，需要用到[`Clock`](https://threejs.org/docs/index.html#api/en/core/Clock)对象的[`getDelta`](https://threejs.org/docs/index.html#api/en/core/Clock.getDelta)方法。
```javascript
const clock = new THREE.Clock(); // 创建时钟
```
在render方法中添加：
```javascript
mixer.update(clock.getDelta()); // 更新动画
```
现在我们的动画就已经动了起来。  

***
**以下是完整代码：**
```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Three.js实现简单开门动画</title>
    <style>
      * {
        margin: 0;
        padding: 0;
        overflow: hidden;
      }
    </style>
  </head>
  <body>
    <script type="module">
      import * as THREE from 'https://threejs.org/build/three.module.js';
      import Stats from 'https://threejs.org/examples/jsm/libs/stats.module.js';
      import { OrbitControls } from 'https://threejs.org/examples/jsm/controls/OrbitControls.js';

      const stats = new Stats(); // 性能监控器，用来查看Three.js渲染帧率

      // 创建div
      const container = document.createElement('div');
      document.body.appendChild(container);

      // 创建场景
      const scene = new THREE.Scene();

      // 创建时钟
      const clock = new THREE.Clock();

      // 创建相机
      const camera = new THREE.PerspectiveCamera( // 透视投影相机
        40, // 视场，表示能够看到的角度范围
        window.innerWidth / window.innerHeight, // 渲染窗口的长宽比，设置为浏览器窗口的长宽比
        0.1, // 从距离相机多远的位置开始渲染
        2000 // 距离相机多远的位置截止渲染
      );
      camera.position.set(-20, 60, 30); // 设置相机位置

      // 创建渲染器
      const renderer = new THREE.WebGLRenderer({
        antialias: true, // 是否执行抗锯齿
      });
      renderer.setPixelRatio(window.devicePixelRatio); // 设置设备像素比率。通常用于HiDPI设备，以防止输出画布模糊。
      renderer.setSize(window.innerWidth, window.innerHeight); // 设置渲染器大小
      renderer.shadowMap.enabled = true;
      container.appendChild(renderer.domElement);

      // 创建控制器
      const controls = new OrbitControls(camera, renderer.domElement);

      // 创建平面
      const planeGeometry = new THREE.PlaneGeometry(300, 300); // 生成平面几何
      const planeMaterial = new THREE.MeshLambertMaterial({
        // 生成材质
        color: 0xcccccc,
      });
      const planeMesh = new THREE.Mesh(planeGeometry, planeMaterial); // 生成平面网格
      planeMesh.rotation.x = -Math.PI / 2; //绕X轴旋转90度
      scene.add(planeMesh); // 添加到场景中

      // 创建平行光源
      const light = new THREE.DirectionalLight(0xffffff, 1); // 平行光，颜色为白色，强度为1
      light.position.set(-40, 40, 20); // 设置灯源位置
      scene.add(light); // 添加到场景中

      // 创建门框
      const doorFrameSide = new THREE.BoxBufferGeometry(1, 20, 2); // 侧边门框
      const doorFrameTop = new THREE.BoxBufferGeometry(12, 1, 2); // 上门框
      const doorFrameMaterial = new THREE.MeshLambertMaterial({
        color: 0xad4800,
      });
      const doorFrameLeftMesh = new THREE.Mesh(
        doorFrameSide,
        doorFrameMaterial
      );
      const doorFrameTopMesh = new THREE.Mesh(doorFrameTop, doorFrameMaterial);
      const doorFrameRightMesh = doorFrameLeftMesh.clone();
      doorFrameLeftMesh.position.set(-5.5, 10, 0);
      doorFrameRightMesh.position.set(5.5, 10, 0);
      doorFrameTopMesh.position.set(0, 20.5, 0);
      scene.add(doorFrameLeftMesh);
      scene.add(doorFrameRightMesh);
      scene.add(doorFrameTopMesh);

      // 创建门
      const door = new THREE.BoxBufferGeometry(10, 20, 0.5);
      const doorMaterial = new THREE.MeshLambertMaterial({ color: 0xd88c00 });
      const doorMesh = new THREE.Mesh(door, doorMaterial);

      // 实现门围绕特定轴旋转
      const group = new THREE.Group();
      group.position.set(5, 10, 0); // 设置外层对象的中心为原本想要旋转的位置
      group.add(doorMesh);
      group.name = 'door';
      doorMesh.position.set(-5, 0, 0);
      scene.add(group);

      const times = [0, 3]; // 关键帧时间数组
      const rotationValues = [0, -Math.PI / 2];
      const rotationTrack = new THREE.KeyframeTrack(
        'door.rotation[y]',
        times,
        rotationValues
      ); // 关键帧轨道
      const duration = 3; // 持续时间 (单位秒)
      const clip = new THREE.AnimationClip('open', duration, [rotationTrack]); // 动画剪辑

      // 播放编辑好的关键帧数据
      const mixer = new THREE.AnimationMixer(group); // 动画混合器
      const AnimationAction = mixer.clipAction(clip); // 返回所传入的剪辑参数的AnimationAction
      AnimationAction.timeScale = 1; // 可以调节播放速度，默认是1。为0时动画暂停。值为负数时动画会反向执行。
      AnimationAction.play(); // 开始播放

      render();

      function render() {
        requestAnimationFrame(render);
        stats.begin();
        renderer.render(scene, camera);
        mixer.update(clock.getDelta());
        stats.end();
      }
    </script>
  </body>
</html>

```
