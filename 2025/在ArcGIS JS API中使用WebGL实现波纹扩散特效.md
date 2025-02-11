# 在ArcGIS JS API中使用WebGL实现波纹扩散特效  
在现代WebGIS开发中，ArcGIS JS API 是一个非常强大的工具，它允许开发者创建丰富的地理信息应用。结合WebGL技术，我们可以实现更加复杂和炫酷的可视化效果。本文将介绍如何使用ArcGIS JS API结合WebGL实现一个波纹扩散特效。  


<video width="640" height="360" controls>
  <source src="https://travelclover.github.io/img/2025/02/波纹扩散效果.mp4" type="video/mp4">
  Your browser does not support the video tag.
</video>

![波纹扩散效果](https://travelclover.github.io/img/2025/02/波纹扩散效果.gif)  

# 1 概述
波纹扩散特效是一种常见的视觉效果，通常用于表示某个点的扩散过程，比如地震波的传播、污染物的扩散等。本文将使用ArcGIS JS API创建的三维场景SceneView和WebGL技术来实现这一效果。通过自定义渲染节点[RenderNode](https://developers.arcgis.com/javascript/latest/api-reference/esri-views-3d-webgl-RenderNode.html)，我们可以在3D场景中生成波纹扩散动态效果。

# 2 准备工作
首先，我们需要引入ArcGIS JS API和WebGL相关的库。在HTML文件中，我们引入了ArcGIS JS API的CSS和JS文件，并使用了gl-matrix库来处理矩阵运算。
```html
<link rel="stylesheet" href="https://js.arcgis.com/4.31/esri/themes/light/main.css" />
<script src="https://js.arcgis.com/4.31/"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/gl-matrix/2.3.2/gl-matrix-min.js"></script>
```

# 3 创建地图场景
我们使用ArcGIS JS API创建一个三维场景SceneView，并设置底图为卫星影像。
```javascript
const view = new SceneView({
  container: 'viewDiv',
  map: new Map({
    basemap: 'satellite', // 影像底图
    ground: 'world-elevation', // 世界高程
  }),
});
```
创建一个点对象，用来表示波纹扩散中心的地理位置。
```javascript
const point = new Point({
  longitude: 103.29189644934428, // 经度
  latitude: 30.688679767265377, // 纬度
  z: 1260, // 高度
  spatialReference: view.spatialReference, // 空间参考
});
```

# 4 创建自定义渲染节点
ArcGIS JS API从4.29版本开始提供了[RenderNode](https://developers.arcgis.com/javascript/latest/api-reference/esri-views-3d-webgl-RenderNode.html)类，该类提供对 SceneView 渲染管道的底层访问，以创建自定义可视化和效果。渲染节点在渲染管道的不同阶段注入自定义 WebGL 代码以更改其输出。
## 4.1 创建渲染节点
为了实现波纹扩散特效，我们通过RenderNode类创建一个自定义的WaveRenderNode子类。这个类负责管理WebGL的渲染过程，包括着色器程序的创建、顶点数据的生成、深度纹理的创建等。
```javascript
const WaveRenderNode = RenderNode.createSubclass({
  // 类的构造函数
  constructor: function (props) {
    //定义渲染节点在渲染管线中的位置
    this.consumes = { required: ['transparent-color'] }; // 声明渲染需要来自引擎的哪些输入
    this.produces = 'transparent-color'; // 声明 render 函数生成的输出
    
    // 其它代码...
  },

  // 初始化函数
  initialize() {
    const gl = this.gl;
    this.viewPort = gl.getParameter(gl.VIEWPORT);
    const width = this.viewPort[2];
    const height = this.viewPort[3];

    this.initProgram(gl); // 初始化着色器程序
    this.getLocations(gl); // 获取着色器程序中的变量位置
    this.createBuffers(gl); // 创建顶点数组并绑定数据
    this.createFramebufferWithDepthTexture(gl, width, height); // 创建深度纹理和帧缓冲区
  },

  // 渲染函数
  render() {},
});
```
## 4.2 初始化着色器程序
我们使用顶点着色器和片元着色器来实现波纹效果。顶点着色器负责将顶点位置映射到屏幕坐标系，片元着色器负责计算波纹每个像素的颜色和透明度。

### 4.2.1 顶点着色器
将顶点位置映射到屏幕坐标系，确保波纹能够正确地显示在屏幕上。
```glsl
#version 300 es
in vec2 a_Position; // 输入的顶点位置（二维坐标）
uniform vec4 u_ScreenRatio_ScreenOffset; // 屏幕比例和偏移量（四维向量）

void main() {
  // 计算顶点在裁剪空间中的最终位置
  gl_Position = vec4(
    a_Position * u_ScreenRatio_ScreenOffset.zw + u_ScreenRatio_ScreenOffset.xy,
    1.0,
    1.0
  );
}
```
`u_ScreenRatio_ScreenOffset`是一个四维向量，存储了4个参数，`x`、`y`分别代表x和y方向的偏移量，`z`、`w`分别代表x和y方向的缩放比例。这些值需要在每一次渲染中通过波纹中心点坐标、相机的视图矩阵和投影矩阵计算得出。

### 4.2.2 片元着色器
在片元着色器中需要计算得出每个片元的透明度值。首先根据片元坐标计算出屏幕uv坐标，通过uv坐标采样深度纹理，获取深度信息后结合变换矩阵的逆矩阵计算出在渲染坐标系中相对于波纹中心点的三维坐标，然后求距离，判断是否超过渲染设置的波纹扩散最大距离，超过则舍弃片元。最后通过距离值加上随时间变化的偏移值，计算得出最终的每个片元透明度值。

```glsl
#version 300 es
precision mediump float; // 设置浮点数精度

uniform float u_distance; // 波纹扩散最大距离
uniform float u_offset; // 波纹扩散偏移量
uniform vec2 u_1_WidthAndHeight; // 屏幕宽高的倒数
uniform mat4 u_InvertMat; // 逆矩阵
uniform sampler2D u_DepthTex; // 深度纹理

out lowp vec4 FragColor;

void main() {
  vec2 uv_screen = gl_FragCoord.xy * u_1_WidthAndHeight; // 计算屏幕UV
  vec4 pos = vec4(uv_screen * 2.0 - 1.0, texture(u_DepthTex, uv_screen).r * 2.0 - 1.0, 1.0); // 计算当前片元在裁剪空间中的位置

  pos = u_InvertMat * pos; // 乘以逆矩阵，计算出渲染坐标系中的三维坐标
  pos.xyz /= pos.w; // 将齐次坐标转换为标准的三维坐标

  float f_distance = length(pos.xy); // 计算距离
  if (f_distance > u_distance) {
    discard;
  }

  float percent = f_distance / u_distance; // 计算当前片元位置相对于最大扩散距离的比例
  float alpha = mod(percent + 1.0 - u_offset, 1.0); // 计算片元透明度

  FragColor = vec4(1.0, 0.0, 0.0, alpha); // 输出片元颜色
}
```

### 4.2.3 创建着色器和链接着色器程序
通过以下代码编译着色器代码并创建和链接 WebGL 着色器程序。
```javascript
// 创建着色器
function createShader(gl, src, type) {
  const shader = gl.createShader(type);
  gl.shaderSource(shader, src);
  gl.compileShader(shader);
  return shader;
}
```
```javascript
// 创建程序
function createProgram(gl, vsSource, fsSource) {
  const program = gl.createProgram();
  if (!program) {
    console.error('Failed to create program');
  }
  const vertexShader = createShader(
    gl,
    vsSource,
    gl.VERTEX_SHADER
  );
  const fragmentShader = createShader(
    gl,
    fsSource,
    gl.FRAGMENT_SHADER
  );
  gl.attachShader(program, vertexShader);
  gl.attachShader(program, fragmentShader);
  gl.linkProgram(program);

  // 获取链接状态
  const success = gl.getProgramParameter(program, gl.LINK_STATUS);
  if (!success) {
    console.error(`Failed to link program:
      error ${gl.getError()},
      info log: ${gl.getProgramInfoLog(program)},
      vertex: ${gl.getShaderParameter(
        vertexShader,
        gl.COMPILE_STATUS
      )},
      fragment: ${gl.getShaderParameter(
        fragmentShader,
        gl.COMPILE_STATUS
      )}
      vertex info log: ${gl.getShaderInfoLog(vertexShader)},
      fragment info log: ${gl.getShaderInfoLog(fragmentShader)}
    `);
  }
  return program;
}
```

## 4.3 获取着色器程序中的变量位置
[gl.getUniformLocation](https://developer.mozilla.org/en-US/docs/Web/API/WebGLRenderingContext/getUniformLocation) 和 [gl.getAttribLocation](https://developer.mozilla.org/zh-CN/docs/Web/API/WebGLRenderingContext/getAttribLocation) 这两个方法用于获取着色器程序中变量的位置（location），这是与着色器程序通信的必要步骤。  
`gl.getUniformLocation`用于获取着色器中 uniform 变量的位置，uniform 变量是着色器中的全局变量，在一次绘制过程中保持不变。  
`gl.getAttribLocation`用于获取着色器中 attribute 变量的位置，attribute 变量用于传递顶点数据，每个顶点都可以不同。  
```javascript
function getLocations(gl) {
  this.a_position = gl.getAttribLocation(this.program, 'a_Position');
  
  this.u_ScreenRatio_ScreenOffset = gl.getUniformLocation(this.program, 'u_ScreenRatio_ScreenOffset');
  this.u_1_WidthAndHeight = gl.getUniformLocation(this.program, 'u_1_WidthAndHeight');
  this.u_distance = gl.getUniformLocation(this.program, 'u_distance');
  this.u_InvertMat = gl.getUniformLocation(this.program,'u_InvertMat');
  this.u_offset = gl.getUniformLocation(this.program, 'u_offset');
  this.u_DepthTex = gl.getUniformLocation(this.program, 'u_DepthTex');
}
```

## 4.4 创建顶点数组对象
创建顶点数组对象（VAO）的主要作用是管理和组织顶点数据的状态和配置。将顶点属性配置（顶点缓冲区、属性指针等）打包在一起，避免在渲染时重复设置这些状态，提高渲染性能。  
首先，我们需要计算出圆的顶点数据。  

### 4.4.1 创建圆的顶点数据
如下图所示，我们需要将圆分成 16 个等份，每个部分创建一个三角形（由圆心和圆周上的两个点组成），生成用于 WebGL 绘制的顶点数组，这些顶点最终会形成一个完整的圆形，用于显示波纹效果。

![波纹扩散效果](https://travelclover.github.io/img/2025/02/圆顶点示意图.jpg)

```javascript
const circleVertexCount = this.circleVertexCount; // 圆的顶点数量
const angle = (2 * Math.PI) / circleVertexCount; // 计算每个顶点的角度增量
const circleRadius = 1 / Math.cos(angle * 0.5); // 计算半径
this.drawDistance = this.distance * circleRadius; // 计算圆的绘制距离
const circleVertext = [];
for (let i = 0; i < circleVertexCount; i++) {
  const nextIndex = i + 1;
  const currentSin = Math.sin(angle * i); // 计算当前顶点的sin值
  const nextSin = Math.sin(angle * nextIndex); // 计算下个顶点的sin值
  const currentCos = Math.cos(angle * i); // 计算当前顶点的cos值
  const nextCos = Math.cos(angle * nextIndex); // 计算下个顶点的cos值
  // 构成圆的三角面的三个顶点数据
  circleVertext.push(
    0,
    0,
    currentCos * circleRadius,
    currentSin * circleRadius,
    nextCos * circleRadius,
    nextSin * circleRadius
  );
}
```
以上代码整个过程实际上是在构建一个由多个三角形组成的圆形，每个三角形都以圆心为顶点之一，这种构建方式有利于实现径向的波纹扩散效果。  

### 4.4.2 设置顶点位置缓冲区
有了圆的顶点数据，然后需要创建缓冲区，将顶点数据传入到缓冲区中，并设置顶点属性指针。
```javascript
function createBuffer(gl, target, data, location, size) {
  const buffer = gl.createBuffer();
  gl.bindBuffer(target, buffer);
  gl.bufferData(target, data, gl.STATIC_DRAW);
  gl.enableVertexAttribArray(location);
  gl.vertexAttribPointer(location, size, gl.FLOAT, false, 0, 0);
  return buffer;
}

const positionBuffer = createBuffer(
  gl,
  gl.ARRAY_BUFFER,
  new Float32Array(circleVertext),
  this.a_position,
  2
);
```

## 4.5 创建深度纹理和帧缓冲区
在片元着色器中需要用到深度数据来反算三维坐标，深度数据需要通过纹理采样来获取，所以需要创建深度纹理并附加到帧缓冲区，然后在每一帧渲染时，将深度数据写入到帧缓冲区中。
```javascript
this.framebuffer = gl.createFramebuffer(); // 创建帧缓冲区
gl.bindFramebuffer(gl.FRAMEBUFFER, this.framebuffer);

this.depthTexture = gl.createTexture(); // 创建深度纹理
gl.bindTexture(gl.TEXTURE_2D, this.depthTexture);
gl.texImage2D(gl.TEXTURE_2D, 0, gl.DEPTH24_STENCIL8, width, height, 0, gl.DEPTH_STENCIL, gl.UNSIGNED_INT_24_8, null);

// 设置纹理参数
gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MAG_FILTER, gl.NEAREST);
gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MIN_FILTER, gl.NEAREST);
gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_S, gl.CLAMP_TO_EDGE);
gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_T, gl.CLAMP_TO_EDGE);

// 将深度纹理附加到帧缓冲区
gl.framebufferTexture2D(gl.DRAW_FRAMEBUFFER, gl.DEPTH_STENCIL_ATTACHMENT, gl.TEXTURE_2D, this.depthTexture, 0);

// 解绑帧缓冲区
gl.bindFramebuffer(gl.FRAMEBUFFER, null);
```

## 4.6 设置相关数据
在类的构造方法中设置波纹扩散效果需要用到的相关参数，例如动画时间、扩散距离等。
```javascript
this.duration = 1800; // 波纹扩散时间, 毫秒
this.distance = 4000; // 波纹最大距离, 单位米
this.currentTime = performance.now(); // 时间，毫秒
this.circleVertexCount = 16; // 圆的顶点数量，点越多，圆就越平滑
```
使用ArcGIS JS API [webgl.toRenderCoordinates()](https://developers.arcgis.com/javascript/latest/api-reference/esri-views-3d-webgl.html#toRenderCoordinates) 方法将波纹中心的地理坐标转换为渲染空间的三维坐标。
```javascript
const localOriginSR = this.view.spatialReference; // arcgis js api创建的三维场景的参考坐标系
const coord = [point.x, point.y, point.z]; // 点坐标 [经度, 纬度, 高程]
this.localOriginRender = webgl.toRenderCoordinates(
  this.view,
  coord,
  0,
  localOriginSR,
  new Float32Array(3),
  0,
  1
);
```
使用ArcGIS JS API [webgl.renderCoordinateTransformAt()](https://developers.arcgis.com/javascript/latest/api-reference/esri-views-3d-webgl.html#renderCoordinateTransformAt) 方法计算出从局部笛卡尔坐标系到虚拟世界坐标系的变换矩阵，再计算出变换矩阵的逆矩阵，反算坐标时会用到。
```javascript
// 计算从局部笛卡尔坐标到虚拟世界坐标系的变换矩阵
const transAt = glMatrix.mat4.create();
webgl.renderCoordinateTransformAt(
  view,
  coord,
  this.view.spatialReference,
  transAt
);

// 计算变换矩阵的逆矩阵
glMatrix.mat4.invert(this.transAt_invert, transAt);
```

## 4.7 设置渲染函数 render
RenderNode 类中的 render 方法，在每一次渲染帧时，都会调用该函数，用于执行自定义渲染逻辑。  

### 4.7.1 保存 webgl 当前状态
在修改相关 webgl 状态之前，应当保存一下当前的 webgl 状态，在执行完我们自定义渲染后，再还原回开始的状态，避免造成 SceneView 本身场景绘制产生错误。  
```javascript
const vbo = gl.getParameter(gl.VERTEX_ARRAY_BINDING); // 获取当前绑定的顶点数组对象
const dBlend = gl.getParameter(gl.BLEND); // 获取是否启用混合的状态
const dDepthMask = gl.getParameter(gl.DEPTH_WRITEMASK); // 获取当前 depthMask 状态
const dSrcRGB = gl.getParameter(gl.BLEND_SRC_RGB); // 获取当前的源 RGB 混合因子
const dDstRGB = gl.getParameter(gl.BLEND_DST_RGB); // 获取当前的目标 RGB 混合因子
const dStencil = gl.getParameter(gl.STENCIL_TEST); // 获取当前的模板测试状态，返回true 或 false
const dDepthText = gl.getParameter(gl.DEPTH_TEST); // 获取深度测试参数
```

### 4.7.2 修改 webgl 状态
修改相关状态，以符合我们的绘制效果。
```javascript
gl.depthMask(false); // 关闭深度写入
gl.disable(gl.STENCIL_TEST); // 关闭模板测试
gl.bindVertexArray(null); // 清除当前的 顶点数组对象（VAO） 绑定
gl.enable(gl.BLEND); // 启用混合状态
gl.blendFunc(gl.SRC_ALPHA, gl.ONE_MINUS_SRC_ALPHA); // 设置混合因子
gl.disable(gl.DEPTH_TEST); // 禁用深度测试
```

### 4.7.3 更新深度纹理尺寸
当用户拖拽三维场景时，webgl 的视口参数会发生变化，可能是 ArcGIS JS API 本身在`window.devicePixelRatio`大于1设备上做的性能优化。所以在视口参数发生变化时，需要更新纹理尺寸，否则会导致纹理采样数据错误。  
```javascript
const viewPort = gl.getParameter(gl.VIEWPORT); // 获取当前的视口参数
if (
  viewPort[2] !== this.viewPort[2] ||
  viewPort[3] !== this.viewPort[3]
) {
  this.viewPort = viewPort;
  const dTexture = gl.getParameter(gl.TEXTURE_BINDING_2D); // 获取当前绑定的2D纹理对象
  gl.bindTexture(gl.TEXTURE_2D, this.depthTexture); // 将指定的纹理对象绑定到2D纹理目标上
  // 为当前绑定的纹理对象分配内存
  gl.texImage2D(
    gl.TEXTURE_2D,
    0,
    gl.DEPTH24_STENCIL8,
    viewPort[2],
    viewPort[3],
    0,
    gl.DEPTH_STENCIL,
    gl.UNSIGNED_INT_24_8,
    null
  );
  gl.bindTexture(gl.TEXTURE_2D, dTexture); // 恢复之前的绑定
}
```

### 4.7.4 更新深度纹理
render函数传入的inputs参数中包含深度帧缓冲数据，需要将它复制到我们创建的深度纹理帧缓冲数据中。
```javascript
const managedFBO = inputs[0];
const fbo_gl = managedFBO.fbo.glName;
const dReadFbo = gl.getParameter(gl.READ_FRAMEBUFFER_BINDING); // 获取当前绑定的读取帧缓冲对象
const dDrawFbo = gl.getParameter(gl.DRAW_FRAMEBUFFER_BINDING); // 获取当前绑定的绘制帧缓冲对象

gl.bindFramebuffer(gl.READ_FRAMEBUFFER, fbo_gl); // 绑定读取的帧缓冲区对象
gl.bindFramebuffer(gl.DRAW_FRAMEBUFFER, this.framebuffer); // 绑定绘制的帧缓冲区对象
// 将像素块从读取的帧缓冲区传输到绘制帧缓冲区
gl.blitFramebuffer(0, 0, this.viewPort[2], this.viewPort[3], 0, 0, this.viewPort[2], this.viewPort[3], gl.DEPTH_BUFFER_BIT, gl.NEAREST);

gl.bindFramebuffer(gl.READ_FRAMEBUFFER, dReadFbo); // 恢复读取的帧缓冲区
gl.bindFramebuffer(gl.DRAW_FRAMEBUFFER, dDrawFbo); // 恢复绘制的帧缓冲区
```

### 4.7.5 启用着色器程序
使用以下代码启用着色器程序，并绑定顶点数组对象，以及激活和绑定纹理。
```javascript
gl.useProgram(this.program); // 使用着色器程序
gl.bindVertexArray(this.vao); // 绑定顶点数组对象
gl.activeTexture(gl.TEXTURE0); // 激活纹理单元0
gl.bindTexture(gl.TEXTURE_2D, this.depthTexture); // 绑定纹理
```

### 4.7.6 设置 uniform 数据
通过[glMatrix](https://glmatrix.net/)库的相关矩阵和向量方法计算着色器中需要用到的数据。
```javascript
// 将视图矩阵与坐标点相乘，计算坐标点在视图坐标系下的位置
glMatrix.vec3.transformMat4(
  this.tempVec3_1,
  this.localOriginRender,
  this.viewMat
);
this.tempVec3_2[0] = this.tempVec3_1[0] + this.drawDistance;
this.tempVec3_2[1] = this.tempVec3_1[1] + this.drawDistance;
this.tempVec3_2[2] = this.tempVec3_1[2];

// 计算坐标点在屏幕坐标系下的位置
glMatrix.vec3.transformMat4(this.tempVec3_1, this.tempVec3_1, this.projMat);
// 计算加上drawDistance后的坐标点在屏幕坐标系下的位置
glMatrix.vec3.transformMat4(this.tempVec3_2, this.tempVec3_2, this.projMat);

this.tempVec4_1[0] = this.tempVec3_1[0]; // 圆心点在屏幕坐标系下的x坐标
this.tempVec4_1[1] = this.tempVec3_1[1]; // 圆心点在屏幕坐标系下的y坐标
this.tempVec4_1[2] = this.tempVec3_2[0] - this.tempVec3_1[0]; // 屏幕坐标系下圆边缘到圆心点位置的x坐标差，表示x方向的缩放
this.tempVec4_1[3] = this.tempVec3_2[1] - this.tempVec3_1[1]; // 屏幕坐标系下圆边缘到圆心点位置的y坐标差，表示y方向的缩放

// 将视图投影矩阵的逆矩阵和变换矩阵的逆矩阵相乘
glMatrix.mat4.multiply(this.tempMat4, this.transAt_invert, this.viewProjMat_invert);

```
使用[`uniform[1234][fi][v]()`](https://developer.mozilla.org/zh-CN/docs/Web/API/WebGLRenderingContext/uniform) 和 [`uniformMatrix[234]fv()
`](https://developer.mozilla.org/zh-CN/docs/Web/API/WebGLRenderingContext/uniformMatrix) 方法传入uniform数据值。
```javascript

gl.uniform4fv(this.u_ScreenRatio_ScreenOffset, this.tempVec4_1); // 设置 u_ScreenRatio_ScreenOffset 数据
gl.uniform1f(this.u_distance, this.distance); // 设置 u_distance的值
gl.uniform1f(this.u_offset, (performance.now() % this.duration) / this.duration);
gl.uniform2fv(this.u_1_WidthAndHeight, this._1_WidthAndHeight); // 设置 u_1_WidthAndHeight的值
gl.uniformMatrix4fv(this.u_InvertMat, false, this.tempMat4);
gl.uniform1i(this.u_DepthTex, 0); // 将纹理单元0与着色器中的u_DepthTex关联
```

### 4.7.7 绘制图形
设置完所有数据后，调用绘制方法，绘制所有三角形面。
```javascript
gl.drawArrays(gl.TRIANGLES, 0, this.circleVertexCount * 3);
```

### 4.7.8 恢复 webgl 状态
完成绘制后，还需恢复 webgl 状态，避免对其它渲染造成影响。
```javascript
if (dBlend) {
  gl.enable(gl.BLEND);
} else {
  gl.disable(gl.BLEND);
}
gl.depthMask(dDepthMask);
gl.blendFunc(dSrcRGB, dDstRGB);
gl.bindVertexArray(vbo);
if (dStencil) gl.enable(gl.STENCIL_TEST);
if (dDepthText) gl.enable(gl.DEPTH_TEST);
```

### 4.7.9 持续渲染
因为我们需要实现的波纹扩散特效，是一个持续的动画效果，所以绘制了一帧后，还需要再次调用绘制方法[requestRender()](https://developers.arcgis.com/javascript/latest/api-reference/esri-views-3d-webgl-RenderNode.html#requestRender)，实现连续的扩散动画。  
```javascript
this.requestRender(); // 请求渲染
```
# 5 如何使用
创建好自定义渲染节点后，使用非常简单，只需new一个实例即可。
```javascript 
// 创建实例
const waveRenderNode = new WaveRenderNode({ view });
```

# 6 解决精度问题
完成上述所有代码后，地图上就已经有了波纹扩散效果，但放大地图后会发现绘制的圆会出现抖动效果，按理说我们的矩阵计算没有放在WebGL中计算，就不会出现单精度浮点数精度不足导致漂移和抖动的问题。其实原因是我们使用的 `glMatrix` 库默认就是单精度的，需要通过以下代码设置为双精度。
```javascript
glMatrix.glMatrix.setMatrixArrayType(Float64Array);
```
上面代码设置矩阵数组类型为Float64Array（双精度浮点数），提供更高的数值精度，避免出现漂移和抖动效果。

# 7 总结
通过结合ArcGIS JS API和WebGL技术，我们成功实现了一个波纹扩散特效。这个特效可以用于表示各种扩散过程，如地震波、污染物扩散等。通过自定义RenderNode，我们可以灵活地控制渲染过程，实现更加复杂的效果。


# 完整代码
如需查看示例效果可点击下载[在ArcGIS JS API中使用WebGL实现波纹扩散特效（源码+详细注释）.zip](https://mbd.pub/o/bread/Z56am5xv
)。代码中包含详细的注释，方便大家的理解。  

---
如果该文章对您有所帮助，请您一定不要吝啬您的鼓励。点赞、评论、分享、收藏、打赏都是您对我的鼓励和支持。  
如果您有`GitHub`账号，还可以[关注我~](https://github.com/travelclover)  
最后，感谢大家的阅读，如有错误，还请各位批评指正。
