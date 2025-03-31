# 在ArcGIS JS API中使用Three.js加载动画模型

在 WebGIS 领域，3D 可视化已成为提升用户体验的重要手段。ArcGIS API for JavaScript 提供了强大的 3D 场景支持，但其内置 3D 模型加载能力有限，比如加载的gltf 模型会丢失动画。因此，我们可以结合 Three.js 来扩展其 3D 可视化能力。本篇文章介绍如何在 ArcGIS JS API 中使用 Three.js 加载动画模型。  
![波纹扩散效果](https://travelclover.github.io/img/2025/03/model.gif)  

# 1 实现原理
本方案的核心是通过ArcGIS JS API的[`RenderNode`](https://developers.arcgis.com/javascript/latest/api-reference/esri-views-3d-webgl-RenderNode.html)机制，在 WebGL 渲染管线中插入自定义的Three.js渲染逻辑。具体流程如下：
1. 创建自定义的RenderNode子类
2. 在渲染节点中初始化Three.js环境
3. 加载GLTF格式的3D模型
4. 处理模型动画
5. 同步ArcGIS视图的相机参数到Three.js相机

# 2 实现步骤
## 2.1 创建自定义的RenderNode子类
首先，我们需要创建一个自定义的RenderNode子类，用于在WebGL渲染管线中插入Three.js的渲染逻辑。如以下代码所示：
```javascript
const ThreeJSRenderNode = RenderNode.createSubclass({
  constructor: function ({ view }) {
    this.consumes = { required: ['composite-color'] };
    this.produces = 'composite-color';

    const coord = [point.x, point.y, point.z]; // 点坐标
    this.transAt = glMatrix.mat4.create();
    webgl.renderCoordinateTransformAt(
      view,
      coord,
      view.spatialReference,
      this.transAt
    );
  },

  render(inputs) {
    // ...

    return inputs[0];
  },
});
```
在constructor构造函数中，需要定义渲染节点(RenderNode)的输入输出关系：
1. **this.consumes**：
   - 指定该渲染节点需要哪些输入数据。
   - `required: ['composite-color']` 表示必须要有'composite-color'输入。
   - `'composite-color'`是ArcGIS场景中其他渲染节点输出的合成颜色数据。
2. **this.produces**：
   - 声明该节点会输出`'composite-color'`数据。
   - 这样其他渲染节点就可以使用本节点的输出作为它们的输入。
  
在ThreeJSRenderNode这个案例中，这种设置实现了：
1. 将ArcGIS原有的场景渲染结果(composite-color)作为输入
2. 在Three.js渲染完成后，将结果合并回ArcGIS的渲染管线
3. 最终输出包含Three.js模型的新合成场景(composite-color)

这种机制允许Three.js渲染与ArcGIS原生渲染无缝集成。  

**`webgl.renderCoordinateTransformAt()`** 方法的作用是计算一个变换矩阵，后续在加载模型后，将模型运用该变换矩阵，确保Three.js中的模型能正确显示在ArcGIS场景中的指定位置。

**`render() {}`** render函数是渲染节点的核心逻辑，整个方法在ArcGIS的每帧渲染流程中被自动调用，返回值为输入参数 inputs 数组的第一个元素，在这里原样返回输入表示不修改ArcGIS原有的渲染结果，只是在其基础上叠加Three.js的渲染内容。  

## 2.2 保存、设置以及恢复webgl状态
在Three.js渲染之前，需要保存当前的WebGL状态，因为ArcGIS JS API有自己的渲染管线，所以需要先保存它的状态，以便在渲染完成后恢复。如以下代码所示：
```javascript
saveGLState(gl) {
  return {
    vbo: gl.getParameter(gl.VERTEX_ARRAY_BINDING), // 获取当前绑定的顶点数组对象
    blend: gl.getParameter(gl.BLEND), // 获取是否启用混合的状态
    depthMask: gl.getParameter(gl.DEPTH_WRITEMASK), // 获取当前 depthMask 状态
    srcRGB: gl.getParameter(gl.BLEND_SRC_RGB), // 获取当前的源 RGB 混合因子
    dstRGB: gl.getParameter(gl.BLEND_DST_RGB), // 获取当前的目标 RGB 混合因子
    stencil: gl.getParameter(gl.STENCIL_TEST), // 获取当前的模板测试状态，返回true 或 false
    depthTest: gl.getParameter(gl.DEPTH_TEST), // 获取深度测试参数
  };
}
```
为了确保Three.js能正确渲染，所以在保存状态后，需要将WebGL的状态设置为Three.js需要的WebGL渲染状态：
```javascript
setupGLState(gl) {
  gl.depthMask(true); // 启用深度写入
  gl.enable(gl.STENCIL_TEST); // 开启模板测试
  gl.bindVertexArray(null); // 清除当前的 顶点数组对象（VAO） 绑定
  gl.enable(gl.BLEND); // 启用混合状态
  gl.blendFunc(gl.SRC_ALPHA, gl.ONE_MINUS_SRC_ALPHA); // 设置混合因子
  gl.enable(gl.DEPTH_TEST); // 启用深度测试
}
```
在每帧Three.js场景渲染完成后，需要恢复WebGL状态：
```javascript
restoreGLState(gl, state) {
  state.blend ? gl.enable(gl.BLEND) : gl.disable(gl.BLEND);
  gl.depthMask(state.depthMask);
  gl.blendFunc(state.srcRGB, state.dstRGB);
  gl.bindVertexArray(state.vbo);
  if (state.stencil) gl.enable(gl.STENCIL_TEST);
  if (state.depthTest) gl.enable(gl.DEPTH_TEST);
}
```

## 2.3 初始化Three.js场景
在Three.js渲染之前，需要初始化Three.js场景。主要包含以下内容：
1. 创建Three.js渲染器
2. 创建Three.js场景和相机
3. 在Three.js场景中添加灯光灯光
4. 加载模型
  
同时需要检查是否已经初始化过Three.js渲染器，如果已经初始化则直接返回，避免重复初始化。
```javascript
initThree() {
  if (this.threeRenderer) return;
  this.setupRenderer();
  this.setupScene();
  this.setupLights();
  this.loadModel();
}
```

### 2.3.1 创建Three.js渲染器
使用ArcGIS提供的WebGL上下文创建Three.js渲染器，并设置相应状态：
```javascript
setupRenderer() {
  const renderer = new THREE.WebGLRenderer({
    canvas: this.view.canvas,
    context: this.gl,
    alpha: true,
  });
  renderer.autoClearDepth = false; // 禁止自动清除深度缓冲区
  renderer.autoClearColor = false; // 可选，禁止清除颜色缓冲区
  renderer.autoClearStencil = false; // 可选，禁止清除模板缓冲区
  renderer.setPixelRatio(window.devicePixelRatio);
  this.threeRenderer = renderer;
}
```
需要注意的是，这里创建渲染器需要使用 ArcGIS JS API 的视图 canvas 和 WebGL 上下文。

### 2.3.2 创建Three.js场景和相机
Three.js的场景和相机需要使用ArcGIS的相机参数进行初始化，并设置相应的状态：
```javascript
setupScene() {
  this.threeScene = new THREE.Scene();
  this.threeCamera = new THREE.PerspectiveCamera(
    this.view.camera.fov,
    this.camera.width / this.camera.height,
    0.1,
    1000
  );
}
```
这里使用了ArcGIS的相机参数来创建了Three.js的透视相机。

### 2.3.3 在Three.js场景中添加灯光
Three.js的场景需要添加灯光，否则模型将无法显示：
```javascript
setupLights() {
  const { ambient, diffuse, direction } = this.sunLight;
  const ambientColor = new THREE.Color().setRGB(...ambient.color);
  const ambientLight = new THREE.AmbientLight(
    ambientColor,
    ambient.intensity
  );
  this.threeScene.add(ambientLight);

  const diffuseColor = new THREE.Color().setRGB(...diffuse.color);
  const directionalLight = new THREE.DirectionalLight(
    diffuseColor,
    diffuse.intensity * 10
  );
  directionalLight.position.set(
    direction[0],
    direction[1],
    direction[2]
  );
  this.threeScene.add(directionalLight);
}
```
这里添加了一个环境光和一个方向光来模拟太阳的光照效果。  

### 2.3.4 加载模型
使用Three.js加载模型非常简单，只需使用模型相对应的加载器即可，在这里我们使用Three.js提供的GLTFLoader加载器来加载GLTF格式的3D模型：
```javascript
loadModel() {
  const loader = new GLTFLoader();
  loader.load('./Cesium_Air.glb', (gltf) => this.onModelLoaded(gltf));
}
```

### 2.3.5 设置模型姿态
加载模型后，需要设置模型的姿态，使其在场景中正确显示。这里我们使用了ArcGIS提供的webgl.renderCoordinateTransformAt方法计算从局部坐标到世界坐标的变换矩阵，然后将模型应用该变换矩阵，确保模型能正确显示在ArcGIS场景中的指定位置。
```javascript
onModelLoaded(gltf) {
  gltf.scene.scale.set(2, 2, 2); // 缩放模型

  const mat4 = new THREE.Matrix4().fromArray(this.transAt);
  gltf.scene.applyMatrix4(mat4);

  this.threeScene.add(gltf.scene);
}
```
![模型姿态](https://travelclover.github.io/img/2025/03/模型姿态.jpg)
模型添加到场景中后，会发现位置对了，但是姿态是错的，需要将模型的姿态进行调整：
```javascript
gltf.scene.rotateX((Math.PI / 180) * 90); // 使模型平行于地面
gltf.scene.rotateY((Math.PI / 180) * -22); // 使飞机模型对齐跑到
gltf.scene.rotateX((Math.PI / 180) * -10); // 使飞机模型向上仰10度，模拟正在起飞的姿态
```
现在模型就调整到正确的姿态了。
![模型姿态调整](https://travelclover.github.io/img/2025/03/模型姿态调整.jpg)

### 2.3.6 处理模型动画
加载模型后，需要处理模型的动画，使其能够正常播放。这里我们使用了Three.js提供的AnimationMixer来处理模型的动画：
```javascript
setupAnimations(gltf) {
  this.mixer = new THREE.AnimationMixer(gltf.scene);
  const animations = gltf.animations;
  if (animations?.length > 0) {
    animations.forEach((clip) => this.mixer.clipAction(clip).play());
  }
}
```
在每帧渲染中，都需要更新模型的动画：
```javascript
const now = performance.now();
if (!this.lastTime) this.lastTime = now;
const deltaTime = now - this.lastTime; // 计算时间差
this.lastTime = now;

if (this.mixer) {
  this.mixer.update(deltaTime / 1000);
}
```

### 2.3.7 渲染模型
在每帧渲染中，都需要将Three.js场景渲染到ArcGIS的WebGL上下文中，在渲染前，需要同步Three.js相机与ArcGIS视图相机：
```javascript
// 设置相机位置
this.threeCamera.position.set(
  this.camera.eye[0],
  this.camera.eye[1],
  this.camera.eye[2]
);
this.threeCamera.up = new THREE.Vector3(
  this.camera.up[0],
  this.camera.up[1],
  this.camera.up[2]
);
this.threeCamera.lookAt(
  this.camera.center[0],
  this.camera.center[1],
  this.camera.center[2]
);

// 投影矩阵同步
this.threeCamera.projectionMatrix.fromArray(
  this.camera.projectionMatrix
);
```
然后渲染Three.js场景：
```javascript
this.threeRenderer.render(this.threeScene, this.threeCamera);
```
渲染后需要重置渲染器状态，避免影响后续ArcGIS的渲染：
```javascript
this.threeRenderer.resetState();
```

### 2.3.8 连续渲染
为了确保动画的连续性，需要在每一帧渲染中都调用 **`requestRender()`** 方法，来请求重新渲染。需要在render函数中添加以下代码：
```javascript
this.requestRender();
```
最终实现模型加载和动画播放的效果。
![波纹扩散效果](https://travelclover.github.io/img/2025/03/model.gif)  

# 3 技术要点
1. **共享WebGL上下文**：Three.js使用ArcGIS已有的WebGL上下文，避免创建额外的canvas元素。
2. **状态管理**：在Three.js渲染前后保存和恢复WebGL状态，确保不影响ArcGIS的正常渲染。
3. **坐标转换**：使用ArcGIS提供的webgl.renderCoordinateTransformAt方法计算从局部坐标到世界坐标的变换矩阵。
4. **相机状态同步**：在每帧渲染中，需要将ArcGIS的相机参数同步到Three.js的相机中。

# 4 总结
通过ArcGIS JS API的RenderNode机制集成Three.js，我们成功在地理场景中加载并动画化3D模型。这种方案既利用了ArcGIS强大的地理可视化能力，又发挥了Three.js在3D模型渲染方面的优势。这种方案不仅可以实现复杂的3D模型渲染，还可以与ArcGIS的其他功能无缝集成，为用户提供更加丰富的地理信息展示体验。  

# 5 源码下载
如需源码可点击[下载链接](https://mbd.pub/o/bread/aJablZZy)，非常感谢您的支持。  
通过关注微信公众号《Web与GIS》，回复关键字`XKJYQPLM`。

![微信公众号](https://travelclover.github.io/img/2025/02/微信公众号.png)

---
如果该文章对您有所帮助，请您一定不要吝啬您的鼓励。点赞、评论、分享、收藏、打赏都是您对我的鼓励和支持。  
如果您有`GitHub`账号，还可以[关注我~](https://github.com/travelclover)  
最后，感谢大家的阅读，如有错误，还请各位批评指正。
