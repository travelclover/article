# ArcGIS JS API+Three.js实现动态航线效果
首先我们来看下效果吧~  
![ArcGIS+Three.js实现航迹线效果图1](https://travelclover.github.io/img/2021/04/ArcGIS%2BThree.js实现航迹线效果1.gif)
![ArcGIS+Three.js实现航迹线效果图2](https://travelclover.github.io/img/2021/04/ArcGIS%2BThree.js实现航迹线效果2.gif)
该示例里有两个动态效果，一个是光线沿着类似抛物线轨迹运动的效果，一个是光线落地后有一个类似波纹扩散的效果。下面就来详细看看怎么实现这两个效果。
## 动态航线效果
实现动态航线的原理很简单，就是每隔一段时间就更新线段的位置，连贯起来就是线段流动的效果。那么首先我们就要得到一条曲线轨迹，three.js中有一个[CatmullRomCurve3](https://threejs.org/docs/index.html#api/zh/extras/curves/CatmullRomCurve3)类，可以用来创建一条平滑的曲线。我们需要三个点来创建曲线，除了起点，终点，我们还需要一个中间点。
![三个点创建曲线](https://travelclover.github.io/img/2021/04/三点曲线.png)

```javascript
const startCoordinate = [116.46, 39.92, 0]; // 起点经纬度坐标、高度值
const endCoordinate = [104.06, 30.67, 0]; // 终点经纬度坐标、高度值
const centerCoordinate = [110.26, 35.295, 200000]; // 中间点经纬度坐标、高度值
```
现在是经纬度加高度值的点，我们需要把点转换成渲染坐标系中的点。下面代码是一个转换方法，将[经度, 纬度, 高度]值转换成渲染坐标系中的[x, y, z]值。
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
    view,
    [pointXY[0], pointXY[1], height], // 坐标在地面上的点[x值, y值, 高度值]
    view.spatialReference,
    transformation
  );
  return [transformation[12], transformation[13], transformation[14]];
}
```
该方法中用到了[`webMercatorUtils.lngLatToXY`](https://developers.arcgis.com/javascript/latest/api-reference/esri-geometry-support-webMercatorUtils.html#lngLatToXY)方法，将给定的纬度和经度（十进制度）值转换为Web Mercator的XY值。还用到了[`externalRenderers.renderCoordinateTransformAt`](https://developers.arcgis.com/javascript/latest/api-reference/esri-views-3d-externalRenderers.html#renderCoordinateTransformAt)方法，用来将坐标转换成渲染坐标系中的值，具体可点击查看[详细文档](https://developers.arcgis.com/javascript/latest/api-reference/esri-views-3d-externalRenderers.html#renderCoordinateTransformAt)。  
将起点、终点、中间点都转换成渲染坐标系中的点。
```javascript
let startPoint = pointTransform(...startCoordinate);
let endPoint = pointTransform(...endCoordinate);
let centerPoint = pointTransform(...centerCoordinate);
```
得到三个渲染坐标系中的点后，就可以用three.js中的[CatmullRomCurve3](https://threejs.org/docs/index.html#api/zh/extras/curves/CatmullRomCurve3)类来生成一条平滑的曲线。
```javascript
const curve = new THREE.CatmullRomCurve3([
  new THREE.Vector3(...startPoint),
  new THREE.Vector3(...centerPoint),
  new THREE.Vector3(...endPoint),
]);
```
有了曲线后，我们需要将曲线分割成许多段，得到分割点的列表数据，后面用这些点数据来更新光线的位置。如下图所示，黑线代表完整的曲线轨迹，红色圆点代表分割点，蓝色线段代表移动的光线。航线动态效果实际上就是改变蓝色线段的位置，举个例子，当前状态如下面那条曲线所示，光线由第3，4，5这3个点坐标构成，下一次渲染画面的时候，状态就如上面那条曲线所示，光线由第4，5，6这3个点构成，连贯起来就是光线向前运动的感觉。
![曲线分割示意图](https://travelclover.github.io/img/2021/04/曲线分割示意图.png)
CatmullRomCurve3的实例上有一个[getPoints](https://threejs.org/docs/index.html#api/zh/extras/core/Curve.getPoints)方法，用来将曲线分割成指定段数，并返回分割点列表。
```javascript
const points = curve.getPoints(60); // 分割成60段，返回61个点 points.length 为61
```
用分割后的点数据创建光线。
```javascript
const highLightGeometry = new THREE.Geometry();
highLightGeometry.vertices = points.slice(0, 3); // 将分割后的前三个点赋值给顶点数据，后面只需改变这个顶点数组
highLightGeometry.verticesNeedUpdate = true; // 如果顶点队列中的数据被修改，该值需要被设置为 true
highLightGeometry.colors = [
  new THREE.Color('#ffff00'),
  new THREE.Color('#ffffff'),
  new THREE.Color('#ffff00'),
];
let highLight = new THREE.Line(
  highLightGeometry,
  highLightMaterial
);
scene.add(highLight);
```
现在已经创建好`光线`了，要让它移动起来，只需要在渲染方法中改变它的顶点坐标就行了，也就是改变`highLight.geometry.vertices `的值。
## 波纹扩散效果
波纹扩撒的效果原理也很简单，创建一个平面圆，随着时间改变圆的透明度和大小。圆的位置很好求出来，就是文章前面提到的方法，将圆中心点的经纬度坐标转换成渲染坐标系中的点坐标就行了。只不过需要注意的是，位置放对了，圆的姿态大概率是错误的，正确的姿态是圆平面应该平行于地面，所以还需要根据计算出来的圆中心点坐标和三角函数来计算圆平面调整的角度。
```javascript
// 计算调整姿态的角度
let deltaX = Math.atan(this.endPoint[2] / this.endPoint[1]);
let deltaZ = Math.atan(
  this.endPoint[0] /
    Math.sqrt(
      this.endPoint[1] * this.endPoint[1] +
        this.endPoint[2] * this.endPoint[2]
  )
);
// 如果 y < 0 需要加上180°
if (this.endPoint[1] < 0) {
  deltaX += Math.PI;
} else {
  deltaZ *= -1;
}
```
```javascript
// 调整平面圆的姿态
circleMesh.rotation.x = deltaX;
circleMesh.rotation.z = deltaZ;
circleMesh.rotateOnAxis(new THREE.Vector3(1, 0, 0), Math.PI / 2); // 再沿X轴旋转90°
```
同样的，还需要在渲染方法中添加改变透明度和大小的代码。
```javascript
render() {
  ...
  ...
  circleMesh.material.opacity = 透明度; // 透明度
  circleMesh.scale.set(缩放比例, 缩放比例, 缩放比例); // 修改缩放比例
  ...
  ...
}
```

## 完整代码
如需查看示例效果可点击下载[完整代码](https://download.csdn.net/download/qq_37155408/16661269)。  

---
如果该文章对您有所帮助，请您一定不要吝啬您的鼓励。点赞、评论、分享、收藏、打赏都是您对我的鼓励和支持。  
如果您有`GitHub`账号，还可以[关注我~](https://github.com/travelclover)  
最后，感谢大家的阅读，如有错误，还请各位批评指正。
