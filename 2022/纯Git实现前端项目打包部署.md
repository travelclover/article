
本篇文章主要是记录实现过程中遇到的问题，以及如何解决出现的问题，原始教程参考**杨成功**所写的这篇文章[《纯 Git 实现前端 CI/CD》](https://segmentfault.com/a/1190000040904889)。  

# 实现原理
利用Git Hooks功能，也就是我们所说的钩子功能，当服务器端的仓库收到客户端的push事件后，将对应的分支checkout到指定的目录，然后执行打包命令，也就实现了自动打包部署的功能。
# 实现步骤
## 1.在服务器中安装相应的软件程序
在服务器中安装所需要用到的软件：Node.js、Git、Nginx，具体的安装过程不细说了，不会的可以在网上查一下。
## 2.服务端创建裸仓库
登入服务器，找一个合适目录下创建一个裸仓库。这里演示是选择的 `/opt` 文件夹，该目录一般是用来安装软件应用程序的文件夹，一般都是空的。
```shell
$ cd /opt
$ git init --bare react-test.git
```
第一行命令是进入`opt`目录。  
第二行命令是创建一个名为`react-test.git`的裸仓库，`git init`是初始化命令，加上`--bare`参数就是创建裸仓库。裸仓库不包含工作区，所以不能在裸仓库上直接提交变更。
操作成功后会在`opt`目录下多出一个`react-test.git`文件夹。  
## 3.添加相应钩子文件
在`react-test.git`下有个`hooks`文件夹，里面存放的就是hook文件，hook文件就是`shell`脚本，在执行某个事件之前或之后进行一些其他额外的操作。  
![hooks文件](https://travelclover.github.io/img/2022/03/hooks文件.png)  
`hooks`文件夹下默认都是一些`.sample`结尾的文件，这些都是示例文件，是不会生效的，如果要生效，需要把文件名末尾的`.sample`去掉才行。例如`pre-commit.sample`需要改为`pre-commit`。  
默认情况下，`hooks`目录下只有部分钩子的示例文件，如果想要查看全部钩子说明，可参考[官网链接](https://git-scm.com/docs/githooks)。  
因为没有对应的钩子示例文件，所以我们需要手动添加一个名为`post-receive`钩子文件，该钩子文件会在代码push到这个裸仓库后执行。  
输入以下命令在`hooks`目录下添加`post-receive`钩子文件：
```shell
$ cd /opt/react-test.git/hooks
$ vim post-receive
```
然后将以下内容复制到`post-receive`文件中：
```shell
#!/bin/bash
echo 'server: received code push...'
cd /home/react-test

echo 'server: checkout latest code from git...'
git --git-dir=/opt/react-test.git --work-tree=/home/react-test checkout -f release

echo 'server: running npm install...'
npm install \

&& echo 'server: buildding...' \
&& npm run build \
&& echo 'server: done...'
```
`echo`命令作用是单纯的输出字符串。`cd /home/react-test`命令进入`/home/react-test`文件夹，该文件夹用来存放从git仓库检出的分支代码，以及后面要进行的项目打包步骤也是在该文件夹进行。  
`git --git-dir=/opt/react-test.git --work-tree=/home/react-test checkout -f release`这句命令是最关键的一句代码，该命令是一句`Git`命令，由于我们创建的是一个裸仓库，必须指定仓库目录地址以及工作区目录。`--git-dir=/opt/react-test.git`意思是指定git仓库目录，`--work-tree=/home/react/test`意思是指定工作区目录，`checkout`命令是检出分支，`-f`参数是指强制执行，`release`是代码仓库分支名称。这一整句Git命令的作用就是将`react-test.git`仓库的`release`分支检出到`/home/react-test`目录下。  
接下来的`npm install`和`npm run build`命令作用分别是安装依赖和代码打包构建，这样就实现了部署文件的更新。  
## 4.添加nginx解析
找到nginx的配置文件`nginx.conf`,修改如下配置：  
```nginx.conf
...其它配置
http {
	...其它配置
	server {
		listen       80;
        server_name  localhost;
        
		location / {
            root   html;
            index  index.html index.htm;
        }

		location /react-test {
            alias   /home/react-test/build;
            index  index.html index.htm;
        }
	}
	...其它配置
}
...其它配置
```
需要注意的是配置指向路径时，不一定都是`/home/react-test/build`，需要根据项目打包后的文件夹名称调整，如果打包后的文件夹是`dist`，那么指向的文件夹路径就应该是`/home/react-test/dist`。  
## 5.本地仓库设置以及推送代码
给本地仓库添加远端仓库地址，在添加前可以通过以下Git命令查看本地仓库已经链接的远端仓库地址：
```shell
$ git remote -v
```
如果是通过仓库地址`clone`的项目，应该可以看到已经有名称为`origin`的远端仓库地址，再添加远端仓库链接时就应该避免重名。通过以下命令添加刚才我们在服务端创建的裸仓库：  
```shell
$ git remote add prod ssh://root@45.40.253.19/opt/react-test.git
```
`git remote add`是添加远端仓库的命令，`prod`是设置名称，可以修改为其它的，`ssh://root@45.40.253.19/opt/react-test.git`是远端仓库地址，`root`是服务器登录账号，`45.40.253.19`是服务器IP地址。  
添加成功后，再次使用`git remote -v`就可以看到已经多出了名称为`prod`的远端仓库链接地址。  
然后就是通过以下命令将本地代码推送到我们的远端裸仓库上：
```shell
$ git push prod release
```
`git push`是向远端推送代码的命令，`prod`是远端仓库链接名称，需要和刚才添加远端仓库时设置的名称一致，`release`意思是需要向远端仓库哪个代码分支推送代码，需要和设置服务端钩子文件时检出的分支名称一致，也就是这句命令中`git --git-dir=/opt/react-test.git --work-tree=/home/react-test checkout -f release`的`release`名称一致，这样才能保证服务端检出到文件夹下的代码是我们想要打包的代码。  
执行这句命令后，就会将本地的代码推送到服务器端代码仓库，仓库收到后执行钩子文件，将最新的`release`分支代码检出到`/home/react-test`目录下，然后安装依赖，执行打包操作，最后文件夹下多出`build`（有可能为其它名称，由前端代码打包相关设置决定）打包后文件夹，完成项目部署。
# 出现的问题以及解决办法
## 1.钩子文件没有运行权限
如果向远端仓库提交代码后出现以下情况说明钩子文件没有运行权限：  
![钩子文件没有运行权限](https://travelclover.github.io/img/2022/03/钩子文件没有运行权限.png)  
这种情况赋予钩子文件执行权限即可，输入以下代码：  
```shell
$ chmod 700 /opt/react-test.git/hooks/pre-commit
```
## 2.Node.js版本太低
如果向远端提交代码出现下面这种情况则是Node.js版本太低：  
![Node.js版本太低](https://travelclover.github.io/img/2022/03/node版本太低.png)  
解决办法升级Node.js，安装`n`组件来管理`node`版本：  
```shell
$ npm install -g n
```
可以通过以下命令来管理版本：  
```shell
# 升级到指定版本 “n 版本号”
$ n 10.0.0
# 安装最新的版本
$ n latest
# 安装最近的稳定版本
$ n stable
```
可能会出现切换node版本失效的情况：  
![切换node版本失效](https://travelclover.github.io/img/2022/03/切换node版本失效.png)  
执行切换命令后没有切换版本成功。使用以下命令查看node安装路径：  
```shell
$ which node
```
`n` 默认安装路径是 `/usr/local`，若你的 node 不是在此路径下，`n` 切换版本就不能把bin、lib、include、share 复制该路径中，所以我们必须通过`N_PREFIX`变量来修改 `n` 的默认node安装路径。  
编辑环境配置文件：  
```shell
$ vim ~/.bash_profile
```
参照下图修改内容：  
![编辑环境配置文件](https://travelclover.github.io/img/2022/03/编辑环境配置文件.png)  
`export N_PREFIX=/usr/local`设置node实际安装位置。  
修改后通过以下命令使配置文件立即生效：  
```shell
$ source ~/.bash_profile
```
执行后，再次查看node版本就会发现已经成功切换版本了。  
![切换node版本成功](https://travelclover.github.io/img/2022/03/切换node版本成功.png)  

---
如果该文章对您有所帮助，请您一定不要吝啬您的鼓励。点赞、评论、分享、收藏、打赏都是您对我的鼓励和支持。  
如果您有`GitHub`账号，还可以[关注我~](https://github.com/travelclover)  
最后，感谢大家的阅读，如有错误，还请各位批评指正。  
