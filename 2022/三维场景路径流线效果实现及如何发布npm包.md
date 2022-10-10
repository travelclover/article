# 三维场景路径流线效果实现及如何发布npm包

## 1 路径流线效果实现
![路径流线效果](https://travelclover.github.io/img/2022/10/路径流线效果.gif)

### 1.1 实现原理和方式
路径流线效果总体上分为三种方式：1.绘制多个点构成一条线，当单位距离内点密度足够大时，看起来像一条线; 2.直接绘制由多条线段首尾相连构成的线；3.由多个面构成的一个看似一条线的面。  

#### 1.1.1 方式一实现原理
在单位距离内，由多个点均匀的排成一条直线，当点的数量足够多时，可以近似的看成一条直线。  

![方式一实现原理](https://travelclover.github.io/img/2022/10/%E7%82%B9%E6%95%88%E6%9E%9C%E5%AE%9E%E7%8E%B0%E5%8E%9F%E7%90%86.jpg)

优点:  
1.实现方式简单，只用绘制点就行。  
2.可以改变绘制点的大小，来实现不同粗细的线。  
缺点：  
1.放大层级范围时，能看出线不是连续的。  
2.实现的样式比较单调。

#### 1.1.2 方式二实现原理
直接由线段实现的方式是相对来说比较简单的。  
![方式二实现原理](https://travelclover.github.io/img/2022/10/%E7%BA%BF%E6%95%88%E6%9E%9C%E5%AE%9E%E7%8E%B0%E5%8E%9F%E7%90%86.png)

优点：  
1.简单。只需将线分割成多个线段，按顺序显示或隐藏对应线段就实现流动效果。  
2.就算放大层级范围，也显示的是完整线条。  
缺点：  
1.不能改变线的粗细。（因为数学家认为线没有粗细之分，所以永远都是细细的一根）  
2.样式过于单调，只能设置单一颜色。  

![方式二实现效果](https://travelclover.github.io/img/2022/10/%E7%BA%BF%E6%95%88%E6%9E%9C%E7%A4%BA%E6%84%8F%E5%9B%BE.gif)

#### 1.1.3 方式三实现原理
由多个三角面构成一条线的平面。  
![方式三实现原理](https://travelclover.github.io/img/2022/10/%E9%9D%A2%E6%95%88%E6%9E%9C%E5%AE%9E%E7%8E%B0%E5%8E%9F%E7%90%86.jpg)  

优点：
1.实现的效果丰富，想要的效果基本上都能实现。  
缺点：  
1.太复杂，太麻烦，复杂的效果需要自己写shader代码。  

### 1.2 方式一代码实现
点击链接查看[源代码](https://github.com/travelclover/flow-line-renderer)。  

## 2 打包代码
使用[webpack](https://webpack.js.org/)构建工具来打包，对源代码做处理。
```javascript
const path = require('path'); // 引入路径模块

function resolve(dir) {
  return path.resolve(__dirname, dir);
}

module.exports = {
  entry: './src/index.ts',
  mode: 'production',
  output: {
    clean: true, // 每次打包时清理对应文件夹
    path: resolve('dist'),
    filename: 'index.min.js', // 打包后文件名称
    library: { // 打包成库
      name: 'FlowLineRenderer', // 指定库的名称
      type: 'umd', // 通用模块定义模式，兼容commonJS、AMD、script标签引入
    },
  },
  module: {
    rules: [
      {
        test: /\.ts$/,
        use: 'ts-loader',
        include: [resolve('src')],
      },
    ],
  },
  resolve: {
    extensions: ['.ts', '.js'], // 针对于'.ts', '.js'这三种文件进行处理引入文件可以不写他的扩展名
  },
};
```

## 3 发布npm包
首先需要一个npm的项目。

### 3.1 创建一个npm项目
通过以下命令来生成一个package.json文件：
```
npm init
```
生成文件后，可以修改部分内容，修改后内容如下：
```
{
  "name": "flow-line-renderer", // 项目包名，发布到npm上就是这个名称，和现有npm库的包名不能重复
  "version": "1.1.0", // 版本号，更新包时必须修改版本号且必须更高的版本号
  "description": "A external renderer for ArcGIS JS API, used to show the flow effect of lines.", // 相关描述
  "keywords": [ // 关键词，在npm上搜索关键词可搜索到你的包
    "renderer",
    "line",
    "arcgis"
  ],
  "main": "dist/index.min.js", // 入口文件路径
  "scripts": { // 脚本命令
    "build": "webpack --config webpack.config.js",
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "repository": { // 仓库信息
    "type": "git",
    "url": "git+https://github.com/travelclover/flow-line-renderer.git"
  },
  "author": "travelclover", // 作者
  "license": "MIT", // 许可
  "files": [ // 上传到npm库的文件
    "/dist",
    "index.d.ts"
  ],
  "types": "index.d.ts", // 类型描述文件
  "bugs": { // 提BUG的地址
    "url": "https://github.com/travelclover/flow-line-renderer/issues"
  },
  "homepage": "https://github.com/travelclover/flow-line-renderer#readme", // 主页
  "dependencies": { // 依赖
    "gl-matrix": "^3.4.3"
  },
  "devDependencies": { // 开发依赖
    "cross-env": "^7.0.3",
    "ts-loader": "^9.4.1",
    "typescript": "^4.8.3",
    "webpack": "^5.74.0",
    "webpack-cli": "^4.10.0"
  }
}

```

### 3.2 注册npm账号
发布npm包之前，需要有一个npm的账号，可以去[npmjs官网](https://www.npmjs.com/)注册，或者通过终端命令注册：
```
npm adduser
```
如果是官网注册的账号就需要在终端中登录账号：
```
npm login
```

### 3.3 调试包
在发布包之前，可以先调试一下自己写的npm包，当然也可以直接跳过测试步骤直接发布。  
先在需要发布包的根目录里运行下面终端命令：  
```
npm link
```
这个命令的作用是在全局环境下，也就是nodejs安装目录下的node_modules目录下，生成一个符号链接文件，在windows下就是创建一个快捷方式文件，该文件的名字就是package.json文件中指定的模块名。因为它是一个快捷方式，所以当我们在项目下修改了什么东西，都会被全局的符号连接文件下面看到。  
然后在其它项目中需要与该npm包关联起来，所以在其它需要用到该包的项目（例如是test项目）中执行以下命令：
```
npm link 包名
```
在test项目下关联npm包后，只需在test项目里正常引用该包就行了：  
```javascript
import FlowLineRenderer from 'flow-line-renderer';
```
为了防止本地调试npm包与发布后的npm包混淆冲突，在调试完成后一定记得手动取消项目关联，在test项目下执行以下命令：
```
npm unlink 包名
```

### 3.4 发布包
发布包之前，先要更新`package.json`文件中的版本号，同样的版本号会导致发布失败。  
```
"version": "1.0.0",
```
执行以下命令发布包：
```
npm publish
```
正常情况下，这样就发布成功了，可以在npm中搜索刚发布的包了。  
当然，如果你用的不是npm官方镜像源，则会发布失败，需要先切换回官方镜像源：
```
npm config set registry https://registry.npmjs.org
```
或者在项目根目录新建`.npmrc`文件，写入以下内容即可：
```
registry=https://registry.npmjs.org
```

### 3.5 发布测试包
同样的需要先修改`package.json`文件中的版本号，可以在版本号后边加上`-beta`、`-beta1`、`-beta2`等标识。
```
"version": "1.0.0-beta",
```
发布测试版本，并打上标签:
```
npm publish --tag=beta
```
在项目中需要用到测试版本时通过以下命令安装：
```
npm install 包名@beta
```
