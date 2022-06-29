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
示例结果  
![bdo示例结果](https://travelclover.github.io/img/2022/06/bdo示例结果.png)  

## 二、&lt;bdi&gt;双向隔离元素
`<bdi>`元素的作用是从周围文字方向设置中隔离出来，允许您设置一段文本，使其脱离其父元素的文本方向设置。  

### 示例
```html
<p><bdo dir="rtl">这是一个从右向左的文本！<bdi>(括号内文本为从左向右)</bdi></bdo></p>
```
示例结果  
![bdi示例结果](https://travelclover.github.io/img/2022/06/bdi示例结果.png)  

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
示例结果  
![details元素示例结果](https://travelclover.github.io/img/2022/06/details标签示例结果.png)  

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
|:--------| :------------- | :------------- |
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
