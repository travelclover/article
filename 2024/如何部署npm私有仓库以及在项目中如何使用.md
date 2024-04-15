# 如何部署npm私有仓库以及在项目中如何使用

## 为什么要部署npm私有仓库？
1. 安全性：私有仓库允许团队存放内部研发的、不宜公开发布的代码包，只对特定用户或者团队可见和可用，从而保护公司的知识产权和商业秘密。  
2. 模块的复用性：私有仓库可以存放一些公司内部使用的模块，例如使用的组件、工具类、配置文件等，这样，在项目开发中，就可以直接使用这些模块，而不需要重复开发。  
3. 性能优化：部署在内网环境下的私有仓库，可以显著减少依赖包在网络传输过程中的延迟和丢包风险，尤其在中国国内环境下，由于网络问题可能导致访问公共npm仓库速度较慢，私有仓库则能够显著提升依赖安装速度。  

## 有哪些工具可以部署私有npm仓库？
1. **[Verdaccio](https://verdaccio.org/)**：Verdaccio 是一款轻量级、易于配置和扩展的私有npm包服务器。它支持多用户、多存储库和灵活的身份验证机制，可以直接通过npm客户端与之交互，非常适合小到大型团队使用。
2. **[Sinopia](https://github.com/rlidwka/sinopia)**：Sinopia 是早期广泛使用的私有npm注册表解决方案，但是请注意，Sinopia目前不再维护，已经被Verdaccio取代，因为它不支持较新的npm特性。
3. **[Nexus Repository OSS](https://github.com/sonatype)**/**[Pro](https://www.sonatype.com/products/sonatype-nexus-oss)**：Nexus是由Sonatype公司提供的仓库管理软件，不仅支持npm私有仓库，还支持其他诸如Maven、Docker、Yum等多种类型的软件包仓库。Nexus Repository OSS是免费且开源的，而Nexus Pro提供了更多企业级的功能。
4. **[Artifactory](https://jfrog.com/artifactory/)**：Artifactory是JFrog公司出品的一款强大的二进制存储库管理器，同样支持npm私有仓库以及其他多种包类型。Artifactory为企业提供了全面的包管理和生命周期解决方案。
5. **[cnpm](https://npmmirror.com/)**: CNPM是阿里云提供的私有npm仓库解决方案，特别适合在中国地区部署和使用，以解决网络访问速度和稳定性问题。CNPM也支持搭建企业内部私有仓库，并且需要配合MySQL数据库进行数据存储。
6. **[定制开发]()**：如果希望深度定制或自行管理，理论上也可以基于npm本身的registry协议搭建自定义的私有仓库，但这通常需要更多的开发工作和技术支持。

## Verdaccio介绍
什么是Verdaccio？它的自我介绍原文是： 
> Verdaccio is a lightweight private npm proxy registry built in Node.js

Verdaccio是一个用Node.js构建的轻量级私有npm代理注册表。什么又是注册表呢？注册表是包的存储库，它实现了符合 CommonJS 规范的软件包注册表规范，用于读取软件包的信息。  

Verdaccio 这个单词的意思是中世纪晚期意大利流行的壁画绿色。

### Verdaccio的优点
1. **轻量级和易用性：**
   - Verdaccio 用 JavaScript 编写的，设计简洁，安装快速，资源占用较少，非常适合小型到中型企业或开发团队使用。
   - 零配置启动，即装即用，同时也支持丰富的自定义配置以满足不同的需求。
2. **易于部署和维护：**
   - 可以在多种操作系统上部署，包括Linux、macOS和Windows。
   - 支持Docker容器化部署，简化运维流程，便于在云端或虚拟机中快速部署和迁移。
3. **高度可定制化：**
   - Verdaccio 提供了丰富的配置选项，允许用户根据自己的需求进行定制化。
   - Verdaccio 是一个开源项目，你可以根据自己的需求自由修改和定制。你可以添加插件、修改源码、定制 UI 界面等，以满足特定的需求和场景。
4. **离线支持：**
   - Verdaccio 支持离线环境，你可以在没有网络连接的情况下使用它。
   - 可以缓存和镜像 npm 包，使得你可以在局域网中快速安装依赖，提高构建和部署的效率。
5. **社区活跃与持续更新：**
   - Verdaccio拥有活跃的社区支持，不断有新功能加入和完善，相比于一些已停止维护的工具（如Sinopia），能跟上npm生态的发展步伐。
   - 它与 yarn、npm 和 pnpm 100% 兼容。


### 如何安装Verdaccio?
因为Verdaccio 是一个 Node.js 应用程序，所以你需要安装 Node.js。如果你还没有安装 Node.js，请访问 [Node.js](https://nodejs.org/en/) 官网下载安装。Node.js 的版本要求`v14`或者更高，npm的版本也有要求`> npm@6.x | yarn@1.x | | yarn@2.x | pnpm@6.x`，不支持`npm@5.x`或更早的版本。  

Verdaccio 可以使用以下任一方法进行全局安装：
使用`npm`
```bash
npm install --location=global verdaccio
```
使用`yarn`
```bash
yarn global add verdaccio
```
使用`pnpm`
```bash
pnpm add -g verdaccio
```

安装Verdaccio就是这么简单，只需要简简单单一行命令，然后就可以使用Verdaccio了。
