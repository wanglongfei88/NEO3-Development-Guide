﻿# RPC

<!-- TOC -->
- [API 参考](#API-参考)
  - [修改配置文件](#修改配置文件)
  - [监听端口](#监听端口)
  - [命令列表](#命令列表)
  - [GET 请求示例](#GET-请求示例)
  - [POST 请求示例](#POST-请求示例)
- [测试工具](#测试工具)
- [其它](#其它)
<!-- TOC -->

## NEO3变更部分

- 更新
    - 调用方式：[getblockheader](api/getblockheader.md)，[getrawmempool](api/getrawmempool.md)
    - 返回结果：[getblock](api/getblock.md)，[getblockheader](api/getblockheader.md)，[getrawtransaction](api/getrawtransaction.md)，[getversion](api/getversion.md)，[getcontractstate](api/getcontractstate.md)

- 删除
    - `claimgas`, `dumpprivkey`, `getaccountstate`, `getapplicationlog`, `getassetstate`, `getbalance`, `getclaimable`, `getmetricblocktimestamp`, `getnep5balances`, `getnep5transfers`, `getnewaddress`, `gettxout`, `getunclaimed`, `getunclaimedgas`, `getunspents`, `getwalletheight`, `importprivkey`, `invoke`, `listaddress`, `sendfrom`, `sendtoaddress`, `sendmany` 


## API 参考

每个 Neo 节点 Neo-CLI 都可选的提供了一套 API 接口，用于从节点获取区块链数据，使得开发区块链应用变得十分方便。接口通过 [JSON-RPC](http://wiki.geekdream.com/Specification/json-rpc_2.0.html) 的方式提供，底层使用 HTTP/HTTPS 协议进行通讯。要启动一个提供 RPC 服务的节点，可运行以下命令：

`dotnet neo-cli.dll /rpc`

### 配置Neo-Cli

* **HTTPS 配置**：若要通过 HTTPS 的方式访问 RPC 服务器，需要在启动节点前修改配置文件 `config.json`，并设置域名、证书和密码：

  ```json
  {
    "ApplicationConfiguration": {
      "Paths": {
        "Chain": "Chain"
      },
      "P2P": {
        "Port": 10333,
        "WsPort": 10334
      },
      "RPC": {
        "Port": 10331,
        "SslCert": "YourSslCertFile.xxx",
        "SslCertPassword": "YourPassword"
      }
    ...
  ```

* **默认打开钱包:** 如果要调用与钱包相关的 API，也需要先修改配置文件 `config.json`，将:
  - unlockwallet 改为 true 的状态，
  - 并填写对象钱包的文件名和密码，如下所示：
  ```json
  ...
  "UnlockWallet": {
        "Path": "YourWallet.json",
        "Password": "YourPassword",
        "StartConsensus": false,
        "IsActive": true
      }
  ...
  ```

  完成配置后打开 NEO-CLI，客户端会在同步到最新区块后自动打开已配置的钱包并进行钱包索引同步。

### 监听端口

JSON-RPC 服务器启动后，会监听 TCP 端口，默认端口如下。P2P 和 WebSocket 的端口详见 [Neo 节点介绍](https://docs.neo.org/zh-cn/node/introduction.html)。

|                | 主网（Main Net） | 测试网（Test Net） |
| -------------- | ------------ | ------------- |
| JSON-RPC HTTPS | 10331        | 20331         |
| JSON-RPC HTTP  | 10332        | 20332         |

### 命令列表
> **NEO3 变更**：
>
>调用方式更新：getblockheader，getrawmempool
>
>返回结果更新：getblock，getblockheader，getrawtransaction，getversion，getcontractstate

| 方法                                       | 参数                                       | 说明                           | 备注       |
| ---------------------------------------- | ---------------------------------------- | ---------------------------- | -------- |
| [getbestblockhash](api/getbestblockhash.md) |                                          | 获取主链中高度最大的区块的散列              |          |
| [getblock](api/getblock.md)              | \<hash> [verbose=0]                      | 根据指定的散列值，返回对应的区块信息           |          |
| | \<index> [verbose=0]                     | 根据指定的索引，返回对应的区块信息            |          |
| [getblockcount](api/getblockcount.md)    |                                          | 获取主链中区块的数量                   |          |
| [getblockhash](api/getblockhash.md)      | \<index>                                 | 根据指定的索引，返回对应区块的散列值           |          |
| [getblockheader](api/getblockheader.md) | \<hash> [verbose=0] | 根据指定的散列值，返回对应的区块头信息。 | |
| | \<index> [verbose=0] | 根据指定的索引，返回对应的区块头信息。 | |
| [getblocksysfee](api/getblocksysfee.md)  | \<index>                                 | 根据指定的索引，返回截止到该区块前的系统手续费      |          |
| [getconnectioncount](api/getconnectioncount.md) |                                          | 获取节点当前的连接数                   |          |
| [getcontractstate](api/getcontractstate.md) | \<script_hash>                           | 根据合约脚本散列，查询合约信息              |          |
| [getpeers](api/getpeers.md)              |                                          | 获得该节点当前已连接/未连接的节点列表          |          |
| [getrawmempool](api/getrawmempool.md)    | [shouldGetUnverified=0]         | 获取内存中未确认的交易列表                |          |
| [getrawtransaction](api/getrawtransaction.md) | \<txid> [verbose=0]                      | 根据指定的散列值，返回对应的交易信息           |          |
| [getstorage](api/getstorage.md)          | \<script_hash>  \<key>                   | 根据合约脚本散列和存储的 key，返回存储的 value |          |
| [gettransactionheight](api/gettransactionheight.md) | \<txid> | 获取交易高度。 | |
| [getvalidators](api/getvalidators.md) | | 查看当前共识节点的信息 | |
| [getversion](api/getversion.md)          |                                          | 获取查询节点的版本信息                  |          |
| [invokefunction](api/invokefunction.md)  | \<script_hash>  \<operation>  \<params>  | 以指定的脚本散列值调用智能合约，传入操作及参数      |          |
| [invokescript](api/invokescript.md)      | \<script>                                | 通过虚拟机运行脚本并返回结果               |          |
| [listplugins](api/listplugins.md) | | 列出节点已加载的所有插件。                           |  |
| [sendrawtransaction](api/sendrawtransaction.md) | \<hex>                                   | 广播交易                         |          |
| [submitblock](api/submitblock.md) | \<hex>                                   | 提交新的区块                       | 需要成为共识节点 |
| [validateaddress](api/validateaddress.md) | \<address>                               | 验证地址是否是正确的 Neo 地址            |          |


### GET 请求示例

一次典型的 JSON-RPC GET 请求格式如下：

下面以获取主链中区块的数量方法为例。

请求 URL：

```
http://somewebsite.com:10332?jsonrpc=2.0&method=getblockcount&params=[]&id=1
```

发送请求后，将会得到如下的响应：

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": 909129
}
```

### POST 请求示例

一次典型的 JSON-RPC Post 请求的格式如下：

下面以获取主链中区块的数量方法为例。

请求 URL：

```
http://somewebsite.com:10332
```

请求 Body：

```json
{
  "jsonrpc": "2.0",
  "method": "getblockcount",
  "params": [],
  "id": 1
}
```

发送请求后，将会得到如下的响应：

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": 909122
}
```

> 注：当使用离线同步包同步区块时，程序可能无法响应 API 请求，建议将区块同步到最新高度后再使用 API，否则返回的结果可能不是最新的。

## 测试工具

你可以用 Chrome 扩展程序中的 Postman 来方便地进行测试（安装 Chrome 扩展程序需要科学上网），下面是测试截图：

![](../../images/api_3.jpg)


## 其它

[C# JSON-RPC 使用方法](https://github.com/chenzhitong/CSharp-JSON-RPC/blob/master/json_rpc/Program.cs)

*点击[此处](../../en/RPC)查看RPC英文版*