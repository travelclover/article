# 不常用但有用的HTML标签
作为一名前端开发人员，不管是高级前端还是入门级前端，都应该至少掌握HTML、CSS和JavaScript这三门技术。其中HTML作为页面的骨架，是最不可缺少的，平常大家用于构建页面的标签应该不多，特别是用了第三方UI组件库后，基本上应该只有`<div>`、`<p>`、`<span>`、`<ul>`会被用到了。今天就来给大家总结下平常我们不常用但是有用的HTML标签。

## 一、&lt;bdo&gt;双向文本代替元素
`<bdo>`元素用于改写文本的方向性，使元素内的文本以不同的方向呈现出来。

### 属性
元素上有一个`"dir"`属性，用来表示元素中文本的方向。该属性允许使用的值有：  
`ltr`（left to right）指示文本应从左到右的方向。  
`rtl`（right to left）指示文本应从右到左的方向。  

### 示例
```html
<p><bdo dir="ltr">这是一个从左向右的文本！</bdo></p>
<p><bdo dir="rtl">这是一个从右向左的文本！</bdo></p>
```
示例效果  
![bdo示例效果](https://travelclover.github.io/img/2022/06/bdo示例结果.png)  

## 二、&lt;bdi&gt;双向隔离元素
`<bdi>`元素的作用是从周围文字方向设置中隔离出来，允许您设置一段文本，使其脱离其父元素的文本方向设置。  

### 示例
```html
<p><bdo dir="rtl">这是一个从右向左的文本！<bdi>(括号内文本为从左向右)</bdi></bdo></p>
```
示例效果  
![bdi示例效果](https://travelclover.github.io/img/2022/06/bdi示例结果.png)  

## 三、&lt;details&gt;详细信息展现元素
`<details>`元素允许用户创建一个可展开折叠的元件，让一段文字或标题包含一些隐藏的信息。

### 属性
`<details>`元素上包含一个`"open"`属性，类型为布尔值，用来表示详细内容是否可见，默认值为`false`。

### 特殊子元素
`<summary>`是`<details>`元素的特有子元素，定义一个可见的标题信息或者摘要信息。

### 事件
除了HTML元素支持的常见事件外，&lt;details&gt;元素还支持`toggle`切换事件。

### 示例
```html
<details>
  <summary>这是示例标题或摘要内容</summary>
  这是示例的详情内容。默认不显示该内容，点击 summary 元素后显示该内容。
</details>
```
示例效果  
![details元素示例](https://travelclover.github.io/img/2022/06/details元素示例-min.gif)  

## 四、&lt;map&gt;图像映射元素
`<map>`元素用来定义一个图像映射。图像映射是指把一幅图像划分为多个区域（即热点区域），每个热点区域对应一个超级链接，当用户点击热点区域，会自动跳转到预先设定好的链接地址。该元素需要配合`<area>`和`<img>`元素一起使用。

### 属性
`<map>`元素上必须存在一个`"name"`属性，且该属性不能为空字符串，且同一文档中不能和另一个`<map>`元素上的`name`属性值相同，如果还指定了`id`属性，则`id`属性值和`name`属性值必须一致。  
**注意：&lt;img&gt;元素上的usemap属性值必须为`"#"`加上&lt;map&gt;元素的name属性值，这样才能建立图像与图像映射之间的关联。**

## 五、&lt;area&gt;图像映射区域元素
`<area>`元素定义了图像映射中的一个区域，该区域为一个几何形状，可关联一个超级链接地址，当点击该区域时，会触发页面跳转。该元素只可在`<map>`元素中使用。

### 属性
`<area>`元素常用的属性有以下几个：
| 属性字段 | 说明      |
|:--------| :------------- |
| shape | 映射区域的形状，取值有"rect"、"circle"、"poly"、"default |
| coords | 热点区域的坐标，每个点的坐标参考点为图像的左上角顶点 |
| href | 区域的超链接目标。它的值是一个有效的 URL。可以省略。 |
| target | 在何处打开href属性指定的目标URL，同&lt;a&gt;元素的target属性值一致。 |

当 shape 属性取不同值时，coords属性值的格式及含义如下：  
| shape取值 | coords值 | 描述 |
| :--------| :------------- | :------------- |
| rect | x1, y1, x2, y2 | 热点区域的形状为矩形，矩形的左上角顶点坐标为（x1, y1），右下角顶点坐标为（x2, y2） |
| circle | x, y, r | 热点区域的形状为圆形，圆心坐标为（x1,y1），半径为r |
| poly | x1, y1, x2, y2, ... | 热点区域的形状为多边形，各顶点坐标依次为（x1, y1）、（x2, y2）、… |
| default | 无 | 如果shape设置为默认值，则不会使用coords属性 |

### 示例
```html
<map name="mapExample">
  <area shape="rect" coords="0,0,75,75" href="rect.html">
  <area shape="circle" coords="275,75,75" href="circle.html">
  <area shape="poly" coords="100,0,200,0,100,75" href="poly.html">
</map>
<img usemap="#mapExample" src="图片地址url">
```

## 六、&lt;dialog&gt;对话框元素
`<dialog>`元素用于展示一个交互式模态对话框。

### 属性
`<dialog>`元素除了全局属性外，还支持一个`"open"`属性，类型为布尔值，即含有`open`属性就显示对话框，否则隐藏。但更多的时候，我们可以获取到`dialog`对象，通过事件的形式控制弹窗的显示和隐藏。

### 事件
`<dialog>`元素支持一下事件：
| 事件 | 描述 |
| :------ | :------ |
| close() | 关闭弹窗 |
| show() | 显示弹窗 |
| showModal() | 显示弹窗 |

> `show()`和`showModal()`的区别是：用showModal打开的对话框是上下左右居中的，对话框的背景还有默认的蒙层，可以通过按键盘上的`ESC`按键进行关闭弹窗操作。

### 示例
```html
<dialog open>
  <p>这是 dialog 元素！</p>
  <form method="dialog">
    <button>关闭</button>
  </form>
</dialog>
```

## 七、&lt;output&gt;输出元素
`<output>`元素用来展现计算或用户操作的结果，通常和`form`表单一起使用。

### 属性
| 属性字段 | 描述 |
| :------ | :------ |
| for | 定义与输出结果相关的一个或多个元素的id，以空格分开 |
| form | 与输出相关联的&lt;form&gt;元素。此属性的值必须是同一文档中&lt;form&gt;元素的id |
| name | 定义对象的唯一名称（表单提交时使用） |

### 示例
```html
<form oninput="result.value=parseInt(number1.value)+parseInt(number2.value)">
  <input type="number" id="number1" value="50" /> +
  <input type="number" id="number2" value="10" /> =
  <output name="result" for="number1 number2">60</output>
</form>
```
示例效果  
![output元素示例效果](https://travelclover.github.io/img/2022/06/output元素示例-min.gif)

## 八、&lt;meter&gt;度量元素
`<meter>`元素用来表示已知范围，且可度量的等级标量或分数值。

### 属性
| 属性字段 | 描述 |
| :------ | :------ |
| value | 标量的实际测量值，如果不指定，默认为0 |
| min | 标量的最小值，该值必须小于属性max的值，如果未指定，则为0 |
| max | 标量的最大值，该值必须大于属性min的值，如果未指定，则为1 |
| low | 标量最小值的上限，该值必须大于属性min的值，且必须小于属性high和属性max的值 |
| high | 标量最大值的下限，该值必须大于属性min和属性low的值，且必须小于属性max的值 |
| optimum | 最佳数值，用来规定标量的最优值的位置 |

通过属性`min`、`low`、`high`和`max`将一个范围分成了3个区间，min和low构成一个低区间，low和high构成一个中间区间，high和max构成一个高区间，如下图所示：  
![meter元素分区示意图](https://travelclover.github.io/img/2022/06/meter元素分区示意图.png)  
`optimum`代表最佳值，该值落在哪个区间就代表哪个区间的值为最优值。比如将考试成绩划分三个区间，[0 - 60]为`low`，[60 - 90]为`medium`，[90 - 100]为`high`，`optimum`最佳值设为100，该值落在high区间，就代表最佳区间。当`value`值落在最佳区间内时，会渲染成绿色。靠近最佳区间的为中间区间，当`value`值落在此区间时会渲染成黄色。离最佳区间较远的是最差区间，当`value`值落在此区间时会渲染成红色。

### 示例
#### 示例1
```html
<p>将考试成绩划分为[0 - 60]、[60 - 90]和[90 - 100]三个区间，最优区间为[90 - 100]</p>
<label for="scoreLow">成绩59分时：</label>
<meter id="scoreLow" min="0" max="100" low="60" high="90" optimum="100" value="59">
  at 59/100
</meter>
<label for="scoreMedium">成绩85分时：</label>
<meter id="scoreMedium" min="0" max="100" low="60" high="90" optimum="100" value="85">
  at 85/100
</meter>
<label for="scoreMedium">成绩90分时：</label>
<meter id="scoreMedium" min="0" max="100" low="60" high="90" optimum="100" value="90">
  at 90/100
</meter>
```
示例1效果  
![meter元素示例效果](https://travelclover.github.io/img/2022/06/meter示例结果.png)
  
#### 示例2
```html
<p>将环境温度划分为[0 - 24]、[24 - 28]和[28 - 45]三个区间，当人处在中间温度时感觉最舒适。</p>
<label for="tempLow">温度20°时：</label>
<meter id="tempLow" min="0" max="45" low="24" high="28" optimum="26" value="20">20°</meter>
<label for="tempMedium">温度26°时：</label>
<meter id="tempMedium" min="0" max="45" low="24" high="28" optimum="26" value="26">26°</meter>
<label for="tempMedium">温度33°时：</label>
<meter id="tempMedium" min="0" max="45" low="24" high="28" optimum="26" value="33">33°</meter>
```
示例2效果  
![meter元素示例2效果](https://travelclover.github.io/img/2022/06/meter示例2结果.png)
  
#### 示例3
```html
<p>将犯错误数量划分为[0 - 5]、[5 - 10]和[10 - 20]三个区间，当所犯错误数量少于5个时为最优，犯错数量在10到20个为最差。</p>
<label for="errLow">错误数量为4时：</label>
<meter id="errLow" min="0" max="20" low="5" high="10" optimum="0" value="4">4</meter>
<label for="errMedium">错误数量为7时：</label>
<meter id="errMedium" min="0" max="20" low="5" high="10" optimum="0" value="7">7</meter>
<label for="errMedium">错误数量为20时：</label>
<meter id="errMedium" min="0" max="20" low="5" high="10" optimum="0" value="20">20</meter>
```
示例3效果  
![meter元素示例3效果](https://travelclover.github.io/img/2022/06/meter示例3结果.png)

> **注意：** 我们不应该将`<meter>`用于进度指示器相关场景，进度指示器场景下我们可以使用`<progress>`进度指示器元素。

## 九、&lt;progress&gt;进度指示器元素
`<progress>`元素用来表示任务或者进程的进度情况，通常显示为进度条。可以用于文件上传的进度显示，文件下载的进度显示等场景，也可以作为一种loading的加载状态条使用。需要注意的是`<progress>`元素不应用来表示度量衡，比如表示磁盘空间的使用情况，如需表示度量衡，我们应使用`<meter>`元素。

### 属性
| 属性字段 | 描述 |
| :------ | :------ |
| max | 用来定义进度条的最大值。如果此属性存在，则值必须是大于0的有效浮点数。该属性默认值为1 |
| value | 用来定义进度条完成的进度值。必须是介于0和属性max值之间的有效浮点数。如果没有属性max值，则该值介于0到1之间。如果没有此属性值，则进度条是不确定的，表明任务正在进行中，可以用作loading状态的表示 |

> **注意：** 与`<meter>`元素不同，`<progress>`元素最小值始终为0，并且不允许使用min属性。

### 示例
#### 示例1 
```html
<p>示例1 显示百分比为70的进度条</p>
<progress max="100" value="70">70%</progress>
```
示例1效果  
![progress元素示例效果](https://travelclover.github.io/img/2022/06/progress元素示例结果.png)

#### 示例2
```html
<p>示例2 作为一种loading的加载状态条使用</p>
<progress max="100"></progress>
```
示例2效果  
![progress元素示例2效果](https://travelclover.github.io/img/2022/06/progress元素示例2结果.gif)

## 十、&lt;datalist&gt;数据列表元素
`<datalist>`元素包含一组`<option>`元素，用于在其他控件中展示一些可选数据的列表。配合`<input>`元素的不同类型控件使用，展示效果也不太一样。

### 示例
#### 示例1 与文本输入框配合使用  
当用户单击或双击type值为"text"、"search"、"url"、"tel"、"email"和"number"的`<input>`控件时，推荐值将显示在下拉菜单中。当输入框里输入信息时，根据输入的内容，自动显示一个提示下拉列表供用户选择。  
```html
<label for="hobby">输入您的爱好:</label>
<input type="text" list="hobbies" id="hobby" name="hobby" />
<datalist id="hobbies">
  <option value="抽烟">抽烟</option>
  <option value="喝酒">喝酒</option>
  <option value="烫头">烫头</option>
  <option value="认真工作">认真工作</option>
  <option value="拼命工作">拼命工作</option>
</datalist>
```
示例1效果  
![datalist元素示例1效果](https://travelclover.github.io/img/2022/06/datalist元素示例1结果.gif)
  
#### 示例2 与日期和时间控件配合使用  
```html
<label for="time">输入时间:</label>
<input type="time" list="popularHours" id="time">
<datalist id="popularHours">
  <option value="12:00">
  <option value="13:00">
  <option value="14:00">
</datalist>
```
示例2效果  
![datalist元素示例2效果](https://travelclover.github.io/img/2022/06/datalist元素示例2结果.gif)
  
#### 示例3 与范围选择控件配合使用  
范围类型中的推荐值列表将显示为用户可以轻松选择到的刻度标记。  
```html
<label for="range">请选择:</label>
<input type="range" list="tickmarks" id="range" />
<datalist id="tickmarks">
  <option value="0">0</option>
  <option value="10">10</option>
  <option value="20">20</option>
  <option value="30">30</option>
  <option value="80">80</option>
</datalist>
```
示例3效果  
![datalist元素示例3效果](https://travelclover.github.io/img/2022/06/datalist元素示例3结果.gif)
  
#### 示例4 与颜色选择控件配合使用  
颜色类型可以在浏览器提供的界面中显示预定义的颜色。  
```html
<label for="color">请选择颜色:</label>
<input type="color" list="colors" id="color">
<datalist id="colors">
  <option value="#ff0000">
  <option value="#00ff00">
  <option value="#0000ff">
  <option value="#ffff00">
  <option value="#00ffff">
</datalist>
```
示例4效果  
![datalist元素示例4效果](https://travelclover.github.io/img/2022/06/datalist元素示例4结果.gif)

# 总结  
今天介绍了10个不常用的HTML元素，虽然不常用，但是部分元素还是能适用于很多场景下。究其不常用原因，我猜想应该有这三点：一是现在前端有很多UI组件库可供开发人员使用；二是部分元素在不同浏览器下效果不一致，有些能用有些不能用；三是开发人员能手写出相同效果的组件。最后，虽然很多HTML元素不常用，但大家多了解一点是一点，毕竟知识永远不嫌多~
