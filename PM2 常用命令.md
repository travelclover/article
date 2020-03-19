# PM2常用命令

最简单的命令：
```shell
$ pm2 start app.js # 启动app.js应用程序
```
还可以加入一些参数，例如：
```shell
$ pm2 start app.js --name <app_name> # 指定应用名称
$ pm2 start app.js --watch # 当文件变化时自动重启应用
$ pm2 start app.js --log <log_path> # 指定日志文件
```
管理程序状态:
```shell
$ pm2 restart app_name # 重启
$ pm2 reload app_name # 重载
$ pm2 stop app_name # 停止
$ pm2 delete app_name # 删除
```
> 你可以将==app_name==替换为:
> ==all== *对所有程序操作*
> ==id== *对特定的进程id操作*

其它用的较多的命令：

```shell
$ pm2 start script.sh # 启动 bash 脚本
$ pm2 list # 列表 PM2 启动的所有的应用程序
$ pm2 monit # 显示每个应用程序的CPU和内存占用情况
$ pm2 show [app-name] # 显示应用程序的所有信息
$ pm2 logs # 显示所有应用程序的日志
$ pm2 logs [app-name] # 显示指定应用程序的日志
$ pm2 flush # 清空所有日志文件
$ pm2 reset [app-name] # 重置元数据，例如重置重启数量

$ pm2 startup # 创建开机自启动命令
$ pm2 unstartup # 禁用自启动命令
$ pm2 save # 保存当前应用列表
$ pm2 resurrect # 重新加载保存的应用列表(通过pm2 save保存的应用)
$ pm2 update # 升级pm2，这之前最好先 pm2 save保存一下
```
线上网址： [https://pm2.keymetrics.io/](https://pm2.keymetrics.io/)
Github地址：[https://github.com/Unitech/pm2](https://github.com/Unitech/pm2)
