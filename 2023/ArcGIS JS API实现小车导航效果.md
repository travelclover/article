
# ArcGIS JS API实现小车导航效果
随着智能交通技术的不断发展，汽车导航系统已经成为现代交通领域的重要组成部分，将导航路径与地图相结合，为用户提供准确的导航指引，极大地改善了交通效率和用户体验。在地理信息系统（GIS）领域中，ArcGIS JS API是一款常用的JavaScript库，用于构建交互式地图应用程序。本文将详细介绍如何利用ArcGIS JS API实现小车导航效果，让用户能够在地图上实时观察小车的空间位置变化。

# 效果展示
老规矩，废话不多说，先来看看实现的效果。小车按照预定轨迹移动。  
![小车导航效果](https://travelclover.github.io/img/2023/07/小车导航效果.gif)  

# 准备工作
在开始实现小车导航效果之前，我们需要进行一些准备工作：  

## 1.小车模型
我们需要准备一个小车模型，模型的格式必须是`glb`或者`gltf`。这里准备了一个Esri提供的模型，[点击下载](https://download.csdn.net/download/qq_37155408/88001543)。  

## 2.导航路径数据
数据可以是经纬度坐标点的集合，也可以是线要素的集合。可以通过GPS轨迹数据、地图绘制工具等方式获取。这篇文章中介绍了如何使用ArcGIS JS API获取鼠标点击的点坐标信息，[点击查看](https://blog.csdn.net/qq_37155408/article/details/124456096)。  

# 实现步骤
## 1.创建地图场景
首先需要引入ArcGIS JS API，包括样式文件和脚本文件。  
```html
<link
  rel="stylesheet"
  href="https://js.arcgis.com/4.27/esri/themes/light/main.css"
/>
<script src="https://js.arcgis.com/4.27/"></script>
```

引入以下需要用到的模块组件：
```javascript
require([
  'esri/Map',
  'esri/Basemap',
  'esri/views/SceneView',
  'esri/layers/TileLayer',
  'esri/layers/StreamLayer',
  'esri/geometry/Polyline',
  'esri/Graphic',
  'esri/geometry/geometryEngine',
], (
  Map, // 用于创建地图
  Basemap, // 用于创建底图
  SceneView, // 用于创建地图三维场景
  TileLayer, // 用于创建底图图层
  StreamLayer, // 用于创建表示小车运动的图层
  Polyline, // 用于创建几何线
  Graphic, // 用于创建路径图形，方便查看运动轨迹
  geometryEngine, // 几何引擎
) => {
  // 其它代码
  // ...
})
```

创建底图，这里采用了捷泰天域提供的一款深蓝色底图：
```javascript
const basemap = new Basemap({
  baseLayers: [
    new TileLayer({
      url: 'http://map.geoq.cn/arcgis/rest/services/ChinaOnlineStreetPurplishBlue/MapServer',
      title: 'Basemap',
    }),
  ],
});
```

创建地图对象：
```javascript
const map = new Map({
  basemap: basemap, // 底图
});
```

创建地图三维场景对象并设置相机位置：
```javascript
const view = new SceneView({
  container: 'viewDiv', // html容器
  map: map, // 实例化的地图对象
  camera: {
    position: [
      104.06779386534953, 30.61402157268845, 1400.646011153236, // 相机位置
    ],
    tilt: 0.5015795975933416, // 相机倾斜角度
  },
  spatialReference: {
    wkid: 3857, // 空间参考系
  },
});
```

利用准备阶段收集到的坐标信息，创建用于展示轨道路径的图形，并加载到地图场景中：
```javascript
const points = [
  [11584716.064248439, 3582737.033464487], // 点坐标信息，不包含高度值
  [11584863.752091048, 3582741.2897331216],
  [11584865.678214367, 3582862.4641863955],
  [11584851.156599585, 3583014.2159552197],
  [11584844.070043512, 3583118.873425072],
  [11584770.389997225, 3583160.3257233445],
  [11584705.427134208, 3583158.697613207],
  [11584716.064248439, 3582737.033464487],
];

const path = new Polyline({
  paths: [points], // 点坐标集合
  spatialReference: view.spatialReference, // 空间参考
});

const pathGraphic = new Graphic({
  geometry: path, // 几何体
  symbol: { // 展示符号
    type: 'simple-line',
    color: 'lightblue', // 颜色
    width: '2px', // 线的粗细
    style: 'solid', // 线的样式
  },
});
view.graphics.add(pathGraphic); // 添加进三维场景中
```

## 2.创建表现小车运动的图层
我们选用[StreamLayer](https://developers.arcgis.com/javascript/latest/api-reference/esri-layers-StreamLayer.html)类型图层。该图层是一种用于实时数据流的图层类型，可以从实时数据源接收数据，并动态地在地图上显示和更新数据。可以用于多种场景，如交通监控、天气预报、物联网数据可视化等。它可以连接到各种实时数据源，包括传感器数据、GPS轨迹、网络流量等，实时获取最新的数据并将其可视化在地图上。  
  
StreamLayer图层在`4.26`的版本更新中，新增了[sendMessageToClient](https://developers.arcgis.com/javascript/latest/api-reference/esri-layers-StreamLayer.html#sendMessageToClient)方法，该方法提供了在本地客户端就可以更新数据的能力，使得我们不再需要实时数据流服务就能更新图层数据。  
当然，使用该图层在本地客户端更新数据也有要求，在创建图层时必须设置以下属性：  
> 1.必须使用 `geometryType` 属性来指定要素的几何类型，因为每个图层只允许一种几何类型。  
> 2.必须要设置 `objectIdField` 属性来指定objectId字段，并且在字段列表中要有指定的字段值。  
> 3.必须在图层的 `timeInfo` 属性中设置 `trackIdField` 字段，以确保能够正确地跟踪和更新流数据的变化。  

创建StreamLayer：
```javascript
const streamLayer = new StreamLayer({
  objectIdField: 'OBJECTID', // required
  fields: [
    {
      name: 'OBJECTID', // required
      alias: 'ObjectId',
      type: 'oid',
    },
    {
      name: 'TRACKID',
      alias: 'TrackId',
      type: 'long',
    },
  ],
  timeInfo: {
    trackIdField: 'TRACKID', // required
  },
  geometryType: 'point', // required
  popupEnabled: false,
});
```

## 3.设置小车的模型
接下来需要设置图层renderer，将[SimpleRenderer](https://developers.arcgis.com/javascript/latest/api-reference/esri-renderers-SimpleRenderer.html)渲染器以及[ObjectSymbol3DLayer](https://developers.arcgis.com/javascript/latest/api-reference/esri-symbols-ObjectSymbol3DLayer.html)符号配合使用来展现小车的模型以及模型车头的朝向。  
[ObjectSymbol3DLayer](https://developers.arcgis.com/javascript/latest/api-reference/esri-symbols-ObjectSymbol3DLayer.html)是一种用于创建3D对象符号的图层，用于在三维场景中以对象为基础进行符号化，可以将自定义的3D模型或内置的3D符号应用于要素。  
```javascript
const symbol = {
  type: 'point-3d', // 符号类型
  symbolLayers: [
    {
      type: 'object', // ObjectSymbol3DLayer
      resource: {
        href: './Audi_A6.glb', // glb或者gltf模型地址
      },
    },
  ],
};
```
[SimpleRenderer](https://developers.arcgis.com/javascript/latest/api-reference/esri-renderers-SimpleRenderer.html)是ArcGIS JavaScript API中提供的一种渲染器类型，用于对图层中的要素进行简单的渲染，通过设置 [visualVariables](https://developers.arcgis.com/javascript/latest/api-reference/esri-renderers-SimpleRenderer.html#visualVariables) 视觉变量属性，可以实现根据每个要素的属性值来设置不同的符号颜色、大小、旋转、透明度等属性。  

```javascript
streamLayer.renderer = {
  type: 'simple', // 渲染器类型
  symbol: symbol, // 渲染的符号
  // 视觉变量
  visualVariables: [
    {
      type: 'size', // 修改大小的视觉变量
      axis: 'height', // height 高，width 宽, depth 深
      field: 'height', // 用于修改该视觉变量的字段值
      valueUnit: 'meters', // 单位米
    },
    {
      type: 'rotation', // 修改旋转的视觉变量
      axis: 'heading', // heading-符号在水平面上的旋转，tilt-符号在纵向垂直平面上的旋转，roll-符号在横向垂直平面上的旋转
      field: 'heading', // 用于修改该视觉变量的字段值
      rotationType: 'arithmetic', // geographic：从正北方向顺时针方向旋转  arithmetic：从正东方向逆时针旋转
    },
  ],
}
```
三轴（heading、tilt、roll）旋转的示意图如下所示：  
![旋转示意图](https://travelclover.github.io/img/2023/07/旋转示意图.png)

## 4.计算车头旋转的角度
想要正确的表示小车的前进方向，我们就需要计算出小车在水平面上的旋转角度。比如说在平面中，我们可以通过两个已知坐标的点，计算出这两个点的连线与正北方向或正东方向的夹角。所以我们可以通过小车当前位置坐标和上一时刻小车的位置坐标来计算小车车头朝向的角度。  
```javascript
// 计算线段的斜率
let slope = (p2.y - p1.y) / (p2.x - p1.x);
// 计算反正切值，得到弧度制的角度
let angleRad = Math.atan(slope);
// 将弧度制转换为角度制
let angleDeg = (angleRad * 180) / Math.PI;
// 如果线段在第二、三象限，则加上180度
if (p2.x < p1.x) {
  angleDeg += 180;
}
// 如果角度超出-180度到180度的范围，则减去360度或加上360度
if (angleDeg > 180) {
  angleDeg -= 360;
} else if (angleDeg < -180) {
  angleDeg += 360;
}
```

## 5.更新图层数据让小车动起来
前面已经提到过，我们可以使用`sendMessageToClient`方法来更新图层数据，在`attributes`属性对象中设置TRACKID、OBJECTID、height、heading。同一轨道ID表示同一实体，如果不一致，图层更新时就会认为是新的小车对象，就不会更新图层中原有的小车对象数据。OBJECTID每次更新时需要不一致。在renderer中设置了大小视觉变量和旋转视觉变量，分别与属性中的height、heading字段关联，所以更新时改变这两个值，模型的大小和朝向就会发生改变。
```javascript
const angle = getAngle(point1, point2); // 通过上一时刻点1和当前时刻点2的坐标计算车头的偏移角度

// 更新图层数据
streamLayer.sendMessageToClient({
  type: 'features', // 类型features表示更新数据，delete删除数据，clear清除所有数据
  features: [
    {
      attributes: {
        TRACKID: 1, // 指定的trackIdField字段，同一实体应保持该值一致
        OBJECTID: objectIdCounter++,
        height: 10, // 设置的模型高度视觉变量的属性值
        heading: angle, // 设置模型水平旋转视觉变量的属性值
      },
      geometry: {
        x: point2.x, // 更新到点2的位置
        y: point2.y,
      },
    },
  ],
});
```

## 6.实现跟随效果
简单的说，跟随效果就是在小车模型移动的同时，也更改相机的位置，使相机的空间位置与小车模型的空间位置保持相对静止。在小车转向的同时也需要更新相机在水平面上的旋转角度，该偏移角度与小车车头的旋转角度一致，所以更新相机位置时不需要再次计算旋转角度，只需要计算相机的空间坐标就行了。计算原理如下图所示：  
![跟随示意图](https://travelclover.github.io/img/2023/07/跟随示意图.png)  
我们可以从`view.camera`对象中读取到当前相机的tilt倾斜角度以及相机的高度，根据三角函数可以计算出相机在地面上的投影点C到点A的距离，又知道点A和点B的坐标，可以计算出它们之间的距离，再通过直线上点B到点A的距离与点A到点C距离的比值关系可以计算出点C的坐标。更新相机位置时，只需要更新水平面旋转角度heading和平面坐标位置就行了。 
计算点C坐标代码如下： 
```javascript
const height = view.camera.position.z; // 相机高度
const tilt = view.camera.tilt; // 相机倾斜角度
const distance = Math.tan((tilt / 180) * Math.PI) * height; // 距离
const d = Math.sqrt(
  (p1.x - p2.x) * (p1.x - p2.x) + (p1.y - p2.y) * (p1.y - p2.y)
);
const ratio = d / (d + distance);
const x = (p1.x - p2.x) / ratio + p2.x;
const y = (p1.y - p2.y) / ratio + p2.y;
const camera = view.camera.clone();
camera.position.x = x;
camera.position.y = y;
camera.heading = -(angle + 90) + 180; // heading为正北方向逆时针旋转，所以需要从正东方向的角度转成从正北方向逆时针旋转的角度
view.goTo(camera);
```

最后通过`window.requestAnimationFrame`方法，不停的更新小车位置和相机位置，这样跟随效果就出来了。  
![导航跟随效果](https://travelclover.github.io/img/2023/07/导航跟随效果.gif)

# 总结
本文详细介绍了如何利用ArcGIS JS API实现小车导航效果。借助ArcGIS JS API的强大功能和灵活性，构建出一个小车实时位置更新可视化效果页面。本示例最主要使用了ArcGIS JavaScript API中的`SimpleRenderer`渲染器、`ObjectSymbol3DLayer`符号图层以及`StreamLayer`图层等API。

# 完整代码
如需查看示例效果可点击下载[ArcGIS JS API实现小车导航效果（源码+详细注释）.zip](https://mbd.pub/o/bread/ZJuUlJxu)。代码中包含详细的注释，方便大家的理解。  

---
如果该文章对您有所帮助，请您一定不要吝啬您的鼓励。点赞、评论、分享、收藏、打赏都是您对我的鼓励和支持。  
如果您有`GitHub`账号，还可以[关注我~](https://github.com/travelclover)  
最后，感谢大家的阅读，如有错误，还请各位批评指正。
