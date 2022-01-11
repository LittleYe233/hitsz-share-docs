# Hikaru Share Documentation - Knowledge Base - Tracker - HTTP Services

## 简介

本文件系 "Hikaru Share" 项目有关 Tracker 的 HTTP/HTTPS 服务的相关知识.

## 框架

在开发细节的[相关章节](/development-details.md#Tracker)中, 有提到:

> Tracker 本质上是一种可以接受 HTTP GET 请求的 HTTP/HTTPS 服务. BitTorrent 客户端需要定期向 Tracker 发送 HTTP GET 请求, 其中包含对于该种子的详细信息以及客户端的相关统计数据, Tracker 在接收请求后, 通过一系列操作 (例如数据库操作) 向客户端返回一个应答 (response) , 其中包含该种子对应的节点 (peers) 列表.

因此项目考虑使用 NodeJS 作为服务端实现一个 [RESTful](https://restfulapi.net/) 的 HTTP/HTTPS 服务, 以满足 Tracker 的需求.

## 处理数据

### URL 编码 (urlencode)

一般 URL 会包含一些特殊字符, 可能会给后续的解析和序列化等操作带来麻烦, 因此 BitTorrent 客户端发送给 Tracker 的请求中应当将可能出现的特殊字符编码为受支持的字符串. 此过程可逆. 具体转换规则如下: 对于任何非数字, 字母, ".", "-", "_" 和 "~" 的单字节字符将被编码为形如 `%<二位十六进制字符串>` 的字符串, 其中 `<二位十六进制的字符串>` 为原字符的值的十六进制形式. 多字节字符 (例如: 汉字) 将对其中的每个字节作类似处理. 例如: `%124Vx%9A%BC%DE%F1%23Eg%89%AB%CD%EF%124Vx%9A` 表示字符串 "\x12\x34\x56\x78\x9a\xbc\xde\xf1\x23\x45\x67\x89\xab\xcd\xef\x12\x34\x56\x78\x9a".

相关标准可参阅 [RFC 1738](https://www.rfc-editor.org/rfc/rfc1738) 文档.

### Bencode

这是一种数据结构序列化的方式, 支持字符串, 整数, 列表, 字典四种数据类型或容器类型. 根据给定的转换规则, 可以将数据结构经过 Bencode 处理, 或将经过 Bencode 处理的字符串还原为原有数据结构. 具体转换规则如下:

- 字符串: `<字符串长度的十进制表示>:<字符串>`. 例如: `4:spam` 表示字符串 "spam", `0:` 表示字符串 "".
- 整数: `i<十进制整数>e`. **注意**非零整数不可带有前导零, 且零不可表为 `i-0e`. 例如: `i3e` 表示整数 "3", `i-3e` 表示整数 "-3".
- 列表: `l<经过 Bencode 处理的元素>e`. 例如: `l1:1i2eli3eed1:4i4eee` 表示列表 "\['1', 2, \[3\], {'4': 4}\]", `le` 表示列表 "\[\]".
- 字典: `d<经过 Bencode 处理的字符串>:<经过 Bencode 处理的元素>`. 例如: `d1:11:11:2i2e1:3li3ee1:4d1:51:5ee` 表示字典 "{'1': '1', '2': 2, '3': \[3\], '4': {'5': '5'}}", `de` 表示字典 "{}".

Bencode 相关的序列化工具已有较成熟实现, 请参见 GitHub [相关页面](https://github.com/benjreinhart/bencode-js).

## 请求 (request)

Tracker 需要获取 HTTP GET 请求中的参数来获取种子和客户端的信息以及授权用户的凭证.

### 目标地址 (announce URL)

目标地址, 即 BitTorrent 客户端所发送的请求指向的网络位置.

一般格式为 `<协议>://<域名>[:<端口>][/路径]`, 其中:

- `<协议>`: 可以是 `http(s)` 或 `udp`, 取决于 Tracker 服务的类型. 本项目中使用 `http` 或 `https` (如果已配置 SSL 证书).
- `<域名>`: Tracker 服务使用的域名. 本项目中使用符合惯例的 `tracker.<domain>`.
- `[:端口]`: **可选项.** Tracker 服务器绑定的端口号. 本项目中使用默认值 `80` 或 `443` (如果已配置 SSL 证书).
- `[/路径]`: **可选项.** 进一步指向目标网络位置的路径. 本项目中使用符合惯例的 `/announce`.

### 参数

BitTorrent 客户端向 Tracker 发送的请求应当或可选包含如下键:

- `passkey`: 授权用户的凭证, 为一个长度为 16 的十六进制串. **注意**该键将取代 BitTorrent 标准协议中的 `key` 键.
- `info_hash`: 元信息文件中 `info` 键值的经过 URL 编码的 (urlencoded) 20 字节长 SHA1 哈希串.
- `peer_id`: BitTorrent 客户端启动时生成的唯一的经过编码的 20 字节长字符串. 有关其的详细信息参见后文[相关章节](#peer_id-参数).
- `port`: BitTorrent 客户端监听的端口.
- `uploaded`: 从 BitTorrent 客户端向 Tracker 发送 "started" 事件开始上传总字节数的十进制表示.
- `downloaded`: 从 BitTorrent 客户端向 Tracker 发送 "started" 事件开始下载总字节数的十进制表示.
- `left`: BitTorrent 客户端使种子完成率达到 100% 所需的下载字节数的十进制表示.
- `compact`: 值为 1 表示 BitTorrent 客户端接受紧凑 (compact) 应答——节点列表改为一个节点字符串, 每个节点对应 6 个字节, 前 4 个字节表示节点的主机地址, 后 2 个字节表示端口数.
- `no_peer_id`: 表示 Tracker 将忽略节点字典中的节点 ID 字段. 若 `compact` 的值为 1, 该选项将被忽略.
- `event`: 若存在, 则值只能为 `started`, `completed`, `stopped` 和空值 (与不存在结果相同); 若不存在, 可忽略.
  - `started`: BitTorrent 客户端发送给 Tracker 的首条请求的 `event` 键的必须值.
  - `stopped`: 当 BitTorrent 客户端正常关闭时发送给 Tracker 的请求的 `event` 键的必须值.
  - `completed`: 当 BitTorrent 客户端下载完毕时发送给 Tracker 的请求的 `event` 键的必须值. **注意**若 BitTorrent 客户端启动时种子完成率已经达到 100% , 则 `event` 键不能为此值.
- `ip`: **可选项.** BitTorrent 客户端机器的*真实* IP 地址, 须为点分隔的 IPv4 地址或符合 [RFC 3513](https://www.rfc-editor.org/rfc/rfc3513) 标准的十六进制 IPv6 地址.
- `numwant`: **可选项.** BitTorrent 客户端欲从 Tracker 获取的节点数量. 该值允许为 0. 若不存在, 一般默认为 50.
- `trackerid`: **可选项.** 值应与上一次声明 (announce) 中的值相同.

### `peer_id` 参数

`peer_id` 是 BitTorrent 客户端启动时生成的唯一的经过编码的 20 字节长字符串, 主要包含 BitTorrent 客户端种类和版本信息和区别于其他 BitTorrent 客户端的随机字符串. 目前被主要使用的两种标准为 Azureus 风格和 Shadow's 风格. 为保证项目代码的简洁性, 项目的 Tracker 将仅支持 Azureus 风格的 `peer_id`, 即便仍有少数 Tracker 服务器和客户端支持 Shadow's 风格.

Azureus 风格的 `peer_id` 形如 `-<两位客户端 ID 字符串><四位数字版本号>-<十二位随机数字标识串>`, 例如: `-AZ2060-0123456789AB` 表示一个版本号 2.0.6, 标识串为 "0123456789AB" 的 Azureus 客户端. 使用此种风格的常见客户端及对应的 ID 字符串如下表:

| 客户端 | ID 字符串 |
| :-: | :-: |
| Azureus | AZ |
| BitComet | BC |
| BitTorrent Pro | BP |
| libtorrent | LT |
| libTorrent | lt |
| qBittorrent | qB |
| Transmission | TR |
| μTorrent for Mac | UM |
| μTorrent | UT |

Tracker 可以检查请求中的该键的值来限定客户端的种类.

## 应答 (respond)

Tracker 的应答的 `Content-Type` 值应为 `text/plain`, 其内容应为经过 bencode 的字典.

### 参数

Tracker 向 BitTorrent 客户端发送的应答应当包含如下键:

- `failure reason`: 若存在, 其他键可以不存在. 值为一段人类可读字符串, 解释请求失败的原因.
- `warning message`: **可选项.** 类似 `failure reason`, 但应答将被正常进行. 警告信息将被视作错误.
- `interval`: BitTorrent 客户端发送给 Tracker 下一条定期请求所需要等待的秒数.
- `min interval`: **可选项.** BitTorrent 客户端最短声明时间间隔.
- `tracker id`: BitTorrent 客户端在下一次声明中将发回的字符串.
- `complete`: 已下载完整种子的节点数量, 即 BitTorrent 客户端中的 "seeders".
- `incomplete`: 非 seeder 的节点数量, 即 BitTorrent 客户端中的 "leechers".
- `peers`: 节点列表, 其格式会受到请求中 `compact` 键的影响. 若可用节点数量超过请求中 `numwant` 键的值或默认值 (50), Tracker 应当自行选择其中部分节点.
  - 字典模型:
    - `peer id`: 在请求中定义的节点 ID.
    - `ip`: 节点的 IPv4 地址, IPv6 地址或 DNS 名称.
    - `port`: 节点的 BitTorrent 客户端监听的端口.
  - 二进制模型: 一个节点字符串, 每个节点对应 6 个字节, 前 4 个字节表示节点的主机地址, 后 2 个字节表示端口数.

## 抓取 (scrape) 惯例

惯例上, 绝大多数 Tracker 支持 "抓取" (scrape) 这一形式的请求. 此类请求是 HTTP GET 请求, 欲查询 Tracker 正在管理的某一指定的或所有的种子的状态.

### 目标地址 (scrape URL)

支持此类请求的 Tracker 需要允许 BitTorrent 客户端将请求发送到符合一定要求的目标地址 (scrape URL) 上, 本项目使用 `<协议>://<域名>[:<端口>]/scrape`. Tracker 的应答的 `Content-Type` 值应为 `text/plain`, 其内容应为经过 bencode 的字典, 且可选择对应答纯文本进行 gzip 压缩.

### 请求参数

BitTorrent 客户端向 Tracker 发送的请求应当或可选包含如下键:

- `passkey`: 授权用户的凭证, 为一个长度为 16 的十六进制串.
- `info_hash`: **可选项.** 元信息文件中 `info` 键值的经过 URL 编码的 (urlencoded) 20 字节长 SHA1 哈希串. 可以在一次请求中携带多组此键值对, 例如: `?info_hash=a&info_hash=b&info_hash=c`. 若不存在, 则表示查询 Tracker 所有管理的种子.

### 应答参数

Tracker 向 BitTorrent 客户端发送的应答应当或可选包含如下键:

- `files`: 包含每个请求查询的种子对应状态的经过 Bencode 处理的字典. 每个键为一个种子的 `info_hash`, 对应的值也为一个经过 Bencode 处理的字典.
  - `complete`: 已下载完整种子的节点数量, 即 BitTorrent 客户端中的 "seeders".
  - `downloaded`: Tracker 登记完成 ("event=complete", 即 BitTorrent 客户端下载结束) 的次数.
  - `incomplete`: 非 seeder 的节点数量, 即 BitTorrent 客户端中的 "leechers".
  - `name`: **可选项.** 种子被种子文件的相关数据限定的内部名称.

例如: `d5:filesd20:0123456789abcdef0123d8:completei5e10:downloadedi50e10:incompletei10eeee` 表示字典 "{'files': {'0123456789abcdef0123d8': {'complete': 5, 'downloaded': 50, 'incomplete': 10}}}".

为保证项目代码的简洁性, 部分非官方的可选应答参数将被忽略, 即便仍有极少数 Tracker 服务器和客户端支持.

## 数据库交互

Tracker 需要在本地数据库中存储有关节点的相关信息才能工作, 具体的交互方式是对已有的数据库进行增, 删, 改, 查等基本操作. 对于常见的关系型数据库 (例如: MySQL), NodeJS 已有功能强大的模块可供操作. 有关数据库的相关设计, 参见开发细节的[相关章节](/development-details.md#数据库).

## 参考文献

1. [BitTorrentSpecification](https://wiki.theory.org/BitTorrentSpecification).TheoryOrg.
2. [TBSource/announce.php at master · QwertyRider/TBSource](https://github.com/QwertyRider/TBSource/blob/master/announce.php).GitHub.