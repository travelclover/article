# ArcGIS JS API之ExtrudeSymbol3DLayer实现白膜效果

在三维地理信息可视化中，白模效果（也称为建筑轮廓效果）是一种常见的技术，用于展示建筑物的基本轮廓和高度信息。通过这种效果，用户可以快速了解城市或区域的建筑分布和高度差异。本文将详细介绍如何使用ArcGIS JS API中的ExtrudeSymbol3DLayer来实现白模效果。  
![白模效果](https://travelclover.github.io/img/2025/02/白模效果-min.gif)

# 1 概述
[ArcGIS JS API](https://developers.arcgis.com/javascript/latest/)是一个强大的JavaScript库，用于构建基于Web的地理信息系统应用。它提供了丰富的API来创建、管理和展示地图、图层、符号等。在三维场景中，[ExtrudeSymbol3DLayer](https://developers.arcgis.com/javascript/latest/api-reference/esri-symbols-ExtrudeSymbol3DLayer.html)是一个常用的符号图层，用于将二维多边形拉伸为三维体块，从而实现建筑物的白模效果。  

# 2 实现步骤
## 2.1 引入ArcGIS JS API
首先，我们需要在HTML文件中引入ArcGIS JS API的CSS和JavaScript文件。这些文件可以通过CDN加载，如下所示：
```html
<link rel="stylesheet" href="https://js.arcgis.com/4.31/esri/themes/light/main.css" />
<script src="https://js.arcgis.com/4.31/"></script>
```
## 2.2 准备数据
在实现白模效果之前，我们需要准备建筑物的数据。这些数据通常包括建筑物的几何信息（如多边形）和属性信息（如类型、高度等）。在本例中，我们假设数据已经存储在buildingData.js文件中，并通过`<script>`标签引入。  
```javascript
<script src="./buildingData.js"></script>
```
大致数据结构如下所示:
```javascript
const buildingData = [
  {
    // 要素几何信息
    geometry: {
      spatialReference: { latestWkid: 4326, wkid: 4326 },
      rings: [
        [
          [-75.08778761499997, 38.32938676900005],
          [-75.08793513999996, 38.32910149700001],
          [-75.08813620000001, 38.32916815500005],
          [-75.088124521, 38.329191802000025],
          [-75.08818068099998, 38.32921165000005],
          [-75.088168, 38.32923679400005],
          [-75.08823773699999, 38.32926013400004],
          [-75.08811461799996, 38.32948719500001],
          [-75.08778761499997, 38.32938676900005],
          [-75.08778761499997, 38.32938676900005],
        ],
      ],
    },
    // 要素关联的属性信息
    attributes: {
      OBJECTID: 1, // 唯一值ID
      TYPE: '住宅公寓', // 类型
      HEIGHT: 11, // 高度
    },
  },
  // 其它要素数据
];
```
## 2.3 创建FeatureLayer
有了相关数据，我们可以通过客户端要素数据创建FeatureLayer，具体方法可以参考这篇文章[《ArcGIS JS API之通过客户端要素创建FeatureLayer》](https://github.com/travelclover/article/blob/master/2025/ArcGIS%20JS%20API%E4%B9%8B%E9%80%9A%E8%BF%87%E5%AE%A2%E6%88%B7%E7%AB%AF%E8%A6%81%E7%B4%A0%E5%88%9B%E5%BB%BAFeatureLayer.md)。  

## 2.4 创建符号
为了实现白模效果，我们需要为不同类型的建筑物创建不同的符号。[ExtrudeSymbol3DLayer](https://developers.arcgis.com/javascript/latest/api-reference/esri-symbols-ExtrudeSymbol3DLayer.html)用于定义建筑物的拉伸效果，而[PolygonSymbol3D](https://developers.arcgis.com/javascript/latest/api-reference/esri-symbols-PolygonSymbol3D.html)则用于将符号应用到多边形几何上。  
```javascript
function createSymbol(color) {
  return new PolygonSymbol3D({
    symbolLayers: [
      new ExtrudeSymbol3DLayer({
        material: {
          color: color, // 材质颜色，根据类型不同生成不同颜色的符号
        },
        // 轮廓边缘效果
        edges: {
          type: 'solid',
          color: '#999',
          size: 0.5,
        },
        // 这里暂时不需要
        // size: 10, // 拉伸的高度（以米为单位）。负值将使多边形表面向下拉伸至地面或低于地面。
      }),
    ],
  });
}
```
在这个函数中，我们为每个建筑物类型定义了一个颜色，并通过ExtrudeSymbol3DLayer设置了建筑物的材质和边缘效果。  
ExtrudeSymbol3DLayer中还有一个设置拉伸高度的属性`size`，在上面代码中我们注释掉了，原因是在当前这个例子中，我们不能将所有的符号高度都设置为一个值，而是需要跟随每个要素属性中的高度值来设置，这就是后续将要讲到的通过视觉变量来设置拉伸高度，才有建筑物高低起伏的效果。  
## 2.5 创建渲染器
为了将不同的符号应用到不同类型的建筑物上，我们需要创建一个唯一值渲染器[UniqueValueRenderer](https://developers.arcgis.com/javascript/latest/api-reference/esri-renderers-UniqueValueRenderer.html)。这个渲染器根据建筑物的类型字段（如TYPE）来分配不同的符号。  
```javascript
function createRenderer() {
  const rendererOptions = {
    type: 'unique-value',
    defaultSymbol: createSymbol('#FFFFFF'),
    defaultLabel: 'Other',
    field: 'TYPE', // 设置根据哪个字段进行分类
    uniqueValueInfos: [
      // 设置每个类型的符号
      {
        value: '住宅',
        symbol: createSymbol('#A7C636'),
        label: '住宅',
      },
      {
        value: '公共/政府',
        symbol: createSymbol('#3f51b5'),
        label: '公共/政府',
      },
      // ...其它类型
    ],
    // 视觉变量设置
    visualVariables: [
      {
        type: 'size', // 类型为大小视觉变量
        field: 'HEIGHT', // 根据属性中的哪个字段来设置拉伸的高度值
      },
    ],
  };
  return new UniqueValueRenderer(rendererOptions);
}
```
在这个渲染器中，我们为每种建筑物类型分配了一个颜色，并通过visualVariables设置了建筑物的高度。visualVariables为一个数组，是因为除了设置[`size`](https://developers.arcgis.com/javascript/latest/api-reference/esri-renderers-visualVariables-SizeVariable.html)大小视觉变量，还可以设置颜色视觉变量[`color`](https://developers.arcgis.com/javascript/latest/api-reference/esri-renderers-visualVariables-ColorVariable.html)、透明度视觉变量[`opacity`](https://developers.arcgis.com/javascript/latest/api-reference/esri-renderers-visualVariables-OpacityVariable.html)和旋转视觉变量[`rotation`](https://developers.arcgis.com/javascript/latest/api-reference/esri-renderers-visualVariables-RotationVariable.html)。  

## 2.6 初始化场景
最后，我们需要初始化一个SceneView来展示三维场景。SceneView是ArcGIS JS API中用于显示三维地图的视图。再将创建的要素图层添加到地图上即可。
```javascript
const view = new SceneView({
  container: 'viewDiv',
  map: new Map({
    basemap: 'satellite', // 影像底图
  }),
});

view.map.add(layer); // 将图层添加到地图中
```

# 3 总结
通过以上步骤，我们成功地使用ArcGIS JS API实现了白模效果。这种效果不仅能够清晰地展示建筑物的轮廓和高度，还能够通过颜色区分不同类型的建筑物。ArcGIS JS API提供了强大的三维可视化功能，使得开发者能够轻松构建复杂的地理信息应用。  
# 4 完整代码
通过关注微信公众号《Web与GIS》，回复关键字：`ES3DSXBMXG`下载源码。
  
![微信公众号](https://travelclover.github.io/img/2025/02/微信公众号.png)

---
如果该文章对您有所帮助，请您一定不要吝啬您的鼓励。点赞、评论、分享、收藏、打赏都是您对我的鼓励和支持。  
如果您有`GitHub`账号，还可以[关注我~](https://github.com/travelclover)  
最后，感谢大家的阅读，如有错误，还请各位批评指正。
