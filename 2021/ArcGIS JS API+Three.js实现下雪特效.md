# ArcGIS JS API+Three.js实现下雪特效
首先还是来看下效果图。
![arcgis三维场景下雪动态效果](https://travelclover.github.io/img/2021/04/三维场景下雪动态效果.gif)
通过观察图片知道，在三维场景中移动和旋转地图，雪花也会有所变化。这是因为本示例中的雪花效果确实是添加进场景中的，是三维空间中的一部分。而有些在三维地图中展示下雪效果的解决方案只是在地图的表面添加了一张GIF的动图，展现不出空间效果。  

本示例效果实现的原理是利用Three.js创建我们自定义的三维场景以及渲染器，在场景中添加雪花效果，雪花效果其实就是粒子效果，然后通过ArcGIS API中的`externalRenderers`接口注册到地图三维场景实例中，就能在地图三维场景中渲染出雪花了。

## 创建自定义外部渲染器类
渲染器实例化后至少需要两个方法：'`setup`'方法和'`render`'方法，具体的看查看ArcGIS API[文档](https://developers.arcgis.com/javascript/latest/api-reference/esri-views-3d-externalRenderers.html#ExternalRenderer)。下面是部分代码。
```javascript
class MyRenderer {
  view = null; // 三维地图视图实例
  renderer = null; // three.js 渲染器
  camera = null; // three.js 相机
  scene = null; // three.js 中的场景

  constructor(options) {
    this.view = options.view;
  }

  setup(context) {
    ...
  }
  render(context) {
    ...
  }
}

// 实例化渲染器
let myRenderer = new MyRenderer({ view: view });

// 注册renderer
externalRenderers.add(view, myRenderer);
```
在`setup`方法中需要利用Three.js创建三维场景以及渲染器，这个具体过程在以前的文章中有写过，详细的可查看我的文章[《ArcGIS API在视图中渲染Three.js场景》](https://blog.csdn.net/qq_37155408/article/details/105295877)了解。

## 创建粒子几何体
雪花是由许多个点组成的，一片雪花就是一个点，所以我们需要创建一个缓冲几何体，存储雪花的位置信息，一个顶点就是一个雪花位置。随机生成10000个点。
```javascript
const geometry = new THREE.BufferGeometry();
const vertices = [];
for (let i = 0; i < 10000; i++) {
  const x = Math.random() * 1000 - 500;
  const y = Math.random() * 1000 - 500;
  const z = Math.random() * 1000 - 500;
  vertices.push(x, y, z);
}
geometry.setAttribute(
  'position',
  new THREE.Float32BufferAttribute(vertices, 3)
);
```

## 加载雪花的纹理贴图
实现雪花效果，首先我们要加载雪花的纹理贴图。Three.js中提供的有纹理加载器`TextureLoader`。点击查看[文档](https://threejs.org/docs/index.html#api/zh/loaders/TextureLoader)。
```javascript
const textureLoader = new THREE.TextureLoader();
const sprite = textureLoader.load('./images/雪花纹理图片.png');
```

## 创建材质
因为要创建的物体是用来展示点，所以材质要选点材质`PointsMaterial`。点击查看[文档](https://threejs.org/docs/index.html#api/zh/materials/PointsMaterial)。
```javascript
// 创建点材质
const material = new THREE.PointsMaterial({
  size: size,
  map: sprite,
  color: new THREE.Color("rgb(255, 255, 255)"),
  blending: THREE.AdditiveBlending, // 融合方式
  depthTest: false, // 是否在渲染此材质时启用深度测试
  transparent: true, // 定义此材质是否透明
  opacity: 1,
});
```

## 创建物体
创建材质后就可以和几何体创建物体了。需要用到Three.js中的`Points`类，点击查看[文档](https://threejs.org/docs/index.html#api/zh/objects/Points)。
```javascript
const particles = new THREE.Points(geometry, material);
```
创建好物体后，需要设置它的位置，设置位置之前就需要将经纬度位置转换成渲染坐标系中的位置。下面是转换方法。
```javascript
/**
  * 经纬度坐标点转换为渲染坐标系中的点坐标
  * @param {number} longitude 经度
  * @param {number} latitude 纬度
  * @param {number} height 高度
  * @return {array} 返回渲染坐标系中的点坐标[x, y, z]
  */
pointTransform(longitude, latitude, height) {
  let transformation = new Array(16);
  // 将经纬度坐标转换为xy值
  let pointXY = webMercatorUtils.lngLatToXY(longitude, latitude);
  // 先转换高度为0的点
  externalRenderers.renderCoordinateTransformAt(
    this.view,
    [pointXY[0], pointXY[1], height], // 坐标在地面上的点[x值, y值, 高度值]
    this.view.spatialReference,
    transformation
  );
  return [transformation[12], transformation[13], transformation[14]];
}
```
设置位置：
```javascript
const location = pointTransform(经度, 纬度, 高度);
particles.position.set(...location);
```

## 添加动画
现在，就可以在地图三维场景中正确的位置上看到雪花效果了，但它是静止的。想要它动起来也很简单，只需在`render`方法中改变geometry中每个顶点的位置就行了，因为每一个顶点位置就代表一片雪花，改变顶点位置，雪花的位置也会随之改变。
![arcgis三维场景下雪动态效果](https://travelclover.github.io/img/2021/04/三维场景下雪动态效果.gif)

## 完整代码
如需查看示例效果可点击下载[完整代码](https://download.csdn.net/download/qq_37155408/16754328)。  

---
如果该文章对您有所帮助，请您一定不要吝啬您的鼓励。点赞、评论、分享、收藏、打赏都是您对我的鼓励和支持。  
如果您有`GitHub`账号，还可以[关注我~](https://github.com/travelclover)  
最后，感谢大家的阅读，如有错误，还请各位批评指正。
