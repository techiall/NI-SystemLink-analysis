## SystemLink 软件分析

### 声明

本项目只提供方法，不提供相关工具。



### 你可以获得什么信息

* SystemLink 各个组件的 API
* SystemLink 使用的数据库
* 获得 SystemLink 数据库的数据 （这个才是重点



### SystemLink API

SystemLink API 文档就放在 GitHub，是官方自己开放的，链接如下：

[https://github.com/ni/systemlink-OpenAPI-documents](https://github.com/ni/systemlink-OpenAPI-documents)

一个文件夹代表 SystemLink 的组件（如 tag，file ...），yml 中就包含了 API 相关信息。yml 使用 swagger 生成的，可以将 yml 转换成 pdf 等格式。



### SystemLink 数据库

> 我们有了 API 文档，为啥还要找 SystemLink 的数据库呢，因为 API 返回的数据，绝大部分是我们不需要的；我们如果可以找到数据库，要是知道了数据库的结构，那我们可以通过数据库查询返回我们需要的数据即可。相比 API，可定制性更强。
>
> （主要是 API 需要验证，需要 session，真麻烦。
>
> 只要拿到了数据库，那相当于拿到整套 SystemLink，我们也就可以在其基础上做一些二次开发；吾，一个远大的计划呼之欲出，我们可以让所有 SystemLink 接入自己二次开发的平台。



SystemLink 是无法选择安装位置的，因此安装位置应该不会有所改变。



在一台刚重装后的 windows 10 上安装 SystemLink。打开 `任务管理器` ，找到 `详细信息` 。扫了一眼，发现了 mongod 和 redis-server，在 `详细信息` 的子列，右键选择 `选择列` ，勾选 `命令行` 。

找到 mongod.exe，一看命令行 `"C:\Program Files\National Instruments\Shared\Skyline\NoSqlDatabase\bin\mongod.exe" --config "C:\ProgramData\National Instruments\Skyline\NoSqlDatabase\mongodb.conf` 。配置文件找到了，那数据库位置和密码还远吗？

进入 `C:\ProgramData\National Instruments\Skyline\NoSqlDatabase`  文件夹，mongo 的数据和配置都看到了。

SystemLink 开放的 mongo 端口为 27018，密码和数据库文件存放位置都写在 `mongodb.conf` 中。使用 Mongodb GUI 连接测试，连接成功。



会到上一级目录，`C:\ProgramData\National Instruments\Skyline` 。这个目录就是 SystemLink 系统的配置目录。里面包括 证书，每个模块各自配置，redis，消息队列等等。

在 `C:\ProgramData\National Instruments\Skyline\Config` 目录中，我们找到了 redis 数据库配置文件（KeyValueDatabase.json）。端口为 6378。使用 redis GUI 进行连接，连接成功。



拿到了数据库的权限，我们就可以根据 SystemLink 找到各个组件存放的数据库。从而可以写代码获得相关数据。

排开 SystemLink 自身带的一些权限认证，SystemLink 最主要的就是 NI 硬件发送数据，用 tag 接收数据。
