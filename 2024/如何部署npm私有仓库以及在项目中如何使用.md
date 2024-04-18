# 如何部署npm私有仓库以及在项目中如何使用

## 为什么要部署npm私有仓库？
1. 安全性：私有仓库允许团队存放内部研发的、不宜公开发布的代码包，只对特定用户或者团队可见和可用，从而保护公司的知识产权和商业秘密。  
2. 模块的复用性：私有仓库可以存放一些公司内部使用的模块，例如使用的组件、工具类、配置文件等，这样在项目开发中，就可以直接使用这些模块，而不需要重复开发。  
3. 性能优化：部署在内网环境下的私有仓库，可以显著减少依赖包在网络传输过程中的延迟和丢包风险，尤其在中国国内环境下，由于网络问题可能导致访问公共npm仓库速度较慢，私有仓库则能够显著提升依赖安装速度。  

## 有哪些工具可以部署私有npm仓库
1. **[Verdaccio](https://verdaccio.org/)**：Verdaccio 是一款轻量级、易于配置和扩展的私有npm包服务器。它支持多用户、多存储库和灵活的身份验证机制，可以直接通过npm客户端与之交互，非常适合小到大型团队使用。
2. **[Sinopia](https://github.com/rlidwka/sinopia)**：Sinopia 是早期广泛使用的私有npm注册表解决方案，但是请注意，Sinopia目前不再维护，已经被Verdaccio取代，因为它不支持较新的npm特性。
3. **[Nexus Repository OSS](https://github.com/sonatype)**/**[Pro](https://www.sonatype.com/products/sonatype-nexus-oss)**：Nexus是由Sonatype公司提供的仓库管理软件，不仅支持npm私有仓库，还支持其他诸如Maven、Docker、Yum等多种类型的软件包仓库。Nexus Repository OSS是免费且开源的，而Nexus Pro提供了更多企业级的功能。
4. **[Artifactory](https://jfrog.com/artifactory/)**：Artifactory是JFrog公司出品的一款强大的二进制存储库管理器，同样支持npm私有仓库以及其他多种包类型。Artifactory为企业提供了全面的包管理和生命周期解决方案。
5. **[cnpm](https://npmmirror.com/)**: CNPM是阿里云提供的私有npm仓库解决方案，特别适合在中国地区部署和使用，以解决网络访问速度和稳定性问题。CNPM也支持搭建企业内部私有仓库，并且需要配合MySQL数据库进行数据存储。
6. **[定制开发]()**：如果希望深度定制或自行管理，理论上也可以基于npm本身的registry协议搭建自定义的私有仓库，但这通常需要更多的开发工作和技术支持。

## Verdaccio
![Verdaccio logo](https://travelclover.github.io/img/2024/04/verdaccio_logo.jpg)  
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


### 如何安装Verdaccio
因为Verdaccio 是一个 Node.js 应用程序，所以你需要安装 Node.js。如果你还没有安装 Node.js，请访问 [Node.js](https://nodejs.org/en/) 官网下载安装。Node.js 的版本要求`v14`或者更高，npm的版本也有要求`> npm@6.x | yarn@1.x | | yarn@2.x | pnpm@6.x`，不支持`npm@5.x`或更早的版本。  

Verdaccio 可以使用以下任一方法进行全局安装：  
使用`npm`
```bash
$ npm install --location=global verdaccio
```
使用`yarn`
```bash
$ yarn global add verdaccio
```
使用`pnpm`
```bash
$ pnpm add -g verdaccio
```

安装Verdaccio就是这么简单，只需要简简单单一行命令，然后就可以使用Verdaccio了。  
输入以下指令：
```bash
$ verdaccio --version
```
如果显示版本号则说明安装成功。

### 如何启动Verdaccio服务
需要启动Verdaccio服务也非常简单，只需要在全局安装后输入以下指令：
```bash
$ verdaccio
```
如果成功启动后，会打印以下内容：
```
info --- config file  - /home/.config/verdaccio/config.yaml
warn --- http address - http://localhost:4873/ - verdaccio/5.30.3
```
在浏览器中输入地址`http://localhost:4873`，就可以访问到Verdaccio的页面了。  
![Verdaccio页面](https://travelclover.github.io/img/2024/04/verdaccio_ui.jpg)  

### Verdaccio的命令
除了上面用到的查看版本的命令，Verdaccio还提供了以下命令：
| 命令   | 默认值   | 示例   | 描述   | 
| :---- | :---- | :---- | :---- |   
| --listen \ -l | http:localhost:4873 | 7000 | 定义协议+主机+端口 |  
| --config \ -c | ~/.local/verdaccio/config.yaml | ~./config.yaml | 设置配置文件的位置 |  
| --info \ -i |  |  | 打印本地环境信息 |  
| --version \ -v |  |  | 显示版本信息 |  

```bash
$ verdaccio --listen http://localhost:7000 --config /root/config.yaml
```
上面的命令中，指定了用`/root/config.yaml`配置文件来启动服务，并且监听本地主机的7000端口。这意味着当Verdaccio启动后，其他客户端可以通过 http://localhost:7000 地址与其交互，例如安装、发布和搜索私有npm包。

### 使用pm2守护Verdaccio进程
pm2是一个Node.js进程管理器，可以轻松管理Verdaccio进程。pm2可以确保Verdaccio进程在系统崩溃或重启时自动重新启动，从而确保服务始终运行。
安装pm2：
```bash
# npm
$ npm install -g pm2

# yarn
$ yarn global add pm2

# pnpm
$ pnpm add -g pm2
```
安装好后，使用以下命令守护并启动Verdaccio：
```bash
$ pm2 start verdaccio
```
使用`pm2 list`命令查看进程列表：
```
$ pm2 list
┌────┬────────────────────┬──────────┬──────┬───────────┬──────────┬──────────┐
│ id │ name               │ mode     │ ↺    │ status    │ cpu      │ memory   │
├────┼────────────────────┼──────────┼──────┼───────────┼──────────┼──────────┤
│ 0  │ verdaccio          │ fork     │ 0    │ online    │ 0%       │ 25.7mb   │
└────┴────────────────────┴──────────┴──────┴───────────┴──────────┴──────────┘
```

还有以下常用选项可以加在启动命令后：
```bash
# 指定应用名称
--name <app_name>

# 设置应用程序加载的内存阈值
--max-memory-restart <200MB>

# 指定日志文件
--log <log_path>

# 自动重启间隔时间
--restart-delay <delay in ms>
```

除了`start`命令外，`pm2`还有以下这些常用命令：
| 命令   | 描述   | 
| :---- | :---- |  
| pm2 list | 显示所有进程状态 |  
| pm2 logs | 打印日志 |  
| pm2 restart <app_name> | 重启指定名称的应用程序 |  
| pm2 reload <app_name> | 平滑地重启指定的应用程序，相对于 pm2 restart，reload 命令旨在尽可能减小因重启过程带来的服务中断时间，特别适用于需要始终保持在线服务的场景 |  
| pm2 stop <app_name> | 停止指定名称的应用程序 |  
| pm2 delete <app_name> | 删除指定名称的应用程序 |  

### Verdaccio配置文件
verdaccio服务启动后，会在启动服务对应的目录下创建一个名为`verdaccio`的文件夹，文件夹下有个`storage`文件夹和`config.yaml`文件。`storage`文件夹下存放的是 Verdaccio 存储的包，`config.yaml`文件是 Verdaccio 的默认配置文件。  

```yaml
# 存储路径
storage: ./storage
# 插件路径
plugins: ./plugins

# Web UI 相关的参数
web:
  title: Verdaccio
  # Gravatar 是一项全球性的头像服务，允许用户使用同一个邮箱在多个网站上使用同一个头像。在这里，将 gravatar 设置为 false 可以禁用 Gravatar 支持。
  gravatar: false
  # 包列表的排序顺序 (asc|desc) 默认为升序asc
  sort_packages: desc
  # 是否启用暗黑模式
  # darkMode: true
  # 是否启用 HTML 缓存
  # html_cache: true
  # 是否启用登录功能
  # login: true
  # 是否显示额外的信息
  # showInfo: true
  # 是否显示设置选项
  # showSettings: true
  # 是否显示主题切换选项
  # showThemeSwitch: true
  # 是否显示页脚
  # showFooter: true
  # 是否显示搜索功能
  # showSearch: true
  # 是否显示原始数据或源代码
  # showRaw: true
  # 是否显示下载 tarball 的选项，tarball 是一种常见的文件压缩格式，通常用于打包和传输文件。
  # 启用 showDownloadTarball 功能可以让用户更方便地获取到特定包的 tarball 文件，以便于离线安装或其他用途。
  # showDownloadTarball: true
  # scriptsBodyAfter:
  #    - '<script type="text/javascript" src="https://my.company.com/customJS.min.js"></script>'
  # metaScripts:
  #    - '<script type="text/javascript" src="https://code.jquery.com/jquery-3.5.1.slim.min.js"></script>'
  #    - '<script type="text/javascript" src="https://browser.sentry-cdn.com/5.15.5/bundle.min.js"></script>'
  #    - '<meta name="robots" content="noindex" />'
  # bodyBefore:
  #    - '<div id="myId">html before webpack scripts</div>'
  # 配置项用于指定公共路径或基础 URL
  # publicPath: http://somedomain.org/

# 身份验证，默认身份验证基于 htpasswd 并内置，可以通过插件修改
auth:
  htpasswd:
    file: ./htpasswd
    # 允许注册的最大用户数, 默认是无限制
    # 设置为-1，表示禁止注册
    max_users: -1
    # 哈希算法，可选项为“bcrypt”、“md5”、“sha1”、“crypt”
    algorithm: bcrypt # 默认算法是 crypt ，被认为对生产环境不安全，建议新安装改用 bcrypt 
    # “bcrypt”的舍入数，对于其他算法将被忽略
    rounds: 10

# 上行链路是具有外部注册表的链接，用于提供对外部包的访问。
# 可以定义多个上行链路，每个上行链路都必须具有唯一的名称
uplinks:
  npmjs:
    url: https://registry.npmjs.org/ # 
  yarn:
    url: https://registry.yarnpkg.com/
    timeout: 10s
  cnpm:
    url: https://registry.npmmirror.com/
    timeout: 10s

# 配置包访问的权限
# "$all" 所有用户, "$anonymous" 匿名用户, "$authenticated" 认证用户
packages:
  # 指定了对以 @dd/ 开头的所有包的访问和发布权限
  '@dd/*':
    # 特定作用域的包 @dd 一般为公司名 后面的为具体包名
    access: $all # 所有用户都能访问
    publish: $authenticated # 注册并且登录的用户才能发布
    unpublish: $authenticated # 注册并且登录的用户才能取消发布
    proxy: npmjs # 如果包在本地不可用，指定代理上行链路中的npmjs注册表

  '**':
    access: $all
    publish: $authenticated
    unpublish: $authenticated
    proxy: npmjs

server:
  keepAliveTimeout: 60
  # Allow `req.ip` to resolve properly when Verdaccio is behind a proxy or load-balancer
  # See: https://expressjs.com/en/guide/behind-proxies.html
  # trustProxy: '127.0.0.1'

# 是否允许离线发布
# publish:
#   allow_offline: false

# 配置 Verdaccio 的公共 URL 前缀，用于解决反向代理问题
# url_prefix: /verdaccio/
# VERDACCIO_PUBLIC_URL='https://somedomain.org';
# url_prefix: '/my_prefix'
# // url -> https://somedomain.org/my_prefix/
# VERDACCIO_PUBLIC_URL='https://somedomain.org';
# url_prefix: '/'
# // url -> https://somedomain.org/
# VERDACCIO_PUBLIC_URL='https://somedomain.org/first_prefix';
# url_prefix: '/second_prefix'
# // url -> https://somedomain.org/second_prefix/'

# 安全相关设置
# security:
#   api:
#     legacy: true
#     jwt:
#       sign:
#         expiresIn: 29d
#       verify:
#         someProp: [value]
#    web:
#      sign:
#        expiresIn: 1h # 1 hour by default
#      verify:
#         someProp: [value]

# 用户速率限制的设置
# userRateLimit:
#   windowMs: 50000
#   max: 1000

# 最大请求体大小限制
# max_body_size: 10mb

# 监听的地址和端口配置
listen:
#  localhost:4873            # 默认值，本机4873端口
#  http://localhost:4873     # 和上个配置相同
 - 0.0.0.0:4873              # 所有网络接口的 4873 端口
# - https://example.org:4873  # if you want to use https
# - "[::1]:4873"                # ipv6
# - unix:/tmp/verdaccio.sock    # unix socket

# https证书相关设置
# https:
#   key: ./path/verdaccio-key.pem
#   cert: ./path/verdaccio-cert.pem
#   ca: ./path/verdaccio-csr.pem

# 配置 HTTP 和 HTTPS 代理的设置
# http_proxy: http://something.local/
# https_proxy: https://something.local/

# 用于配置通知功能
# notify:
#   method: POST
#   headers: [{ "Content-Type": "application/json" }]
#   endpoint: https://usagge.hipchat.com/v2/room/3729485/notification?auth_token=mySecretToken
#   content: '{"color":"green","message":"New package published: * {{ name }}*","notify":true,"message_format":"text"}'

# 启用审核中间件
middlewares:
  audit:
    enabled: true

# 日志设置
# type: stdout：指定日志输出到标准输出流（stdout），通常是终端或命令行界面
# format: pretty：指定日志格式为漂亮的可读格式，这意味着日志信息将以易于阅读的方式呈现，通常包括时间戳、日志级别、消息内容等
# level: http：指定日志记录的级别为 http。这意味着只有级别为 http 及以上的日志信息（例如 http、warn、error 等）才会被输出。
log: { type: stdout, format: pretty, level: http }

# 一些实验性功能的设置
# experiments:
#  # support for npm token command
#  token: false
#  # disable writing body size to logs, read more on ticket 1912
#  bytesin_off: false
#  # enable tarball URL redirect for hosting tarball with a different server, the tarball_url_redirect can be a template string
#  tarball_url_redirect: 'https://mycdn.com/verdaccio/${packageName}/${filename}'
#  # the tarball_url_redirect can be a function, takes packageName and filename and returns the url, when working with a js configuration file
#  tarball_url_redirect(packageName, filename) {
#    const signedUrl = // generate a signed url
#    return signedUrl;
#  }

# 指定了网页界面的语言
i18n:
  web: zh-CN

```

## 在项目中使用私有仓库
有了私有仓库，我们在项目中有多种使用方式，大多数私有仓库只允许认证用户访问，所以需要先注册用户。
### 注册私有仓库用户
首先需要注册私有仓库用户，前提是私有仓库已经设置为允许注册用户。客户端身份验证由 npm 客户端本身处理，使用以下命令创建用户，并按照提示输入名字、密码、邮箱：
```bash
$ npm adduser --registry http://localhost:4873
```
`http://localhost:4873`为私有仓库的地址。如果不在命令中指定参数`--registry http://localhost:4873`，则注册用户的操作就是面向官方的npm仓库。当然也可以在执行上述命令之前先切换仓库源：
```bash
$ npm config set registry http://localhost:4873
```
这样设置以后，就改变了全局的仓库源，后续的操作中就不需要每次输入命令时都带上`--registry http://localhost:4873`参数，但后续的所有命令都是针对`http://localhost:4873`源。如果需要切换回官方的仓库源，只需要再次输入以下命令：
```bash
$ npm config set registry https://registry.npmjs.org
```
> 注意：后续示例代码中默认已经切换了仓库源，就不再每条命令后添加`--registry`参数。

### 登录私有仓库
如果私有仓库设置了身份验证（大多数私有仓库都会设置身份验证），就需要登录：
```bash
$ npm login --registry http://localhost:4873 # 如果已经切换了仓库源可以不用设置 registry 参数
```
登录以后可以输入以下命令查看是否登录成功：
```bash
$ npm whoami --registry http://localhost:4873 # 如果已经切换了仓库源可以不用设置 registry 参数
```
如果打印出用户名，说明登录成功。

### 发布包到私有仓库
将需要发布的包代码准备好后，确保`package.json`中的版本号已经修改，每次发布新版本的版本号只能比以前的版本号大才能发布成功，输入以下命令发布包到私有仓库：
```bash
$ npm publish --registry http://localhost:4873 # 如果已经切换了仓库源可以不用设置 registry 参数
```
如果不出意外，将会打印包的信息，发布成功以后，在私有仓库中可以看到已经发布了的包。

### 从私有仓库拉取包
如果是已经切换成私人仓库源的话，直接使用 `npm install` 命令即可。如果是需要安装某个私人仓库里的某个特定的包，可以使用以下命令：
```bash
$ npm install <package_name> --registry=http://localhost:4873
```
更加推荐的做法是在项目中新建 `.npmrc` 文件，指定特殊的命名空间（scope）的来源：
```
@dd:registry=http://localhost:4873
```
以@dd 开头的包从 `registry=http://localhost:4873` 这里下载，其余的会从默认仓库源下载。这样操作以后，后续再安装依赖时，特定的包走私有源，其他包走正常的源安装。

### 从私有仓库删除包
如果是需要删除指定版本的某个包，输入以下命令：
```bash
$ npm unpublish <package_name>@<version>
```
如果是需要删除整个包：
```bash
$ npm unpublish <package_name> --force
```
不建议删除已经发布的包，可能会对已经引用的项目造成影响。推荐使用以下命令来标记包已经过时，不推荐安装使用：  
```bash
$ npm deprecate <package_name>@<version> <message>
```

### 退出私有仓库
使用npm退出私有仓库：
```bash
$ npm logout --registry http://localhost:4873 # 如果已经切换了仓库源可以不用设置 registry 参数
```
如果没有任何提示就代表退出成功，再次输入`npm whoami`将会打印错误信息。

## 可能会遇到的问题
在整个过程中，可能会遇到以下问题，可以尝试以下方案解决。

### 1. npm 全局安装 verdaccio 失败
使用npm安装verdaccio时，可能会出现如下错误：
```
npm ERR! code 127
npm ERR! path /root/.nvm/versions/node/v16.17.1/lib/node_modules/verdaccio/node_modules/core-js
npm ERR! command failed
npm ERR! command sh /tmp/postinstall-e406646f.sh
npm ERR! /tmp/postinstall-e406646f.sh:行1: node: 未找到命令
```
再确定终端中分别输入`node --version`和`npm --version`都能正确打印版本信息，则可能是对应的文件夹没有相关权限导致安装失败，可以尝试给对应文件夹权限：
```bash
$ chmod -R 777 /root
```
然后再使用`npm install -g verdaccio`安装。

### 2. 使用 pm2 启动 verdaccio 失败
使用pm2启动verdaccio后，输入`pm2 list`命令，会显示 verdaccio 程序状态为 `errored`，使用`pm2 logs`命令打印日志，输出如下错误：
```
Error: Cannot find module '/root/verdaccio'
    at Function.Module._resolveFilename (node:internal/modules/cjs/loader:956:15)
    at Function.Module._load (node:internal/modules/cjs/loader:804:27)
    at Object.<anonymous> (/root/node-v16.19.1-linux-x64/lib/node_modules/pm2/lib/ProcessContainerFork.js:33:23)
    at Module._compile (node:internal/modules/cjs/loader:1126:14)
    at Object.Module._extensions..js (node:internal/modules/cjs/loader:1180:10)
    at Module.load (node:internal/modules/cjs/loader:1004:32)
    at Function.Module._load (node:internal/modules/cjs/loader:839:12)
    at Function.executeUserEntryPoint [as runMain] (node:internal/modules/run_main:81:12)
    at node:internal/main/run_main_module:17:47 {
  code: 'MODULE_NOT_FOUND',
  requireStack: []
}
```
报找不到verdacio模块的错误，可以尝试使用以下命令重新启动verdaccio：
```bash
$ pm2 start `which verdaccio`
```
反引号 \` 是一种在很多编程语言中用来执行命令并将结果插入到字符串中的语法。在这个特定的命令中，`which verdaccio` 执行了一个 shell 命令，即 which verdaccio，它会返回 Verdaccio 可执行文件的路径。这个路径会被插入到 pm2 start 命令中，以便启动 Verdaccio 服务。

### 3. 服务启动后，互联网用户不能访问
服务器中启动verdaccio服务后，互联网用户访问不到，首先要确定云服务器是否开启了对应端口。同时还需要将verdaccio配置文件中的端口监听由`localhost:4873`改为`0.0.0.0:4873`。

## 总结
使用 Verdaccio 可以很轻松容易的搭建一个私有的 npm 仓库，可以为团队或个人提供一个方便管理和分享 npm 包的平台，部署和使用都非常简单，大家快用起来吧！
