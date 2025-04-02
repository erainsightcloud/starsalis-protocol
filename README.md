# Starsalis

Starsalis是由李凌翌（Github：[@contiunity](https://github.com/contiunity)）设计的一种去中心化协议，用于承载《碧蓝航线》官方的百合向讨论社区。

## 实现

目前没有任何公共实现（黄鸡的官方实现是邀请制且需要实名认证）。

你可以自己实现一个，然后提issue或者pr，将你的实现加到这篇文档中。

## 1. 用户寻址

### 第一步

Starsalis通过以太坊ENS域名标识用户。具体而言，用户需要在ENS网页中添加一个`/starsalis-wellknown.json`，内容格式如下：
```
{
  "server": "<PDS服务器URL>",
  "user": <用户编号>,
  "addr": [
    "<用于签名的地址>"
  ]
}
```
例如：
```json
{
  "server": "https://pds.example.com/pds.php",
  "user": 1,
  "addr": [
    "0x3447fF...696499E"
  ]
}
```
### 第二步

向PDS服务器发送如下请求：

```
GET <PDS服务器URL>?type=user&user=<用户编号>
```
服务器应当返回一个JSON字典（如果添加`mp=1`项，则是messagepack），每一项为用户的一条信息。

信息如果以`$`开头，且以`$`结尾，则代表变量，否则代表用户信息栏的自定义信息。非变量的自定义信息必须为string，变量的数据类型取决于变量的性质。

用户变量定义列表：
| 变量名 Name | 数据类型 Type | 解释 Description |
| --- | --- | --- |
| system.ens | string | ENS域名 |
| system.webhook | string | 互动WebHook接口 |
| nickname | string | 用户昵称 |
| email | array<string>\[2] | 电子邮件 |
| phone | number | 电话号码 |
| phone.area | number | 电话国家/地区号 |
| social.ap | array<string>\[2] | ActivityPub用户名 |
| social.qq | number | QQ号 |

其中，`system.ens`变量是极其特殊的，其必须存在且与用户的ENS域名相同，否则视为用户无效。

如果没有`system.webhook`变量，代表用户不接受任何互动。

示例：
```json
{
  "$system.ens$": "alice.eth",
  "$system.webhook$": "https://pds.example.com/hook.php?user=alice",
  "$nickname$": "Alice Corplic Li",
  "$email$": ["alice", "example.org"],
  "$social.qq$": 10002,
  "$social.ap$": ["alice", "mastodon.social"]
}
```
## 2. 帖子信息

向PDS服务器发送如下请求：

```
GET <PDS服务器URL>?type=author&user=<用户编号>&page=<页码>
```
其中，页码从0开始。服务器返回一个`list<string>[*]`（字符串组成的JSON列表），每个字符串代表帖子的IPFS QID。

帖子是IPFS中的一个普通文件，为Protobuf格式，定义：
```proto
message Post {
  uint64 timeseq = 1; // 发送时间戳（秒级）
  string sender = 2; // 发送者ENS域名
  string data = 3; // 发送的内容（HTML）
  bytes sign = 4; // 签名
}
```
其中，签名的格式如下：
```
Sign = ECDSA( SHA3( "CubNsVVe25" + SHA3(data) + SHA3(timeseq) + SHA3(sender) ) )
```
在获取帖子后，应该请求PDS，检查帖子是否已被删除；

```
GET <PDS服务器URL>?type=post&user=<用户编号>&page=<帖子QID>
```

这个端点返回一个JSON：
```json
{
  "status": "<状态>"
}
```
| 状态 State | 定义 Define |
| --- | --- |
| normal | 这个帖子正常存在。 |
| deleted | 这个帖子已经被用户自己删除了。 |
| rejected | 这个帖子不符合用户所在PDS的使用政策。 |
| replaced | 这个帖子有错误，被另一条修正后的帖子代替。<br/>（返回json中添加一个qid项，表示新的帖子的qid）。 |
| notfound | PDS根本不知道这个帖子的存在。 |

## 3. 互动

向相关用户的Webhook地址发送如下JSON：
```
{
  "type": "<互动类型>",
  "from": "<原用户>",
  "target": "<目标>",
  "signature": "<签名Base64>"
}
```
其中，目标可以是用户（ENS域名）或帖子（QID）。
| 互动类型 Action | 目标类型 Target | 定义 Define |
| --- | --- | --- |
| subscribe | 用户 | 关注用户 |
| unsubscribe | 用户 | 取关用户 |
| block | 用户 | 声明屏蔽用户 |
| like | 帖子 | 点赞 |
| unlike | 帖子 | 取消点赞 |

签名算法如下：

```
Sign = ECDSA( SHA3( "PAm1V9AkSSv" + SHA3(action_type) + SHA3(sender) + SHA3(target) ) )
```

## 4. 私聊

### 第一步

发起者向接受者的Webhook地址发送如下JSON：
```
{
  "type": "secretchat:create",
  "from": "<发起者ENS>",
  "matrix": "<发起者Matrix>",
  "target": "<接受者ENS>",
  "signature": "<签名Base64>",
  "proofsalt": "<盐>"
}
```

签名算法如下：

```
Sign = ECDSA( SHA3( "1CkVAaeQ" + SHA3(founder_ens) + SHA3(founder_matrix) + SHA3(acceptor) + proofsalt ) )
```

### 第二步

接受者向发起者的Webhook地址发送如下JSON：

```
{
  "type": "secretchat:accept",
  "from": "<接受者ENS>",
  "matrix": "<接受者Matrix>",
  "target": "<发起者ENS>",
  "id": "<盐的SHA3哈希>",
  "signature": "<签名Base64>"
}
```

签名算法如下：

```
Sign = ECDSA( SHA3( "2kVAdmC9kqY" + SHA3(founder) + SHA3(acceptor_ens) + SHA3(acceptor_matrix) + proofsalt ) )
```

### 第三步

双方执行常规的Matrix发起私聊流程。

## 5. 统计

### 5.1. 源信息披露

向PDS服务器发送如下请求：

```
GET <PDS服务器URL>?type=user&user=<用户编号>&bigdata=1&mp=1&page=<页码>
```

大数据模式（`bigdata=1`参数）会使PDS返回几个特殊变量。为了节约带宽，大数据模式必须启用messagepack（即`mp=1`），否则是非法请求。该模式添加的变量是分页的，通过`page`参数选择页面，页码从0开始。

| 变量名 Name | 数据类型 Type | 解释 Description |
| --- | --- | --- |
| v.subscribe | array\<string>\[*] | 关注的用户 |
| v.subscribe | array\<string>\[*] | 屏蔽的用户 |
| v.likes | array\<string\>\[*] | 点赞的帖子 |

### 5.2. 统计接口

用户自己选择统计服务，用于获取各种数据。统计服务独立于Starsalis，可以自行选择是否遵守下面的接口。

#### 5.2.1. 用户关系列表

```
GET <统计API端点>?key=<API密钥>&id=<用户ID>&rel=<关系类别>&page=<页码>
```
| 关系类别 Type | 解释 Description |
| --- | --- |
| subscribe | 该用户关注的用户 |
| fans | 关注该用户的用户 |
| blocks | 该用户拉黑的用户 |
| bads | 拉黑该用户的用户 | 

返回结果：

```
{
  "sect_sum": <条目总数>,
  "probab_sum": <统计方估计的实际总数>,
  "pages": <总页数>,
  "data": [
    "<用户ENS地址>" ...
  ]
}
```

例如：
```
{
  "sect_sum": 3,
  "probab_sum": 2,
  "pages": 1,
  "data": [
    "alice.eth",
    "a.alice.eth",
    "bob.eth"
  ]
}
```

#### 5.2.1. 帖子点赞者列表

```
GET <统计API端点>?key=<API密钥>&id=<帖子CID>&rel=like&page=<页码>
```
返回结果：

```
{
  "sect_sum": <条目总数>,
  "probab_sum": <统计方估计的实际总数>,
  "pages": <总页数>,
  "data": [
    "<用户ENS地址>" ...
  ]
}
```
