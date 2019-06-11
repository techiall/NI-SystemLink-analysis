## SystemLink 软件分析

### 声明

本项目只提供方法，不提供相关工具。



### 你可以获得什么信息

* SystemLink 各个组件的 API
* SystemLink 使用的数据库
* 获得 SystemLink 数据库的数据 （这个才是重点
* 我的个人分析



### SystemLink API

SystemLink API 文档就放在 GitHub，是官方自己开放的，链接如下：

[https://github.com/ni/systemlink-OpenAPI-documents](https://github.com/ni/systemlink-OpenAPI-documents)

一个文件夹代表 SystemLink 的组件（如 tag，file ...），yml 中就包含了 API 相关信息。yml 使用 swagger 生成的，可以将 yml 转换成 pdf 等格式。

从 API 接口中我们可以发现 SystemLink 是使用 C# 写的，从时间戳那里发现的。



我们有了 API 文档，为啥还要找 SystemLink 的数据库呢，因为 API 返回的数据，绝大部分是我们不需要的；我们如果可以找到数据库，要是知道了数据库的结构，那我们可以通过数据库查询返回我们需要的数据即可。相比 API，可定制性更强。（主要是 API 需要验证，需要 session，真麻烦。



只要拿到了数据库，那相当于拿到整套 SystemLink，我们也就可以在其基础上做一些二次开发；吾，一个远大的计划呼之欲出，我们可以让所有 SystemLink 接入自己二次开发的平台。



### 寻找 SystemLink 数据库位置

>  SystemLink 是无法选择安装位置的，因此安装位置应该不会有所改变，以下相关配置文件的位置也应该不会改变。



在一台刚重装后的 windows 10 上安装 SystemLink。打开 `任务管理器` ，找到 `详细信息` 。扫了一眼，发现了 mongod 和 redis-server， 在 `详细信息` 的子列，右键选择 `选择列` ，勾选 `命令行` 。而且这两者命令行路径和 `ni` 有关，SystemLink 就用了这两个数据库。

找到 mongod.exe，一看命令行，mongodb 的配置文件路径找到了，那数据库位置和密码还远吗？

```shell
"C:\Program Files\National Instruments\Shared\Skyline\NoSqlDatabase\bin\mongod.exe" --config "C:\ProgramData\National Instruments\Skyline\NoSqlDatabase\mongodb.conf"
```



C:\ProgramData\National Instruments\Skyline\NoSqlDatabase`  文件夹就是 mongo 的数据，也是 SystemLink 的数据！

回到上一级目录 `C:\ProgramData\National Instruments\Skyline` 。这个目录就是 SystemLink 系统基本组件。包括了SystemLink 各模块配置，mongo， 证书，redis，消息队列等等，路径下的文件如下：

```shell
$ ls
Certificates/  HttpConfigurations/  Logs/           SkylineConfigurations/
Config/        Install/             NoSqlDatabase/  tracelogger.cfg
Data/          KeyValueDatabase/    RabbitMQ/
```



在 `C:\ProgramData\National Instruments\Skyline\Config` 目录中，可以看到很多 SystemLink 对每个模块的配置文件，其中我们找到了 redis 数据库配置文件（KeyValueDatabase.json）。

```shell
$ ls
AlarmInstance.json               Smtp.json
AssetPerformanceManagement.json  Smtp.json.defaults
AuditTrail.json                  SystemsStateManager.json
CloudConnector.json              TagHistorian.json
DashboardBuilder.json            TagIngestion.json
FileIngestion.json               TagRuleEngine.json
FileIngestion.json.defaults      TagRuleEngineRules/
FileMoving.json                  TagSubscription.json
KeyValueDatabase.json            TDMReader.json
NoSqlDatabase.json               TestMonitor.json
Notification.json                UserData.json
Repository.json                  UserData.json.defaults
Services/
```



拿到了数据库的权限，我们就可以根据 SystemLink 找到各个组件存放的数据库。从而可以写代码获得相关数据。

我们可以通过写脚本，从刚才的路径自动获得密码和端口。

以下是总结，`消息队列` 和 `邮件提醒` 模块还没了解。

| 数据库 | 端口  | 说明                           |
| ------ | ----- | ------------------------------ |
| mongo  | 27018 | 密码见 `NoSqlDatabase.conf`    |
| redis  | 6378  | 密码见 `KeyValueDatabase.json` |

> mongo 连接数据库命令
>
> `mongo --port 27018 -u <username> -p <password> `
>
> redis 连接数据库命令
>
> `redis-cli -h <host> -p 6378 -a <password>`



### 连接数据库

使用命令行连接数据库

mongo 数据库如下：

```shell
> show databases;
admin               0.000GB
local               0.000GB
nialarms            0.000GB
niapm               0.000GB
niaudittrail        0.002GB
nidashboardbuilder  0.000GB
nifile              0.000GB
ninotification      0.000GB
nirepo              0.000GB
nisalt              0.001GB
nitag               0.000GB
nitaghistorian      0.041GB
nitagrule           0.000GB
nitestmonitor       0.001GB
niuserdata          0.000GB
```



redis 数据库中存放着所有 tag 的缓存。

| key                   | 说明                               |
| --------------------- | ---------------------------------- |
| tagInfoCurrentValues* | tag 当前数值，路径，时间戳和平均值 |
| tagInfoJson*          | tag 基本信息。                     |
| tagInfoPaths:         | 所有 tag 路径列表                  |

> 上表是通过 tag 模块中的数据分析得来的。



### 个人分析

以上这些都在两台安装了 SystemLink 软件的电脑上测试过了。我们可以通过写个脚本。自动获得端口和密码。并配置在自己二次开发的项目上。 排开 SystemLink 自身带的一些权限认证模块，个人感觉 SystemLink 最主要作用的就是 NI 硬件发送到服务器的数据，以下的分析和 tag 有关。

redis 中保存着每个 tag 的最新收到的数据，如果没有将 tag 的数据设置 `留存`，他只会存放在 redis，不会存在 mongo 中。

一旦 tag 数据设置 `留存`，就会在 mongo 中创建一个集合。集合名称是以 `Metadata_ ` 开头。tag 需要留存的数据，会在 `PathMap` 中建立映射关系，关系如下：

```json
{
	"_id" : ObjectId("123123411bf94013a47b1234"),
	"Path" : "Health.CPU.0.UsePercentage",
	"CollectionId" : ObjectId("123829811bf9401555557488")
}
```

`Path`  和 `CollectionId ` 是一一对应，我们可以通过 Path 找到 CollectionId，从而在 `Metadata_123829811bf9401555557488` 中找到这个 Path 的所有数据。



 `/nitag/v2/tags-with-values` 接口数据在 mongo 中的位置，发现找不到，原来是缓存在 redis 中，通过分析 redis 数据库，发现该接口返回的数据和 redis 中的一一对应。怪不得 tag 数据量那么多，速度还那么快。



> 等有时间整理一下之前写的项目，将自己写的一个 SpringBoot 读取 SystemLink 数据库的项目开源。