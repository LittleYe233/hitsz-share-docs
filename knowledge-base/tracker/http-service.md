# Hikaru Share Documentation - Knowledge Base - Tracker - Requests and Responses

## 简介

本文件系 "Hikaru Share" 项目有关 Tracker 的 HTTP/HTTPS 服务的相关知识.

## 框架

在开发细节的[相关章节](/development-details.md#Tracker)中, 有提到:

> Tracker 本质上是一种可以接受 HTTP GET 请求的 HTTP/HTTPS 服务. BitTorrent 客户端需要定期向 Tracker 发送 HTTP GET 请求, 其中包含对于该种子的详细信息以及客户端的相关统计数据, Tracker 在接收请求后, 通过一系列操作 (例如数据库操作) 向客户端返回一个应答 (response) , 其中包含该种子对应的节点 (peers) 列表.

因此小组考虑使用 NodeJS 作为服务端实现一个 [RESTful](https://restfulapi.net/) 的 HTTP/HTTPS 服务, 以满足 Tracker 的需求.

## 请求 (request)

Tracker 需要获取 HTTP GET 请求中的参数来获取种子和客户端的信息以及授权用户的凭证.

### 目标地址 (announce URL)

目标地址, 即 BitTorrent 客户端所发送的请求指向的网络位置.

一般格式为 `<协议>://<域名>[:<端口>][/路径]` , 其中:

- `<协议>`: 可以是 `http(s)` 或 `udp`, 取决于 Tracker 服务的类型. 本项目中使用 `http` 或 `https` (如果已配置 SSL 证书).
- `<域名>`: Tracker 服务使用的域名. 本项目中使用符合惯例的 `tracker.<domain>`.
- `[:端口]`: **可选项.** Tracker 服务器绑定的端口号. 本项目中使用默认值 `80` 或 `443` (如果已配置 SSL 证书).
- `[/路径]`: **可选项.** 进一步指向目标网络位置的路径. 本项目中使用符合惯例的 `/announce`.

### 参数

BitTorrent 客户端向 Tracker 发送的请求应当或可选包含如下键:

- `passkey`: 授权用户的凭证, 为一个固定长度的十六进制串. **注意**该键将取代 BitTorrent 标准协议中的 `key` 键.
- `info_hash`: 元信息文件中 `info` 键值的经过编码的 (urlencoded) 20 字节长 SHA1 哈希串.
- `peer_id`: BitTorrent 客户端启动时生成的唯一的经过编码的 20 字节长字符串.
- `port`: BitTorrent 客户端监听的端口.
- `uploaded`: 从 BitTorrent 客户端向 Tracker 发送 "started" 事件开始上传总字节数的十进制表示.
- `downloaded`: 从 BitTorrent 客户端向 Tracker 发送 "started" 事件开始下载总字节数的十进制表示.
- `left`: BitTorrent 客户端使种子完成率达到 100% 所需的下载字节数的十进制表示.
- `compact`: 值为 1 表示 BitTorrent 客户端接受紧凑 (compact) 应答——节点列表改为一个节点字符串, 每个节点对应 6 个字节, 前 4 个字节表示节点的主机地址, 后 2 个字节表示端口数.
- `no_peer_id`: 表示 Tracker 将忽略节点字典中的节点 ID 字段. 若 `compact` 的值为 1 , 该选项将被忽略.
- `event`: 若存在, 则值只能为 `started` 、 `completed` 、 `stopped` 和空值 (与不存在结果相同) ；若不存在, 可忽略.
  - `started`: BitTorrent 客户端发送给 Tracker 的首条请求的 `event` 键的必须值.
  - `stopped`: 当 BitTorrent 客户端正常关闭时发送给 Tracker 的请求的 `event` 键的必须值.
  - `completed`: 当 BitTorrent 客户端下载完毕时发送给 Tracker 的请求的 `event` 键的必须值. **注意**若 BitTorrent 客户端启动时种子完成率已经达到 100% , 则 `event` 键不能为此值.
- `ip`: **可选项.** BitTorrent 客户端机器的*真实* IP 地址, 须为点分隔的 IPv4 地址或符合 [RFC 3513](https://www.rfc-editor.org/rfc/rfc3513) 标准的十六进制 IPv6 地址.
- `numwant`: **可选项.** BitTorrent 客户端欲从 Tracker 获取的节点数量. 该值允许为 0 . 若不存在, 一般默认为 50 .
- `trackerid`: **可选项.**值应与上一次声明 (announce) 中的值相同.

## 应答 (respond)

Tracker 的应答的 `Content-Type` 值应为 `text/plain` , 其内容应为经过 bencode 的字典.

### 参数

Tracker 向 BitTorrent 客户端发送的应答应当包含如下键:

- `failure reason`: 若存在, 其他键可以不存在. 值为一段人类可读字符串, 解释请求失败的原因.
- `warning message`: **可选项.**类似 `failure reason` , 但应答将被正常进行. 警告信息将被视作错误.
- `interval`: BitTorrent 客户端发送给 Tracker 下一条定期请求所需要等待的秒数.
- `min interval`: **可选项.** BitTorrent 客户端最短声明时间间隔.
- `tracker id`: BitTorrent 客户端在下一次声明中将发回的字符串.
- `complete`: 已下载完整种子的节点数量, 即 BitTorrent 客户端中的 "seeders" .
- `incomplete`: 非 seeder 的节点数量, 即 BitTorrent 客户端中的 "leechers" .
- `peers`: 节点列表, 其格式会受到请求中 `compact` 键的影响. 若可用节点数量超过请求中 `numwant` 键的值或默认值 (50), Tracker 应当自行选择其中部分节点.
  - 字典模型:
    - `peer id`: 在请求中定义的节点 ID .
    - `ip`: 节点的 IPv4 地址、 IPv6 地址或 DNS 名称.
    - `port`: 节点的 BitTorrent 客户端监听的端口.
  - 二进制模型: 一个节点字符串, 每个节点对应 6 个字节, 前 4 个字节表示节点的主机地址, 后 2 个字节表示端口数.

## 参考文献

1. [BitTorrentSpecification](https://wiki.theory.org/BitTorrentSpecification#Tracker_HTTP.2FHTTPS_Protocol).TheoryOrg.
2. [TBSource/announce.php at master · QwertyRider/TBSource](https://github.com/QwertyRider/TBSource/blob/master/announce.php).GitHub.