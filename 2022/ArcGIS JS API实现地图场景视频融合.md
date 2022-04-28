
# 效果展示
老规矩，还是先来看下在地图场景中实现视频融合的效果。![地图场景视频融合效果](https://travelclover.github.io/img/2022/04/地图场景视频融合效果.gif)  

效果展示了在地图场景中播放视频画面的效果，这个效果用ArcGIS API for JavaScript中提供的`externalRenderers`对象再配合`Three.js`库也能实现，但实现过程相对来说比较复杂，今天我们就来使用一种简单的方法来实现在三维地图场景中展示视频画面的功能，整个代码只会用到ArcGIS的API，不会用到任何第三方库。
# 实现步骤
## 1.创建地图场景
首先肯定是要先引入ArcGIS的API，这里我们就使用标签的形式来引用所要用到的文件。  
```html
<link
  rel="stylesheet"
  href="https://js.arcgis.com/4.23/esri/themes/light/main.css"
/>
<script src="https://js.arcgis.com/4.23/"></script>
```
引入了4.23版本的JS脚本以及样式文件。  
## 2.引入相应模块并创建地图场景
需要用`require`从API中引入使用到的模块，如下所示：
```javascript
require([
  'esri/Map',
  'esri/views/SceneView',
  'esri/geometry/Mesh',
  'esri/geometry/support/MeshComponent',
  'esri/Graphic',
], (Map, SceneView, Mesh, MeshComponent, Graphic) => {
 // ...
})
```
创建地图和视图场景：
```html
<body>
  <div id="viewDiv"></div>
</body>
```
```javascript
// 创建地图
const map = new Map({
  basemap: 'topo-vector',
  ground: 'world-elevation',
});
// 创建视图场景
const view = new SceneView({
  container: 'viewDiv',
  map: map,
});
```
## 3.获取点坐标
我们需要获取如图所示的几个点的坐标信息：
![点示意图](https://travelclover.github.io/img/2022/04/点示意图.png)
三维地图场景中，由6个点可以构成2个平面，其中[P1, P2, P3, P4]构成的平面平行于地面，[P3, P4, P5, P6]构成的平面垂直于地面。P5、P6这2个点的坐标信息可以由P4、P3坐标点各自在Z轴加上一个高度得来。所以我们至少需要知道P1、P2、P3和P4这4个点的坐标信息。  
为了方便得到这几个点的坐标信息，我们可以给三维场景绑定一个点击事件，每次点击时打印出事件信息。
```javascript
// 地图场景点击事件
view.on('click', (e) => {
  console.log(e);
});
```
绑定事件后，每次在地图上点击鼠标，都会在调试控制台里打印出这次事件的具体信息，其中就包括点击时的地图坐标点信息，如图所示：
![点坐标信息](https://travelclover.github.io/img/2022/04/点坐标信息.png)
通过以上操作生成六个点坐标信息：
```javascript
const height = 120; // 用于P5、P6两个点Z值向上偏移高度
const heightOffset = 1; // 所有点的Z值都向上偏移1个单位，避免部分画面可能低于地面导致看不见
const point1 = [
  11584169.159857891,
  3588572.695132413,
  496.27007252703896 + heightOffset,
];
const point2 = [
  11584396.703916386,
  3588574.979544681,
  495.8151520746715 + heightOffset,
];
const point3 = [
  11584394.889440097,
  3588724.2562674284,
  496.4738873814049 + heightOffset,
];
const point4 = [
  11584163.8079081,
  3588718.3809192777,
  496.2824081689934 + heightOffset,
];
const point5 = [
  11584163.8079081,
  3588718.3809192777,
  496.2824081689934 + heightOffset + height,
];
const point6 = [
  11584394.889440097,
  3588724.2562674284,
  496.4738873814049 + heightOffset + height,
];
```
## 4.生成网格
我们需要用到ArcGIS API中的[`Mesh（网格）`](https://developers.arcgis.com/javascript/latest/api-reference/esri-geometry-Mesh.html)类。网格主要由`vertexAttributes（顶点坐标）`、`components（网格组件）`、`spatialReference（空间参考系）`等属性组成。点击查看[Mesh类](https://developers.arcgis.com/javascript/latest/api-reference/esri-geometry-Mesh.html)的具体文档信息。  
首先我们来创建垂直于地面的网格。需要用到P4、P3、P5、P6这四个点的坐标。代码如下所示：
```javascript
// 垂直于地面的网格
const verticalMesh = new Mesh({
  vertexAttributes: {
    position: [...point4, ...point3, ...point6, ...point5],
    uv: [0, 1, 1, 1, 1, 0, 0, 0],
  },
  components: [meshComponent],
  spatialReference: { wkid: 3857 },
});
```
`vertexAttributes`是描述网格每个顶点属性的对象，里面包含顶点的位置信息`[position] `、顶点uv信息`[uv]`。  
"position: [...point4, ...point3, ...point6, ...point5]"这句代码用到了JavaScript中的展开语法（点击查看语法[详情文档](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Operators/Spread_syntax)），将每个点的[x, y, z]值展开，那么`position`的值为一个数组，包含4个点，每个点有x、y、z三个坐标值，所以`position`属性为一个有12个数值的**一维**数组。（如果包含5个点就是15个值，8个点就是24个值）  
`uv`属性为顶点uv坐标的一维数组，每个顶点包含2个值，每个值的取值范围从0到1。该属性和贴图有关，如果要将图片按正确的位置贴在网格上就要设置正确的坐标。
贴图的左上角坐标为(0, 0)，右上角为(1, 0)，左下角为(0, 1)，右下角为(1, 1)，uv坐标如下图所示：  
![uv坐标示意图](https://travelclover.github.io/img/2022/04/uv坐标示意图.png)
所以要想将图片按你想要的方向贴在网格上，就要将uv坐标与顶点坐标按顺序对应起来，绿色线条为对应指示线，如下图所示：
![贴图位置示意图](https://travelclover.github.io/img/2022/04/贴图位置示意图.png)
由P4、P3、P6、P5的顺序将uv坐标值串连起来就得到了`uv`属性值：`[0, 1, 1, 1, 1, 0, 0, 0]`。  
`spatialReference`属性为空间参考系，`3857`代表Web Mercator的空间参考。  
`components`属性为包含多个网格组件的数组，单个网格组件用来描述将某种材质应用于网格中哪些面。如果`components`包含多个`MeshComponent`就意为着将多个材质应用于网格中的多个区域。生成网格组件的代码如下：
```javascript
const meshComponent = new MeshComponent({
  faces: [0, 1, 2, 0, 2, 3],
  material: {
    colorTexture: {
      data: video,
    },
    doubleSided: true, // 是否显示两面
  },
});
```
`faces`属性指定了将材质应用于网格中的哪些面。网格是由多个三角面构成，每个三角面由3个顶点构成，`faces`数组中存储的就是顶点位置顺序的索引值，每3个值为一组， **` 逆时针 `** 绘制三角面，所以这里的`faces`属性就指定了2个三角面，构成了一个矩形面。在设置顶点位置的属性中，顶点的位置为[P4, P3, P6, P5]，索引值从0开始，[0, 1, 2]就是下图所示中由点P4、P3、P6构成的绿色三角面，[0, 2, 3]就是下图中点P4、P6、P5构成的黄色三角面，下图中蓝色数字为索引值：
![faces属性示意图](https://travelclover.github.io/img/2022/04/faces属性示意图.png)
`material`属性指定渲染网格面的材质。点击查看详细材质[文档信息](https://developers.arcgis.com/javascript/latest/api-reference/esri-geometry-support-MeshMaterial.html)。在这里，我们指定了一个视频`video`对象来作为材质。最后生成的图形添加进场景中就会一直播放视频画面。
```javascript
const video = document.createElement('video'); // 创建video标签
video.src = './video.mp4'; // 视频地址
video.autoplay = true; // 自动播放
video.muted = true; // 设置静音，设置静音后才能自动播放
video.loop = true; // 循环播放
document.body.appendChild(video); // 添加到文档中
video.style.position = 'absolute'; // 设置绝对定位
video.style.top = 0; // 设置距离顶部0像素
video.style.height = 0; // 设置高度0像素
video.style.visibility = 'hidden'; // 设置元素隐藏
```
这样我们就生成了一个垂直于地面的网格。现在我们再来生成一个平行于地面的网格，由顶点P1、P2、P3、P4构成的平面。步骤和上面的一样，只是顶点顺序不同以及uv坐标不同，因为地面的画面是垂直画面的倒影，所以示意图如下图所示：
![faces属性示意图2](https://travelclover.github.io/img/2022/04/faces属性示意图2.png)
绿色线条为uv坐标点和顶点对应的指示线，蓝色数字为`faces`属性中对顶点引用的索引值。
## 5.生成图形并添加进场景中
使用`Graphic`类将生成的垂直于地面的网格用来创建图形：
```javascript
let verticalGraphic = new Graphic({
  geometry: verticalMesh,
  symbol: {
    type: 'mesh-3d',
    symbolLayers: [{ type: 'fill' }],
  },
});
```
`geometry`属性用来指定生成的网格。  
`symbol`属性指定渲染网格类型几何体所对应的symbol。点击查看[详情文档](https://developers.arcgis.com/javascript/latest/api-reference/esri-symbols-MeshSymbol3D.html)。
然后将图形添加进三维场景中：
```javascript
view.graphics.add(verticalGraphic); // 将生成的graphic添加进视图中
```
最后用同样方法创建平行于地面的图形，并添加进场景中。
# 总结
最后地图三维场景视频融合效果如下图所示：
![最终效果图](https://travelclover.github.io/img/2022/04/最终效果图.png)
最终实现的效果，一个立着的画面，一个平铺的画面，平铺的画面为立着画面的倒影。  
实现原理其实就是通过多个顶点坐标生成几何体，用视频video对象作为材质，一起组合成一个模型，添加进场景中。通过设置网格组件列表，可以实现很多各式各样的视频融合示例，关键点是网格顶点坐标、设置三角面以及uv坐标。

# 完整代码
如需查看示例效果可点击下载[完整代码](https://download.csdn.net/download/qq_37155408/85233871)。代码中包含详细的注释，方便大家的理解。  

---
如果该文章对您有所帮助，请您一定不要吝啬您的鼓励。点赞、评论、分享、收藏、打赏都是您对我的鼓励和支持。  
如果您有`GitHub`账号，还可以[关注我~](https://github.com/travelclover)  
最后，感谢大家的阅读，如有错误，还请各位批评指正。
