# 使用ArcGIS API和Three.js在三维场景中实现动态立体墙效果
废话不多说，直接先来看下最终实现的动态立体墙效果图。
![动态立体墙效果图](https://travelclover.github.io/img/2020/08/%E5%8A%A8%E6%80%81%E7%AB%8B%E4%BD%93%E5%A2%99%E6%95%88%E6%9E%9C%E5%9B%BE.gif)
如果图片还不够直观，那么点击链接查看[在线示例](https://travelclover.github.io/demo/example/%E4%BD%BF%E7%94%A8ArcGIS%20API%E5%92%8CThree.js%E5%9C%A8%E4%B8%89%E7%BB%B4%E5%9C%BA%E6%99%AF%E4%B8%AD%E5%AE%9E%E7%8E%B0%E5%8A%A8%E6%80%81%E7%AB%8B%E4%BD%93%E5%A2%99%E6%95%88%E6%9E%9C.html)。
 
首先我们需要用到ArcGIS API中的[`externalRenderers`](https://developers.arcgis.com/javascript/latest/api-reference/esri-views-3d-externalRenderers.html)类将外部的Three.js渲染器加载到地图三维场景中，如果不知道怎么使用的可以查看我的这篇文章[《ArcGIS API在视图中渲染Three.js场景》](https://github.com/travelclover/article/blob/master/2020/ArcGIS%20API%E5%9C%A8%E8%A7%86%E5%9B%BE%E4%B8%AD%E6%B8%B2%E6%9F%93Three.js%E5%9C%BA%E6%99%AF.md)。那篇文章中加载的是一个三维模型，而本示例中只需加载一面“墙”，也就是一个平面，并增加一个动态效果。所以重点就是怎么加载一个垂直于地球表面的平面，以及如何实现动态效果。  

## 1 垂直于地球表面的墙
如图所示，先确定出两个“墙角”的坐标。
![墙角坐标点.png](https://travelclover.github.io/img/2020/08/墙角坐标点.png)
```javascript
let points = [
  [104.06179498614645, 30.659871702738265], // 坐标1
  [104.06494384459816, 30.659931252383917], // 坐标2
];
```
现在我们有了两个经纬度坐标的点，但是我们需要4个顶点才能构成一个矩形面，所以我们还需要2个点。假设2个墙角坐标贴近于地面，那么它们的高度就为0，那就再只需要2个同样经纬度坐标但高度大于0的点就能构成一个在地面上并且垂直于地面的矩形面了。所以在我们定义的`myRenderer`对象中添加一个`height`属性。
```javascript
let myRenderer = {
  // ... 其它属性
  height: 100, // 墙的高度
  // ... 其它属性、方法
};
```
现在我们有4个由经纬度加高度构成的点，如果要在视图中渲染成一个矩形面，我们要先将这4个点转换为渲染坐标系中的点，再将每3个顶点为一组构成一个三角面，最后由2个三角面构成一个矩形面。这样做是因为在Three.js中所有的模型都是由顶点加三角面构成的。  

### 1.1 顶点转换
在顶点转换之前，我们还需要做一个操作，那就是将我们的经纬度坐标转换为XY坐标。这需要用到ArcGIS API中的[`webMercatorUtils`](https://developers.arcgis.com/javascript/latest/api-reference/esri-geometry-support-webMercatorUtils.html)工具中的[`lngLatToXY`](https://developers.arcgis.com/javascript/latest/api-reference/esri-geometry-support-webMercatorUtils.html#lngLatToXY)方法，该方法将给定的经度和纬度转换为Web Mercator的XY值。
```javascript
points.forEach((point) => {
  // 将经纬度坐标转换为xy值
  let pointXY = webMercatorUtils.lngLatToXY(point[0], point[1]);
});
```
然后需要用到Three.js的数学库中的四维矩阵[`Matrix4`](https://threejs.org/docs/index.html#api/en/math/Matrix4)类以及ArcGIS API中[`externalRenderers`](https://developers.arcgis.com/javascript/latest/api-reference/esri-views-3d-externalRenderers.html)对象上的[`renderCoordinateTransformAt`]()方法将点转换为渲染坐标系中的点坐标。
```javascript
let transform = new THREE.Matrix4(); // 变换矩阵
let transformation = new Array(16);
let vector3List = []; // 顶点数组
points.forEach((point) => {
  // 将经纬度坐标转换为xy值\
  let pointXY = webMercatorUtils.lngLatToXY(point[0], point[1]);
  // 先转换高度为0的点
  transform.fromArray(
    externalRenderers.renderCoordinateTransformAt(
      this.view,
      [pointXY[0], pointXY[1], 0], // 坐标在地面上的点[x值, y值, 高度值]
      this.view.spatialReference,
      transformation
    )
  );
  vector3List.push(
    new THREE.Vector3(
      transform.elements[12],
      transform.elements[13],
      transform.elements[14]
    )
  );
  // 再转换距离地面高度为height的点
  transform.fromArray(
    externalRenderers.renderCoordinateTransformAt(
      this.view,
      [pointXY[0], pointXY[1], this.height], // 坐标在空中的点[x值, y值, 高度值]
      this.view.spatialReference,
      transformation
    )
  );
  vector3List.push(
    new THREE.Vector3(
      transform.elements[12],
      transform.elements[13],
      transform.elements[14]
    )
  );
});
```
> `renderCoordinateTransformAt`方法的作用是计算一个4x4变换矩阵，该矩阵构成从局部笛卡尔坐标系到虚拟世界坐标系的线性坐标变换。该方法传入4个参数：  
> 1 view，ArcGIS API生成的三维视图。  
> 2 origin，局部笛卡尔坐标系中原点的全局坐标，也就是[经纬度转换后的X坐标, 经纬度转换后的y坐标, 高度值]。  
> 3 srcSpatialReference，原点坐标的空间参考。  
> 4 dest，存储16个矩阵元素的数组的引用。生成的矩阵遵循OpenGL约定，其中转换组件占据第13、14和15个元素。  
  
现在，`vector3List`变量中存储的就是每个顶点转换后的三维向量，一共为4个顶点。顺序是[第一个经纬度的地面顶点, 第一个经纬度的空中顶点, 第二个经纬度的地面顶点, 第二个经纬度的空中顶点]，这个顶点的顺序很重要，后面会用到。  

### 1.2 生成三角面以及面的UV队列
因为Three.js中的面都是由小三角面构成的，所以我们需要根据顶点列表中的顶点来组成三角面，每三个顶点构成一个三角面，一定要注意构成三角面的的顶点顺序，因为要和面的UV队列一一对应起来，这样给每个面贴的纹理材质才能正确显示出来。  
纹理贴图的坐标系统是这样的：图片左下角为原点(0, 0)，右下角为(1, 0)，右上角为(1, 1)，左上角为(0, 1)，这和图片的大小宽高无关。如下图所示：  
![纹理贴图示意图.png](https://travelclover.github.io/img/2020/08/纹理贴图示意图.png)  
将纹理坐标关系转换为二维向量表示。
```javascript
const t0 = new THREE.Vector2(0, 0); // 图片左下角
const t1 = new THREE.Vector2(1, 0); // 图片右下角
const t2 = new THREE.Vector2(1, 1); // 图片右上角
const t3 = new THREE.Vector2(0, 1); // 图片左上角
```
一个简单的矩形面由4个顶点和2个小三角面构成，顶点和三角面关系如下图所示：  
![顶点顺序示意图.png](https://travelclover.github.io/img/2020/08/顶点顺序示意图.png)  
图中**0、1、2、3**序号代表`vector3List`变量中顶点的顺序。按照**逆时针**规则画出2个三角面，下三角面为绿色三角面[0, 2, 1]，上三角面为蓝色三角面[1, 2, 3]。例如，要将纹理贴图和绿色三角面映射起来，那么绿色三角面对应的UV就是[t0, t1, t3]，蓝色三角面对应的UV就是[t3, t1, t2]。  
![顶点和纹理坐标向量对应图.png](https://travelclover.github.io/img/2020/08/顶点和纹理坐标向量对应图.png)  
根据以上原理生成三角面列表以及UV队列。
```javascript
let faceList = []; // 三角面数组
let faceVertexUvs = []; // 面的 UV 层的队列，该队列用于将纹理和几何信息进行映射

for (let i = 0; i < vector3List.length - 2; i++) {
  if (i % 2 === 0) { // 下三角面
    faceList.push(new THREE.Face3(i, i + 2, i + 1));
    faceVertexUvs.push([t0, t1, t3]);
  } else { // 上三角面
    faceList.push(new THREE.Face3(i, i + 1, i + 2));
    faceVertexUvs.push([t3, t1, t2]);
  }
}
```

### 1.3 生成几何体
使用Three.js中的[`Geometry`](https://threejs.org/docs/index.html#api/en/core/Geometry)构造函数来生成自定义几何体。
```javascript
const geometry = new THREE.Geometry(); // 生成几何体
geometry.vertices = vector3List; // 几何体顶点
geometry.faces = faceList; // 几何体三角面
geometry.faceVertexUvs[0] = faceVertexUvs; // 面的UV队列，用于将纹理信息映射到几何体上
```
> `geometry.faceVertexUvs`的属性值为数组是因为有多组UV。颜色贴图、法线贴图、高光贴图、金属度贴图等共用一组纹理坐标UV即`geometry.faceVertexUvs[0]`，设置阴影的光照贴图lightMap使用另外一组纹理坐标，也就是`geometry.faceVertexUvs[1]`。默认情况下，`geometry.faceVertexUvs`属性中会存在一个元素，所以可以直接对`geometry.faceVertexUvs[0]`进行赋值操作。  
> **注意**：对于缓冲区类型几何体也就是通过[`BufferGeometry`](https://threejs.org/docs/index.html#api/en/core/BufferGeometry)构造函数生成的几何体，是通过设置.attributes.uv和.attributes.uv2两个属性分别定义两组顶点纹理坐标。
