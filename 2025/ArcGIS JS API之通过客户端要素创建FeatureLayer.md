# ArcGIS JS API之通过客户端要素创建FeatureLayer  

在WebGIS开发中，ArcGIS JS API是一个非常强大的工具，它允许开发者轻松地将地图和地理空间数据集成到Web应用中。本文将详细介绍如何通过客户端要素创建[FeatureLayer](https://developers.arcgis.com/javascript/latest/api-reference/esri-layers-FeatureLayer.html)，并在地图中展示这些要素。我们将使用ArcGIS JS API 4.31版本，并结合一个简单的示例代码来演示这一过程。  

![通过客户端要素创建FeatureLayer](https://travelclover.github.io/img/2025/02/通过客户端要素创建FeatureLayer.jpg) 

#  1 什么是FeatureLayer？
`FeatureLayer`是 ArcGIS JS API 中的一个核心图层类，用于表示地图中的矢量数据。它可以包含点、线、面等几何类型，并且每个要素都可以关联属性信息。`FeatureLayer`支持交互操作，如点击要素弹出信息窗口等。  
通过 ArcGIS JS API 创建 FeatureLayer 有以下三种方式：  
 1. 通过引用服务的URL。直接设置`url`属性来创建要素服务实例，这是创建要素图层最常用的方式，但前提是已经有可用的要素服务或地图服务。  
```javascript
const layer = new FeatureLayer({
  url: "https://sampleserver6.arcgisonline.com/arcgis/rest/services/Census/MapServer/3"
});
```
 2. 通过引用 ArcGIS 门户项目 ID。如果 FeatureLayer 作为项目存在于 ArcGIS Online 或 ArcGIS Enterprise 中，您也可以根据其 ID 创建 FeatureLayer。
```javascript
const layer = new FeatureLayer({
  portalItem: {
    id: "8444e275037549c1acab02d2626daaee"
  }
});
```
 3. 通过客户端要素。当没有可用的服务和门户网站时，可以通过本地客户端要素来创建 FeatureLayer，这就是本文接下来要介绍的方式。

# 2 通过客户端要素创建FeatureLayer
在某些场景下，我们可能需要在客户端动态生成要素数据，而不是从服务器加载。ArcGIS JS API允许我们通过客户端要素创建`FeatureLayer`。下面我们将通过一个示例来演示如何实现这一功能。

## 2.1 准备工作
首先，我们需要引入ArcGIS JS API的CSS和JavaScript文件。在HTML文件的`<head>`部分，添加以下代码：
```html
<link rel="stylesheet" href="https://js.arcgis.com/4.31/esri/themes/light/main.css" />
<script src="https://js.arcgis.com/4.31/"></script>
```
接下来，我们需要准备一些客户端要素数据。这些数据可以是一个包含几何信息和属性信息的数组，在后续创建要素图层时会用到。在本示例中，我们假设这些数据已经存储在一个名为buildingData.js的文件中，并且通过`<script>`标签引入：
```html
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
      OBJECTID: 1,
      TYPE: '住宅公寓',
      HEIGHT: 11,
    },
  },
  // 其它要素数据
];
```

## 2.2 设置要素图层所需属性
通过服务地址创建图层时可以只设置一个`url`属性，但通过客户端要素创建图层时需要设置多个属性，个别属性在特殊情况下是必须设置的。
### 2.2.1 fields属性
fields属性表示图层中的字段数组，每个字段都表示一个属性。
```javascript
const fields = [
  { name: 'OBJECTID', alias: 'OBJECTID', type: 'oid' },
  { name: 'TYPE', alias: '类型', type: 'string' },
  { name: 'HEIGHT', alias: '高度', type: 'integer' },
];
const layerOptions = {
  // ... 其它图层属性
  fields: fields, // 字段数组
};
```
> **name**：字段名称。**alias**：字段的显示名称。**type**：字段的数据类型。  

***`注意:`*** 需要用到的关联字段都应在这列出。  

### 2.2.2 objectIdField属性
objectIdField 是表示每个要素的唯一值字段。通过客户端要素集合构造 FeatureLayer 时，此属性是必需的，应包含在fields数组中，且类型应该为`oid`。如果未指定，它将从 fields 数组中推断出来。
```javascript
const layerOptions = {
  // ... 其它图层属性
  objectIdField: 'OBJECTID', // 字段数组
};
```

### 2.2.3 geometryType属性
geometryType 表示图层中要素的几何类型。从客户端要素创建 FeatureLayer 时，此属性可以由图层的 source 属性中提供的要素的 geometryType 推断出来。如果图层在初始化时 source 为空数组，则必须设置此属性。  
```javascript
const layerOptions = {
  // ... 其它图层属性
  geometryType: 'polygon', // source 为空数组时，必须设置该属性
  source: [], // 
};
```

### 2.2.4 spatialReference属性
spatialReference 表示图层的空间参考。如果图层在初始化时 source 为空数组，则必须设置此属性。  
```javascript
const layerOptions = {
  // ... 其它图层属性
  spatialReference: { wkid: 4326 }, // source 为空数组时，必须设置该属性
  source: [], // 
};
```

### 2.2.5 source属性
用于创建 FeatureLayer 的 Graphic 对象的集合。从客户端要素创建 FeatureLayer 时，必须设置此属性。
```javascript
const graphics = buildingData.map(
  (item) =>
    new Graphic({
      geometry: new Polygon(item.geometry),
      attributes: item.attributes,
    })
);

const layerOptions = {
  // ... 其它图层属性
  source: graphics, // 
};
```

## 2.3 初始化地图并添加FeatureLayer
接下来，我们需要初始化地图并将FeatureLayer添加到地图中。
```javascript
const view = new SceneView({
  container: 'viewDiv',
  map: new Map({
    basemap: 'topo', // 设置底图
    ground: 'world-elevation', // 世界高程
  }),
});

const layer = new FeatureLayer(layerOptions); // 创建要素图层
view.map.add(layer); // 向地图中添加图层
```
最后可以通过`goTo()`方法，将地图缩放到图层所在范围。
```javascript
view.whenLayerView(layer).then(() => {
  view.goTo(layer.fullExtent);
});
```

# 3 总结
通过本文的介绍，我们学习了如何使用ArcGIS JS API通过客户端要素创建FeatureLayer。我们首先准备了客户端要素数据，然后将其转换为Graphic对象，最后通过设置必要参数创建FeatureLayer并将这些要素展示在地图上。这种方法非常适合在客户端动态生成地理数据的场景，能够极大地提高WebGIS应用的灵活性和交互性。  

# 4 完整代码
通过关注微信公众号《Web与GIS》，回复关键字：`TGKHDYSCJTC`下载源码。  

![微信公众号](https://travelclover.github.io/img/2025/02/微信公众号.png)

---
如果该文章对您有所帮助，请您一定不要吝啬您的鼓励。点赞、评论、分享、收藏、打赏都是您对我的鼓励和支持。  
如果您有`GitHub`账号，还可以[关注我~](https://github.com/travelclover)  
最后，感谢大家的阅读，如有错误，还请各位批评指正。
