# Cesium通过PostProcess后处理实现圆的扩散效果

在现代WebGIS开发中，Cesium作为一款强大的三维地球可视化库，提供了丰富的功能来创建各种视觉效果。本文将详细介绍如何利用Cesium的PostProcess后处理功能实现一个圆形的扩散效果，这种效果可以用于表示某个事件或现象的影响范围，比如疫情传播、自然灾害影响区域等，也可以用于表现某种力量的扩散，比如爆炸、能量波动等。  
后处理(Post-Processing)是计算机图形学中常见的技术，指在场景渲染完成后对图像进行额外的处理。在Cesium中，我们可以通过PostProcessStage来实现各种后处理效果，如模糊、发光、色彩校正等。  

https://github.com/user-attachments/assets/446167da-58bb-4e30-a1aa-79f4f1726dab


# 1 效果展示
我们将实现一个`以某个地理位置为圆心、向外扩散的红色圆圈特效`，随着时间推进，圆圈会不断扩张并循环动画，如同水波一样向外扩散。  
![cesium圆扩散效果](https://github.com/user-attachments/assets/8cc116f6-c80b-48c2-a055-8299c965e089)

# 2 实现圆形扩散效果的核心思路
我们的目标是创建一个以某点为中心，随时间向外扩散的圆形效果。实现这一效果需要以下几个关键步骤：
1. 获取当前渲染的像素在世界坐标系中的位置
2. 计算该像素在平行于地面的目标中心点所在平面的投影坐标
3. 计算投影坐标点与目标中心点的距离
4. 根据距离和时间参数计算出该像素需叠加颜色的透明度
5. 将对应透明度的叠加颜色与原场景颜色混合

# 3 代码实现详解
## 3.1 基础设置
首先，我们创建一个基本的Cesium场景并添加OSM建筑数据：
```javascript
const viewer = new Cesium.Viewer('cesiumContainer', {
  terrain: Cesium.Terrain.fromWorldTerrain(), // 地形数据
});

const longitude = 104.06334672354294; // 经度
const latitude = 30.659847825898527; // 纬度
const height = 460.5298252741061; // 高度
const point = Cesium.Cartesian3.fromDegrees(longitude, latitude, height);

// 跳转摄像头
viewer.camera.flyTo({
  destination: Cesium.Cartesian3.fromDegrees(longitude, latitude, 3000),
});

// 加载 OSM 建筑
const buildingsTileset = await Cesium.createOsmBuildingsAsync();
viewer.scene.primitives.add(buildingsTileset);
```

## 3.2 创建后处理阶段
后处理的核心是 `PostProcessStage`，我们在其中自定义了一个片元着色器 `fragmentShader`，它会对场景中每个像素（片元）进行处理。
```javascript
const circleEffect = viewer.scene.postProcessStages.add(
  new Cesium.PostProcessStage({
    name: 'circleEffect',
    fragmentShader: `...`, // 详细着色器代码见下文
    uniforms: {
      u_inverseViewMatrix: () => Cesium.Matrix4.inverse(viewer.camera.viewMatrix, new Cesium.Matrix4()), // 逆视图矩阵
      u_position: () => transformedEyePosition, // 视空间中的圆心位置
      u_panelNormal: () => normalVector, // 平行于地面的平面的法向量
      u_radius: 500, // 圆扩散的最大半径
      u_duration: 2000, // 圆扩散的持续时间
      u_time: () => performance.now(), // 当前时间戳
    },
  })
);

```

## 3.3 Uniforms参数详解
在Cesium中，后处理阶段需要一些动态参数，通过uniforms传递数据给shader。  
**`u_inverseViewMatrix`**: 视图矩阵的逆矩阵，用于将视空间坐标转换为世界空间坐标。如果在着色器中需要重建世界坐标则需要该矩阵。
```javascript
function() {
  return Cesium.Matrix4.inverse(viewer.camera.viewMatrix, new Cesium.Matrix4());
}
```
**`u_panelNormal`**: 平行于地面的圆中心点所在平面的法向量。在shader中用于计算像素所对应的视空间坐标点在该平面上的投影点坐标。
```javascript
function () {
  // 高于地面500米的点
  const pointH = Cesium.Cartesian3.fromDegrees(
    longitude,
    latitude,
    height + 500 // 高于地面500米
  );
  const viewMatrix = viewer.camera.viewMatrix; // 视图矩阵
  const transformedPoint = Cesium.Matrix4.multiplyByPoint(
    viewMatrix,
    point,
    new Cesium.Cartesian3()
  );
  const transformedPointH = Cesium.Matrix4.multiplyByPoint(
    viewMatrix,
    pointH,
    new Cesium.Cartesian3()
  );
  // 计算法向量
  let normal = Cesium.Cartesian3.subtract(
    transformedPointH,
    transformedPoint,
    new Cesium.Cartesian3()
  );
  Cesium.Cartesian3.normalize(normal, normal); // 归一化，单位法向量
  return normal;
}
```
![法向量求平面示意图](https://github.com/user-attachments/assets/5eb2104a-1303-444f-9a97-db3ec090ac27)
  
通过点A和点B求出法向量，通过法向量和点A坐标算出平行于地面的平面，后续可以求得其他点在该平面的投影点坐标。  

**`u_position`**: 圆扩散的中心点在视空间中的位置。  
```javascript
function () {
  const viewMatrix = viewer.camera.viewMatrix; // 视图矩阵
  const transformedPoint = Cesium.Matrix4.multiplyByPoint(
    viewMatrix,
    point,
    new Cesium.Cartesian3()
  );
  return transformedPoint;
}
```

## 3.4 片元着色器代码详解
核心目标是确定当前片元是否位于圆形区域内。要实现这一点，我们需要正确计算出每个像素在世界空间中的准确位置，再通过位置坐标去算距离，过程如下：  

### 3.4.1 获取片元深度值
根据纹理坐标和深度纹理计算出深度值，有了深度值才能计算片元对应的空间坐标。
```glsl
vec4 deepColor = texture(depthTexture, v_textureCoordinates);
float depth = czm_unpackDepth(deepColor);
```
在Cesium中，`czm_unpackDepth`是一个GLSL函数，用于从深度纹理（如深度缓冲区或深度贴图）中解压出原始的深度值。它的作用是将压缩存储的深度值还原为原始的浮点数深度值（范围为[0, 1]）。

### 3.4.2 将片元坐标转换为视空间坐标
将窗口坐标+深度值转换为视空间坐标。
```glsl
vec4 eyeCoord = czm_windowToEyeCoordinates(gl_FragCoord.xy, depth);
vec3 eyeCoordinate3 = eyeCoord.xyz / eyeCoord.w;
```
**gl_FragCoord.xy**：当前片元在屏幕空间（窗口空间）的坐标（单位：像素）。  
**czm_windowToEyeCoordinates**：是Cesium内置函数，将屏幕空间坐标 + 深度值转换为视空间齐次坐标（vec4）。  
对视空间齐次坐标进行透视除法（除以w分量），得到眼空间中的3D坐标**eyeCoordinate3**。这一步是将齐次坐标转换为笛卡尔坐标，得到点在视图坐标系中的真实位置。  

### 3.4.3 从视空间反推世界空间（选做）
通常我们希望在世界坐标系下计算距离。但由于世界坐标值较大，精度容易损失，在这个效果中我们直接在视空间中进行距离计算，避免抖动。
```glsl
// 计算世界坐标
vec4 worldPosition = u_inverseViewMatrix * vec4(eyeCoordinate3, 1.0);
vec3 worldCoordinate = worldPosition.xyz / worldPosition.w;
```

### 3.4.4 将视空间点投影到圆所在平面
```glsl
// 计算点到平面的投影点的函数
vec3 projectPointOnPlane(vec3 normal, vec3 pointA, vec3 pointB) {
  // 确保法向量是单位向量（归一化）
  vec3 n = normalize(normal);
  // 计算点B到点A的向量
  vec3 ba = pointB - pointA;
  // 计算点B到平面的垂直距离:dot(n, ba)
  float distance = dot(n, ba);
  // 计算投影点坐标
  vec3 projection = pointB - distance * n;
  return projection;
}
```
调用函数计算投影点坐标：
```glsl
vec3 projectionPoint = projectPointOnPlane(u_panelNormal, u_position, eyeCoordinate3);
```
这个函数会将当前像素点在视空间中的位置投影到以圆心为中心、法向量为 `u_panelNormal `的平面上，确保我们的扩散距离只在这个平面内判断。  

![distance计算示意图](https://github.com/user-attachments/assets/2d447727-4f93-46c8-b431-53f3c59ab8e1)


### 3.4.5 计算与圆心的距离
使用`length()`函数计算距离：
```glsl
float distance = length(projectionPoint - u_position); // 用视空间坐标计算距离，优点：数值较小，精度较高，不会产生抖动效果
```
如果是通过`3.4.3`步骤中算出的世界坐标去计算距离，则在Uniforms中的u_position变量不需要用视图矩阵去对圆心坐标做转换，用原始笛卡尔坐标。  
```glsl
float distance = length(worldPosition.xyz - u_position); // 用世界坐标计算距离，缺点：数值较大，精度较低
```
这里推荐使用第一种，用视空间坐标计算，避免抖动的问题。  

### 3.4.6 计算当前片元叠加扩散颜色的透明度
通过时间参数和片元离圆心的距离计算透明度：
```glsl
float t = mod(u_time, u_duration) / u_duration;
float alpha = mod(distance / u_radius - t, 1.0);
```

### 3.4.7 设置最终片元颜色
判断计算出的distance距离是否小于扩散半径u_radius，小于的片元需要计算颜色混合，大于的片元直接使用颜色纹理中对应的颜色值。
```glsl
out_FragColor = texture(colorTexture, v_textureCoordinates); // 获取颜色纹理的颜色
if (distance < u_radius) {
  //带透明度的颜色和不带透明度颜色的融合
  out_FragColor = blend(vec4(1.0, 0.0, 0.0, alpha), out_FragColor);
}
```
`blend()`函数代码如下：
```glsl
vec4 blend(vec4 src, vec4 dst) {
  return src * src.a + dst * (1.0 - src.a);
}
```

# 4 总结
通过Cesium的后处理功能，我们可以实现各种复杂的视觉效果。本文详细介绍了如何在 Cesium 中使用 `PostProcessStage` 创建一个动态扩散圆圈效果的实现原理，特别是坐标转换和距离计算的关键技术。理解这些原理后，开发者可以进一步扩展实现更复杂的后处理效果，如不规则形状的扩散、多层效果叠加等，也可推广至雷达扫描、波纹弹射、扇形扫描等效果。

# 5 源码下载
如需源码可点击[下载链接](https://mbd.pub/o/bread/mbd-aJeTmp9v)，非常感谢您的支持。  
通过关注微信公众号《Web与GIS》，获取更多内容。  
![微信公众号](https://github.com/user-attachments/assets/225501b4-659b-44d7-99d3-947d7244b139)
  
---
如果该文章对您有所帮助，请您一定不要吝啬您的鼓励。点赞、评论、分享、收藏、打赏都是您对我的鼓励和支持。  
如果您有`GitHub`账号，还可以[关注我~](https://github.com/travelclover)  
最后，感谢大家的阅读，如有错误，还请各位批评指正。
