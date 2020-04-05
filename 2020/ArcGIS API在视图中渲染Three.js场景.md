# ArcGIS API在视图中渲染Three.js场景
ArcGIS API中的`SceneView`使用`WebGL`在屏幕上渲染地图和场景，还提供了一个底层接口来访问SceneView的WebGL上下文，因此可以创建与场景交互的自定义​​可视化效果，方式与内置图层相同。那么我们可以直接编写WebGL代码，也可以集成第三方WebGL库（例如Three.js）。  
现在我门就来尝试在ArcGIS的三维场景中加入一个物体，比如说`UFO`：  
![UFO模型](https://github.com/travelclover/img/blob/master/2020/04/UFO.jpg)
> 该模型下载自[CG模型网](https://www.cgmodel.com/model-918.html)。然后通过`3DS MAX`软件导出为obj格式模型。

## 1.引入需要用到的类  
引入以下所要用到的类，并创建一个地图三维场景：
```javascript
require([
    'esri/Map', // 生成地图的类
    'esri/views/SceneView', // 生成三维场景的类
    'esri/views/3d/externalRenderers', // 外部渲染器对象
    'esri/geometry/SpatialReference', // 空间参考的类
    ], function(Map, SceneView, externalRenderers, SpatialReference) {
    const map = new Map({
        basemap: 'topo-vector',
    });

    const view = new SceneView({
        container: 'viewDiv', // 包含视图的容器
        map: map,
        center: [105, 29],
        zoom: 3,
    });
});
```

## 2.定义外部渲染器对象
我们需要使用回调方法和属性来定义一个外部渲染器，在向SceneView注册时需要用到。
```javascript
const myRenderer = {
    renderer: null, // three.js 渲染器
    camera: null, // three.js 相机
    scene: null, // three.js 中的场景
    ambient: null, // three.js中的环境光
    sun: null, // three.js中的平行光源，模拟太阳光
    ufo: null, // ufo

    setup: function(context) {},
    render: function(context) {},
    dispose: function(context) {},
};
```
其中`setup`、`render`和`dispose`为渲染器回调。  
> **`setup`** 函数通常在将外部渲染器添加到视图后调用一次，或者每当SceneView准备就绪时调用一次。如果就绪状态循环（例如，当将不同的Map分配给视图时），则可以再次调用它。接收一个类型为[*RenderContext*](https://developers.arcgis.com/javascript/latest/api-reference/esri-views-3d-externalRenderers.html#RenderContext)的参数。  
> **`render`** 函数在每一帧中调用以执行状态更新和绘制。接收一个类型为[*RenderContext*](https://developers.arcgis.com/javascript/latest/api-reference/esri-views-3d-externalRenderers.html#RenderContext)的参数。  
> **`dispose`** 函数在从视图中移除外部渲染器时，或者视图的就绪状态变为false时调用。接收一个类型为[*RenderContext*](https://developers.arcgis.com/javascript/latest/api-reference/esri-views-3d-externalRenderers.html#RenderContext)的参数。  

### 2.1完善setup回调函数
我们需要在该回调函数中定义Three.js的渲染器、场景、摄像机和光源，还要导入需要加载到场景中的UFO模型。  

#### ①定义Three.js的渲染器
```javascript
this.renderer = new THREE.WebGLRenderer({
    context: context.gl, // 可用于将渲染器附加到已有的渲染环境(RenderingContext)中
    premultipliedAlpha: false, // renderer是否假设颜色有 premultiplied alpha. 默认为true
});
```
设置设备像素比，可以避免HiDPI设备上绘图模糊：
```javascript
this.renderer.setPixelRatio(window.devicePixelRatio);
```
设置视口大小和三维场景的大小一样：
```javascript
this.renderer.setViewport(0, 0, view.width, view.height);
```
为了防止Three.js清除ArcGIS JS API提供的缓冲区，需要添加以下代码：
```javascript
this.renderer.autoClearDepth = false; // 定义renderer是否清除深度缓存
this.renderer.autoClearStencil = false; // 定义renderer是否清除模板缓存
this.renderer.autoClearColor = false; // 定义renderer是否清除颜色缓存
```
ArcGIS JS API渲染自定义离屏缓冲区，而不是默认的帧缓冲区。我们必须将这段代码注入到Three.js运行时中，以便绑定这些缓冲区而不是默认的缓冲区。
```javascript
const originalSetRenderTarget = this.renderer.setRenderTarget.bind(
    this.renderer
);
this.renderer.setRenderTarget = function(target) {
    originalSetRenderTarget(target);
    if (target == null) {
        context.bindRenderTarget();
    }
};
```

#### ②定义场景和相机
```javascript
this.scene = new THREE.Scene(); // 场景
this.camera = new THREE.PerspectiveCamera(); // 相机
```

#### ③定义光源并添加到场景中
```javascript
this.ambient = new THREE.AmbientLight(0xffffff, 0.5); // 环境光
this.scene.add(this.ambient); // 把环境光添加到场景中
this.sun = new THREE.DirectionalLight(0xffffff, 0.5); // 平行光（模拟太阳光）
this.scene.add(this.sun); // 把太阳光添加到场景中
```

#### ④添加辅助工具
为了更好的理解空间位置，可以添加坐标轴辅助工具：
```javascript
const axesHelper = new THREE.AxesHelper(10000000);
this.scene.add(axesHelper);
```

#### ⑤加载OBJ模型
加载模型之前先要加载模型的材质信息文件，也就是`.mtl`格式的文件，需要用到`MTLLoader`加载器。加载obj模型则需要用到`OBJLoader`加载器。它们都可以在全球最大同性交友网站([GitHub](https://github.com/))的[`three.js`](https://github.com/mrdoob/three.js)代码仓库下找到。
```javascript
let mtlLoader = new MTLLoader();
mtlLoader.setPath('../assets/model/');
mtlLoader.load('ufo.mtl', materials => {
    materials.preload();
    // OBJLoader
    const loader = new OBJLoader();
    loader.setMaterials(materials);
    loader.setPath('../assets/model/');
    loader.load(
        'ufo.obj', // 资源地址
        // 加载成功后的回调
        object => {
            // ...
        },
        // 加载过程中的回调
        function(xhr) {
            // console.log((xhr.loaded / xhr.total) * 100 + '% loaded');
        },
        // 加载模型出错的回调
        function(error) {
            console.error('An error happened: ', error);
        }
    );
});
```
加载成功的回调方法接收一个参数，该参数就是[Object3D](https://threejs.org/docs/index.html#api/en/core/Object3D)对象，也就是我们要加载的3D模型对象。在该回调中，我们可以进行模型的位置调整，以及大小调整等设置，然后添加到场景中。
```javascript
this.ufo = object;
const entryPos = [70, 0, 550000]; // 输入位置 [经度, 纬度, 高程]
const renderPos = [0, 0, 0]; // 渲染位置
externalRenderers.toRenderCoordinates(
    view,
    entryPos,
    0,
    SpatialReference.WGS84,
    renderPos,
    0,
    1
);
this.ufo.scale.set(100000, 100000, 100000); // UFO放大一点
this.ufo.position.set( // 设置UFO位置
    renderPos[0],
    renderPos[1],
    renderPos[2]
);
this.scene.add(this.ufo); // 添加到场景中
```
> `externalRenderers`对象的[`toRenderCoordinates`](https://developers.arcgis.com/javascript/latest/api-reference/esri-views-3d-externalRenderers.html#toRenderCoordinates)方法是将位置从给定的空间参考转换为内部渲染坐标系，共接收7个参数(view, srcCoordinates, srcStart, srcSpatialReference, destCoordinates, destStart, count )。  
> **view**: 地图场景。该参数类型为*SceneView*  
> **srcCoordinates**: 一个或多个向量坐标组成的一维数组，例如[x1, y1, z1, x2, y2, z2]，数组中元素数量必须是3的倍数。该参数类型为*Array*  
> **srcStart**: srcCoordinates中的索引，从该索引开始读取坐标。该参数类型为*Number*  
> **srcSpatialReference**: 输入坐标的空间参考。如果为null，则用view.spatialReference替代。该参数类型为*SpatialReference*  
> **destCoordinates**: 对要写入结果的数组的引用。该参数类型为*Array*  
> **destStart**: destCoordinates中的索引，坐标将从索引处开始写入。该参数类型为*Number*  
> **count**: 要转换的坐标数量。该参数类型为*Number*

### 2.2完善render回调函数
在每一帧中都会调用该回调函数，接收一个类型为`RenderContext`的参数。在该回调中我们可以进行相机参数更新，模型位置更新等操作。
```javascript
// 更新相机参数
const cam = context.camera;
this.camera.position.set(cam.eye[0], cam.eye[1], cam.eye[2]);
this.camera.up.set(cam.up[0], cam.up[1], cam.up[2]);
this.camera.lookAt(
    new THREE.Vector3(cam.center[0], cam.center[1], cam.center[2])
);
// 投影矩阵可以直接复制
this.camera.projectionMatrix.fromArray(cam.projectionMatrix);

// 更新UFO
this.ufo.rotation.y += 0.1;

// 绘制场景
this.renderer.state.reset();
this.renderer.render(this.scene, this.camera);

externalRenderers.requestRender(view); // 请求重绘视图。

// 清除WebGL状态
context.resetWebGLState();
```

## 添加外部渲染器
最后还有一个关键步骤，向`SceneView`实例注册外部渲染器:
```javascript
externalRenderers.add(view, myRenderer);
```
这样我们就成功地在地图三维场景中渲染出用Three.js加载的外部模型UFO啦！  
![地图中加载UFO模型](https://github.com/travelclover/img/blob/master/2020/04/externalRenderers.jpg)

***
**以下是完整代码**
```html
<html>
  <head>
    <meta charset="utf-8" />
    <meta
      name="viewport"
      content="initial-scale=1, maximum-scale=1, user-scalable=no"
    />
    <title>ArcGIS API在视图中渲染Three.js场景</title>
    <style>
      html,
      body,
      #viewDiv {
        padding: 0;
        margin: 0;
        height: 100%;
        width: 100%;
      }
    </style>

    <link
      rel="stylesheet"
      href="https://js.arcgis.com/4.14/esri/css/main.css"
    />
    <script src="https://js.arcgis.com/4.14/"></script>

    <script type="module">
      import * as THREE from 'https://threejs.org/build/three.module.js';
      import { OBJLoader } from 'https://threejs.org/examples/jsm/loaders/OBJLoader.js';
      import { MTLLoader } from 'https://threejs.org/examples/jsm/loaders/MTLLoader.js';

      require([
        'esri/Map',
        'esri/views/SceneView',
        'esri/views/3d/externalRenderers',
        'esri/geometry/SpatialReference',
      ], function(Map, SceneView, externalRenderers, SpatialReference) {
        const map = new Map({
          basemap: 'topo-vector',
        });

        const view = new SceneView({
          container: 'viewDiv',
          map: map,
          center: [105, 29],
          zoom: 3,
        });

        const myRenderer = {
          renderer: null, // three.js 渲染器
          camera: null, // three.js 相机
          scene: null, // three.js 中的场景
          ambient: null, // three.js中的环境光
          sun: null, // three.js中的平行光源，模拟太阳光
          ufo: null, // ufo

          setup: function(context) {
            this.renderer = new THREE.WebGLRenderer({
              context: context.gl, // 可用于将渲染器附加到已有的渲染环境(RenderingContext)中
              premultipliedAlpha: false, // renderer是否假设颜色有 premultiplied alpha. 默认为true
            });
            this.renderer.setPixelRatio(window.devicePixelRatio); // 设置设备像素比。通常用于避免HiDPI设备上绘图模糊
            this.renderer.setViewport(0, 0, view.width, view.height); // 视口大小设置

            // 防止Three.js清除ArcGIS JS API提供的缓冲区。
            this.renderer.autoClearDepth = false; // 定义renderer是否清除深度缓存
            this.renderer.autoClearStencil = false; // 定义renderer是否清除模板缓存
            this.renderer.autoClearColor = false; // 定义renderer是否清除颜色缓存

            // ArcGIS JS API渲染自定义离屏缓冲区，而不是默认的帧缓冲区。
            // 我们必须将这段代码注入到three.js运行时中，以便绑定这些缓冲区而不是默认的缓冲区。
            const originalSetRenderTarget = this.renderer.setRenderTarget.bind(
              this.renderer
            );
            this.renderer.setRenderTarget = function(target) {
              originalSetRenderTarget(target);
              if (target == null) {
                // 绑定外部渲染器应该渲染到的颜色和深度缓冲区
                context.bindRenderTarget();
              }
            };

            this.scene = new THREE.Scene(); // 场景
            this.camera = new THREE.PerspectiveCamera(); // 相机

            this.ambient = new THREE.AmbientLight(0xffffff, 0.5); // 环境光
            this.scene.add(this.ambient); // 把环境光添加到场景中
            this.sun = new THREE.DirectionalLight(0xffffff, 0.5); // 平行光（模拟太阳光）
            this.scene.add(this.sun); // 把太阳光添加到场景中

            // 添加坐标轴辅助工具
            const axesHelper = new THREE.AxesHelper(10000000);
            this.scene.add(axesHelper);

            // 加载模型
            let mtlLoader = new MTLLoader();
            mtlLoader.setPath('../assets/model/');
            mtlLoader.load('ufo.mtl', materials => {
              materials.preload();
              // OBJLoader
              const loader = new OBJLoader();
              loader.setMaterials(materials);
              loader.setPath('../assets/model/');
              loader.load(
                'ufo.obj', // 资源地址
                // 加载成功后的回调
                object => {
                  this.ufo = object;
                  const entryPos = [70, 0, 550000]; // 输入位置
                  const renderPos = [0, 0, 0]; // 渲染位置
                  externalRenderers.toRenderCoordinates(
                    view,
                    entryPos,
                    0,
                    SpatialReference.WGS84,
                    renderPos,
                    0,
                    1
                  );
                  this.ufo.scale.set(100000, 100000, 100000); // ufo放大一点
                  this.ufo.position.set(
                    renderPos[0],
                    renderPos[1],
                    renderPos[2]
                  );
                  this.scene.add(this.ufo);
                  context.resetWebGLState();
                },
                // 加载过程中的回调
                function(xhr) {
                  // console.log((xhr.loaded / xhr.total) * 100 + '% loaded');
                },
                // 加载模型出错的回调
                function(error) {
                  console.error('An error happened: ', error);
                }
              );
            });
          },
          render: function(context) {
            // 更新相机参数
            const cam = context.camera;
            this.camera.position.set(cam.eye[0], cam.eye[1], cam.eye[2]);
            this.camera.up.set(cam.up[0], cam.up[1], cam.up[2]);
            this.camera.lookAt(
              new THREE.Vector3(cam.center[0], cam.center[1], cam.center[2])
            );
            // 投影矩阵可以直接复制
            this.camera.projectionMatrix.fromArray(cam.projectionMatrix);
            // 更新UFO
            this.ufo.rotation.y += 0.1;
            // 绘制场景
            this.renderer.state.reset();
            this.renderer.render(this.scene, this.camera);
            // 请求重绘视图。
            externalRenderers.requestRender(view); 
            // cleanup
            context.resetWebGLState();
          },
        };
        // 注册renderer
        externalRenderers.add(view, myRenderer);
      });
    </script>
  </head>
  <body>
    <div id="viewDiv"></div>
  </body>
</html>
```
