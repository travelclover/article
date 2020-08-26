# ArcGIS API中PictureMarkerSymbol使用GIF图片
在前几天，ArcGIS API for JavaScript发布了新版本，也就是4.15版本。其中有一个小功能更新，我认为还是挺有用的，可以使我们地图的表现形式更加丰富。这个小的功能更新就是[`PictureMarkerSymbol`](https://developers.arcgis.com/javascript/latest/api-reference/esri-symbols-PictureMarkerSymbol.html)支持GIF动态图和PNG格式的透明图了，不过有点遗憾的是不支持在3D场景[SceneView](https://developers.arcgis.com/javascript/latest/api-reference/esri-views-SceneView.html)中使用，只能在2D [MapView](https://developers.arcgis.com/javascript/latest/api-reference/esri-views-MapView.html)中使用。下面我们就来试一下这个功能，做一个简单的例子来看一看在地图中加载GIF动态图的效果。  

## 动态图片资源
首先，我们需要一个格式GIF的动态图片资源，随便在网上找一个就行，我找的是一个可爱的小乌龟。  
![gif图](https://travelclover.github.io/img/2020/04/gif.gif)

## 创建地图
```javascript
const map = new Map({
    basemap: 'topo-vector', // 底图
});
const view = new MapView({
    container: 'viewDiv', // 包含视图的DOM元素
    map, // 要在视图中显示的Map对象的实例
    center: [105, 29], // 视图的中心点
    zoom: 6, // 表示视图中心的详细程度
});
```

## 创建点数据
因为[`PictureMarkerSymbol`](https://developers.arcgis.com/javascript/latest/api-reference/esri-symbols-PictureMarkerSymbol.html)是用来渲染[`Point`](https://developers.arcgis.com/javascript/latest/api-reference/esri-geometry-Point.html)点图形的，所以我们需要创建一些点数据。
```javascript
const points = [ // 点坐标数组
    [116.39, 39.92],
    [104.02, 30.63],
    [108.87, 34.29],
];
const graphics = points.map(point => ({
    geometry: {
        type: 'point',
        longitude: point[0], // 经度
        latitude: point[1], // 纬度
    },
}));
```

## 设置symbol
```javascript
const symbol = {
    type: 'picture-marker',
    url: './images/rain.gif',
    width: '128px',
    height: '128px',
};
```
> **需要注意的是**：*width* 和 *height* 属性值不能设置超过**200px**。

## 创建图层以及设置图层renderer
```javascript
// 设置渲染
const renderer = {
    type: 'simple',
    symbol,
};
// 创建要素图层
const layer = new FeatureLayer({
    source: graphics,
    objectIdField: 'ObjectID',
    renderer,
});
// 把图层添加到地图中
map.add(layer);
```

## 展示效果
最后，我们就来看一下效果吧。symbol已经动起来了！  
![效果图](https://travelclover.github.io/img/2020/04/ditu-gui.gif)

## 思考1：能否在3D场景SceneView中使用
虽然文档上已经说了在SceneView中不支持PictureMarkerSymbol显示GIF格式图片，但我还是好奇决定试一下，看一看是什么效果。最终经过实际测试，发现在3D场景中其实能加载GIF图片，能成功地显示出来，只是不能让图片动起来，和普通图片一样。如下图所示：  
![SceneView中PictureMarkerSymbol加载GIF效果图](https://travelclover.github.io/img/2020/04/SceneView-gif.jpg)

## 思考2：该功能的实际用处有哪些
我第一想到的就是我们能通过该功能实现新闻联播中天气预报各种气象动态效果，比如下雨天，如下图所示：  
![天气预报雨天效果图](https://travelclover.github.io/img/2020/04/map-rain.gif)  

***
**以下是完整代码**
```html
<!DOCTYPE html>
<html>
  <head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>ArcGIS API中PictureMarkerSymbol使用GIF图片</title>

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
      href="https://js.arcgis.com/4.15/esri/css/main.css"
    />
    <script src="https://js.arcgis.com/4.15/"></script>

    <script type="module">
      require([
        'esri/Map',
        'esri/views/SceneView',
        'esri/views/MapView',
        'esri/layers/FeatureLayer',
      ], function (Map, SceneView, MapView, FeatureLayer) {
        const map = new Map({
          basemap: 'topo-vector',
        });

        const view = new MapView({
          container: 'viewDiv',
          map: map,
          center: [105, 29],
          zoom: 5,
        });

        const points = [
          [116.39, 39.92],
          [104.02, 30.63],
          [108.87, 34.29],
        ];

        const symbol = {
          type: 'picture-marker',
          url: './images/gif.gif',
          width: '128px',
          height: '128px',
        };

        const graphics = points.map((point) => ({
          geometry: {
            type: 'point',
            longitude: point[0],
            latitude: point[1],
          },
        }));

        const renderer = {
          type: 'simple',
          symbol,
        };

        const layer = new FeatureLayer({
          source: graphics,
          objectIdField: 'ObjectID',
          renderer,
        });

        map.add(layer);
      });
    </script>
  </head>
  <body>
    <div id="viewDiv"></div>
  </body>
</html>
```
