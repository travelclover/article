# 安装人大金仓数据库KingbaseES以及PostGIS扩展

[人大金仓数据库（KingbaseES）](https://www.kingbase.com.cn/product/details_549_476.html)是一款国产的高性能关系型数据库管理系统，广泛应用于政府、金融、电信等领域。[PostGIS](https://postgis.net/)是[PostgreSQL](https://www.postgresql.org/)数据库的一个空间数据库扩展，提供了丰富的空间数据处理功能。本文将详细介绍如何在KingbaseES中安装PostGIS扩展，以便在KingbaseES中使用空间数据，以及用于geoserver发布服务，QGIS、GeoscenePro等软件连接等。

# 1 安装人大金仓数据库KingbaseES
## 1.1 下载数据库
首先需要下载安装包，下载地址：[https://download.kingbase.com.cn/xzzx/index.htm](https://download.kingbase.com.cn/xzzx/index.htm)。  
选择目标版本，然后根据服务器CPU型号和操作系统下载对应的安装文件。本文示例选择的版本为`V008R006C008B0020`，CPU为`X86`，操作系统为`Linux`。点击下载按钮下载文件，如下图所示。  
![kingbase下载页面](https://travelclover.github.io/img/2025/02/kingbase下载页面.jpg)

## 1.2 下载数据库授权文件
安装KingbaseES需要授权文件，正常情况需要购买才会有长期授权文件。我们可以下载一个临时的短期的授权文件。需要注意的是右上角选择的版本要与刚才下载的数据库版本一致，并且不要选择 *`开发版`* 和 *`标准版`* 的授权文件下载，因为这两种不支持GIS扩展。  
![授权文件下载页面](https://travelclover.github.io/img/2025/02/授权文件下载页面.jpg)  
授权版本差异如下所示：
| 标准版   | 专业版   | 企业版   | 开发版   | 
| :---- | :---- | :---- | :---- |     
| 金仓数据库管理系统KES标准版是为了满足政府部门、中小型企业客户的简单应用而推出的一款精简数据库产品，该产品提供除集群、GIS以外的全部应用开发与系统管理功能，支持 TB 级海量数据，支持多用户并发访问，支持所有国内外主流的CPU、操作系统与云平台部署。 | 金仓数据库管理系统KES专业版是一款入选双名录的产品，可满足党政客户、中小企业客户的中等规模、非核心业务应用场景的要求，提供全部应用开发及系统管理功能，支持主备集群、读写分离集群等集群架构，支持所有国内外主流CPU、操作系统与云平台部署。 | 金仓数据库管理系统 KES 企业版是面向全行业、全客户关键应用的大型通用数据库管理系统，适用于联机事务处理、查询密集型数据仓库、要求苛刻的互联网应用等场景，提供全部应用开发及系统管理功能，提供性能增强特性，可支持主备集群、读写分离集群、多活共享存储集群等全集群架构，具有高性能、高安全、高可用、易使用、易管理、易维护的特点，支持所有国内外主流CPU、操作系统与云平台部署。 | 金仓数据库管理系统KES开发版是为了满足用户的产品试用、日常开发需求而推出的一个精简版本，提供除集群、GIS以外的全部应用开发及系统管理功能。该版本可用10个数据库连接，试用1年，不能用于商业用途。 |  

## 1.3 创建安装用户
在安装KingbaseES时，安装用户对于安装路径需有“读”、“写”、“执行”的权限。在Linux系统中，需要以非root用户执行安装程序，且该用户要有标准的home目录。  
因此，建议在正式安装前，新建kingbase用户作为KingbaseES专用的系统用户，您可以先使用root用户运行如下命令创建kingbase用户：  
```shell
useradd -m kingbase
```
> **`useradd`**: 这是用于创建新用户的命令。  
> **`-m`**: 这个选项表示在创建用户时同时创建该用户的主目录。如果没有这个选项，用户将不会有主目录。主目录通常位于 /home 下，名称与用户名相同（例如，/home/kingbase）  
> **`kingbase`**: 这是要创建的新用户的用户名。在这个例子中，将创建一个名为 kingbase 的用户。

创建用户后，该用户将没有密码，因此你需要为其设置密码，可以使用 `passwd` 命令：
```shell
passwd kingbase
```
> **`注意`**: 密码需要输入两次，保证两次输入的密码相同。

## 1.4 创建安装目录
KingbaseES默认的安装目录是 `/opt/Kingbase/ES/V8` 。如果不存在，您需要使用root用户输入以下命令来创建该目录：
```shell
mkdir -p /opt/Kingbase/ES/V8
```
> **`mkdir`**: 这是用于创建新目录的命令。  
> **`-p`**: 这个选项表示“父目录”的意思。使用 -p 选项时，如果上级目录不存在，mkdir 将会自动创建所有必要的上级目录，而不会返回错误。  
> **`/opt/Kingbase/ES/V8`**: 这是要创建的目标目录的完整路径。在这个例子中，命令会创建 V8 目录，并确保 /opt/Kingbase/ES 及其上级目录（如果它们不存在）也会被创建。

然后输入以下命令赋予kingbase用户对该目录的读写权限：
```shell
chmod o+rwx /opt/Kingbase/ES/V8
```
> **`chmod`**: 这是用于更改文件或目录权限的命令。  
> **`o`**: 代表 "others"，即其他用户（不属于文件或目录所有者的用户和不属于文件或目录所属组的用户）。
> **`+`**: 表示添加权限。
> **`r`**: 读取权限（read）。
> **`w`**: 写入权限（write）。
> **`x`**: 执行权限（execute）。

## 1.5 安装包的挂载与取消
将1.1步骤下载的安装包放到服务器的某个目录下，比如 `/opt/` 目录下。  
iso格式的安装程序包需要先挂载才能使用。挂载iso文件需要使用root用户。比如挂载的目录是iso文件同级目录KingbaseES，您可以使用以下命令进入对应文件夹，然后创建挂载目录`KingbaseES`：
```shell
cd /opt
mkdir KingbaseES
```
然后使用以下命令挂载iso文件：
```shell
mount ./KingbaseES_V008R006C008B0020_Lin64_install.iso ./KingbaseES
```
操作成功后，可以在KingbaseES文件夹下看到setup目录和setup.sh脚本。  
![查看挂载目录](https://travelclover.github.io/img/2025/02/查看挂载目录.jpg)

安装完成后您可以使用root用户运行如下命令取消挂载iso文件：
```shell  
umount ./KingbaseES
```
操作成功后KingbaseES目录和iso文件已经解除挂载关系，在KingbaseES目录下不会再看到安装相关文件。  

## 1.6 运行安装程序
在挂载iso文件后，将用户切换到kingbase：
```shell
su - kingbase
```
进入KingbaseES目录再执行安装：
```shell
cd /opt/KingbaseES
```
![切换kingbase用户](https://travelclover.github.io/img/2025/02/切换kingbase用户.jpg)  

在setup.sh文件所在目录下，使用以下命令运行setup.sh脚本：  
```shell
sh setup.sh -i console
```
![启动安装程序](https://travelclover.github.io/img/2025/02/启动安装程序.jpg)

当出现以下类似信息时，需要按回车键继续：
![按回车键继续](https://travelclover.github.io/img/2025/02/按回车键继续.jpg)

当出现是否同意时，输入`Y`，然后回车继续：
![输入Y回车键](https://travelclover.github.io/img/2025/02/输入Y回车键.jpg)

## 1.7 选择安装集
根据安装后数据库服务功能的不同，KingbaseES可分为完全安装、客户端安装和定制安装三种安装集。  
1. Full: 完全安装，包括数据库服务器、高可用组件、接口、数据库开发管理工具、数据库迁移工具、数据库部署工具。  
2. Client: 客户端安装，包括接口、数据库开发管理工具、数据库迁移工具、数据库部署工具。
3. Custom: 定制安装，在数据库服务器、高可用组件、接口、数据库开发管理工具、数据库迁移工具、数据库部署工具所有组件中自由选择。
  
![选择安装集](https://travelclover.github.io/img/2025/02/选择安装集.jpg)
作为初次安装的我们，直接按回车键，或者输入1，选择完全安装。

## 1.8 选择授权文件
我们可以提前将下载的临时授权文件解压后放到`/opt`路径下，当出现下面内容时，输入授权文件的绝对路径后，按回车键以检查授权文件，若授权文件有效，则进入下一步骤。
![输入授权文件路径](https://travelclover.github.io/img/2025/02/输入授权文件路径.jpg)

## 1.9 选择安装路径
当出现下面内容时，我们可以设置安装路径。默认安装路径是`/opt/Kingbase/ES/V8`，我们也已经提前创建好了目录，所以直接按回车键即可。
![设置安装目录](https://travelclover.github.io/img/2025/02/设置安装目录.jpg)
如果提示以下信息：You do not have write permissions to the chosen installation destination. Please choose a different location for installation. 可以运行以下命令修改目录的所有者和所属组：
```shell
chown -R kingbase:kingbase /opt/Kingbase/ES/V8
```
安装路径没问题后，会输出一些摘要信息，包括：产品名称、安装文件夹、指定安装的功能组件和安装路径所在磁盘空间信息，直接按回车键继续。
![安装时的摘要信息](https://travelclover.github.io/img/2025/02/安装时的摘要信息.jpg)
继续回车键。
![准备开始安装](https://travelclover.github.io/img/2025/02/准备开始安装.jpg)

## 1.10 设置初始化数据库参数
### 1.10.1 设置数据库数据目录
出现以下内容时，设置数据库数据目录，直接回车即可，默认数据库数据目录为安装目录下的data目录。
![选择数据目录](https://travelclover.github.io/img/2025/02/选择数据目录.jpg)

### 1.10.2 设置端口
默认端口为`54321`，可以不修改，直接按回车键继续。
![设置端口](https://travelclover.github.io/img/2025/02/设置端口.jpg)

### 1.10.3 设置数据库管理员
默认为`system`，可以不修改，直接按回车键继续。
![设置数据库管理员](https://travelclover.github.io/img/2025/02/设置数据库管理员.jpg)

### 1.10.4 设置密码
设置自定义密码，并保存好，需要连续输入两次，两次都必须是一样的。
![设置密码](https://travelclover.github.io/img/2025/02/设置密码.jpg)

### 1.10.5 设置字符集编码
默认编码为`UTF8`，可以不修改，直接按回车键继续。
![设置字符集编码](https://travelclover.github.io/img/2025/02/设置字符集编码.jpg)

### 1.10.6 设置数据库区域
直接按回车键继续。
![设置数据库区域](https://travelclover.github.io/img/2025/02/设置数据库区域.jpg)

### 1.10.7 设置数据库兼容模式
这里数据库兼容模式需要改为`PG`，不要使用默认值，后续我们需要安装`postgis`扩展，所以这里输入`1`，然后回车。
![设置数据库兼容模式](https://travelclover.github.io/img/2025/02/设置数据库兼容模式.jpg)

### 1.10.8 设置大小写敏感
默认大小写敏感为`YES`，可以不修改，直接按回车键继续。
![设置大小写敏感](https://travelclover.github.io/img/2025/02/设置大小写敏感.jpg)

### 1.10.9 设置存储块大小
默认存储块大小为`8KB`，可以不修改，直接按回车键继续。
![设置存储块大小](https://travelclover.github.io/img/2025/02/设置存储块大小.jpg)

### 1.10.10 设置身份认证方法
使用默认，直接按回车键继续。
![设置身份认证方法](https://travelclover.github.io/img/2025/02/设置身份认证方法.jpg)

### 1.10.11 完成安装
如果没有错误信息，且出现以下信息，则代表安装完成，按回车键可退出安装程序。
![完成安装](https://travelclover.github.io/img/2025/02/完成安装.jpg)
完成安装后，可切换为root用户，执行取消挂载操作。如果系统返回错误信息 `target is busy`，这意味着该挂载点正在被某些进程使用，导致无法卸载，可以尝试使用 `-l` 选项进行懒卸载：
```shell
umount -l /opt/KingbaseES
```

## 1.11 执行root.sh
如果想注册数据库服务为系统服务，您可以在安装并初始化数据库成功后，执行root.sh脚本来注册并启动数据库服务，具体步骤如下：
切换回root用户：
```shell
su - root
```
执行root.sh脚本：
```shell
/opt/Kingbase/ES/V8/install/script/root.sh
```

## 1.12 启动或停止数据库服务
如果想启动或停止数据库服务，可执行如下命令。因为无法以root用户运行，所以需要先切换为非root用户，比如我们开始创建的kingbase用户：
```shell
su - kingbase
```
启动服务：
```shell
/opt/Kingbase/ES/V8/Server/bin/sys_ctl -w start -D /opt/Kingbase/ES/V8/data -l "/opt/Kingbase/ES/V8/data/sys_log/startup.log"
```
停止服务：
```shell
/opt/Kingbase/ES/V8/Server/bin/sys_ctl stop -m fast -w -D /opt/Kingbase/ES/V8/data
```
为了不每次都输入这么长的命令，我们可以先进入到`bin`目录下：
```shell
cd /opt/Kingbase/ES/V8/Server/bin
```
再输入命令：
```shell
./sys_ctl stop -m fast -w -D /opt/Kingbase/ES/V8/data
```

## 1.13 连接数据库
我们可以通过命令行工具连接数据库，比如使用以下命令：
```shell
./ksql -U system -d test -p 54321
```
> **`-U system`**：指定以 system 用户身份连接数据库。
> **`-d test`**：指定连接到名为 test 的数据库，test为默认就有的数据库。
> **`-p 54321`**：指定连接到数据库的端口号为 54321。

正确输入密码后，就会如下图所示，光标的前面是连接的数据名称，说明已经连接成功，已经进入`ksql`命令行界面，可以开始执行SQL查询。
![连接test数据库](https://travelclover.github.io/img/2025/02/连接test数据库.jpg)
输入以下命令，加回车，可查看安装的版本：
```sql
SELECT version();
```

最后可通过输入`\q`，按回车，退出`ksql`命令行界面。

# 2 安装postgis扩展
首先，安装的postgis版本需要与安装的kingbaseES版本对应起来，否则会导致错误。正常情况如果购买了正版授权文件，可以找工作人员索要对应版本的postgis安装包。  
通过关注微信公众号《Web与GIS》，回复关键字：`POSTGIS` 可下载适用于版本为`V008R006C008B0020`的postgis安装包。  

## 2.1 解压postgis安装包
将下载好的postgis安装包先传到服务器某个路径下，例如`/opt`，然后使用解压命令：
```shell
tar -zxvf postgis-3.1.2_X86_V008R006C008B0020.tar.gz
```
解压后会得到名称为`postgis-3.1.2`的文件夹。
![解压postgis安装包](https://travelclover.github.io/img/2025/02/解压postgis安装包.jpg)

## 2.2 将文件拷贝到指定路径下
`postgis-3.1.2`文件夹下会有三个文件夹，分别是`bin`、`lib`和`share`。  
先将`bin`文件夹下的文件拷贝到`/opt/Kingbase/ES/V8/KESRealPro/V008R006C008B0020/Server/bin`目录下：
```shell
cp -f /opt/postgis-3.1.2/bin/* /opt/Kingbase/ES/V8/KESRealPro/V008R006C008B0020/Server/bin/
```
然后将`lib`文件夹下的文件拷贝到`/opt/Kingbase/ES/V8/KESRealPro/V008R006C008B0020/Server/lib`目录下：
```shell
cp -f /opt/postgis-3.1.2/lib/* /opt/Kingbase/ES/V8/KESRealPro/V008R006C008B0020/Server/lib/
```
如果询问是否覆盖同名文件，输入`y`，然后回车。  
最后将`share`文件夹下的`extension`下的所有文件拷贝到`/opt/Kingbase/ES/V8/KESRealPro/V008R006C008B0020/Server/share/extension`目录下：
```shell
cp -f /opt/postgis-3.1.2/share/extension/* /opt/Kingbase/ES/V8/KESRealPro/V008R006C008B0020/Server/share/extension/
```

## 2.3 创建空间数据库（SDE库）
首先使用数据库system用户连接数据库，并创建一个新的名为`gisTest`的数据库。
```shell
./ksql -U system -d test -p 54321
```
```sql
CREATE DATABASE gisTest;
```
可以使用`\l`来查看全部数据库，看是否已经有了刚创建的数据库。
![创建数据库](https://travelclover.github.io/img/2025/02/创建数据库.jpg)

然后切换到新创建的数据库：
```sql
\c gistest
```
![切换数据库](https://travelclover.github.io/img/2025/02/切换数据库.jpg)

安装postgis扩展：
```sql
CREATE EXTENSION postgis;
CREATE EXTENSION postgis_topology;
```
除了以上扩展外，还可以安装其他扩展，请自行搜索。

如果希望将空间数据的管理权限与其他用户隔离，建议新建一个专用的 sde 用户。例如：
```sql
CREATE USER sde WITH PASSWORD 'sde@123';
```
创建 sde 模式：
```sql
CREATE SCHEMA sde;
```
授予 sde 用户对 sde 模式的权限：
```sql
GRANT ALL PRIVILEGES ON SCHEMA sde TO sde;
```
授予 sde 用户对数据库的权限：
```sql
GRANT ALL PRIVILEGES ON DATABASE gistest TO sde;
```

现在就可以通过QGIS或者Geoscene Pro连接空间数据库了。

# 3 总结
通过本文的步骤，您可以成功安装 KingbaseES 数据库并启用 PostGIS 扩展，为空间数据的管理和分析提供强大支持。KingbaseES 作为国产数据库，结合 PostGIS 扩展，能够满足地理信息系统（GIS）和空间数据处理的多样化需求。希望本文能为您的空间数据库实践提供帮助！


# 4 安装包下载
1. 人大金仓数据库和授权文件下载地址：[https://download.kingbase.com.cn/xzzx/index.htm](https://download.kingbase.com.cn/xzzx/index.htm)。
2. postgis安装包下载地址：通过关注微信公众号《Web与GIS》，回复关键字`POSTGIS`下载。
> **`注意`**：本文人大金仓数据库版本为`V008R006C008B0020`，下载的postgis安装包支持该版本。

![微信公众号](https://travelclover.github.io/img/2025/02/微信公众号.png)

---
如果该文章对您有所帮助，请您一定不要吝啬您的鼓励。点赞、评论、分享、收藏、打赏都是您对我的鼓励和支持。  
如果您有`GitHub`账号，还可以[关注我~](https://github.com/travelclover)  
最后，感谢大家的阅读，如有错误，还请各位批评指正。
