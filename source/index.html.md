---
title: PKU Helper 后端指南

language_tabs:
  - shell

toc_footers:
  - 用Slate制作
  - 2017-09-03

includes:
  - errors

search: true
---

# 介绍

PKU Helper 新一代后端于2017年8月在一个僻静的机场中被创作, 主要想法源自于 [@luolc](https://github.com/luolc) 与 [@jimmielin](https://github.com/jimmielin) 的一些讨论. 其特点为:

- **可以热修复的API呼叫机制.** 基于之前提出的 [Dynamic Request Workflow](https://github.com/PKUHelper/docs/wiki/Dynamic-Local-Request-Workflow), 后端的YAML文件对于善变的北京大学API ("一堆乱七八糟的外包网页") 进行描述, 由 `/manifest/latestisa` 接口进行分发给各个客户端进行热更新, 以避免计算中心开炮对我们服务的影响.

- **基于RSA的不对称加密机制设计的API访问控制与用户识别.** 所有的上行信息采取加密的格式, 加密秘钥由后端保管, 公钥由客户端保管, 既可以对于API访问进行控制 (比如replay attack) 也可以为用户隐私的保护做铺垫 (然而众所周知我国有相关的法律规定).

- **采用node.js/express组合制成**, 满足各种新一代互联网设计爱好者的工具包需求.

# 基本访问格式

> 任何访问都需要包括如下的一些基本信息 (`preshared_public_key.pub` 的内容参见附录):


```shell
# 开发模式接受application/json, 解密POST body
curl
  -H "Content-Type: application/json"
  -X POST
  -d "{'hello': 'world'}"
  "https://api.pkuhelper/test"

# 生产模式只接受application/x.pku, 加密版POST body
'{ "hello": "world"}' | openssl rsautl -encrypt -pubin -inkey preshared_public_key.pub |
curl 
  -H "Content-Type: application/x.pku"
  -H "Authorization: 8faa2562030bc5be6b9160e6ddb3290dc0c4e1cf4fe47ae0a8d459baaa1ddd47"
  -H "X-PKU-Integrity: ..." # 之后再解释
  -X POST
  -d "$(</dev/stdin)"
  "https://api.pkuhelper/test"
```

> 注意这里采用的是未登陆的key (`preshared_public_key.pub`), 其Authorization值为public key的SHA-2 (SHA256) hash.
> 注意X-PKU-Integrity也是密文, 请用相同的key签发一个JSON Object: `{nva: ..., nvb: ...}`, NVA即 "Not Valid After", NVB即 "Not Valid Before" (请求时的unix timestamp, milliseconds), NVA-NVB值不得超过60,000ms (1 min). 此值也称为 TTL ("Time to live") 值, 为避免replay attack所用.

> 这里的访问结果应该是成功的. 无论你进行明文还是密文传输, `POST /test` 都会给你一个有趣的返回值.

PKU Helper的上行信息全部采用密文传输, 即便是公开访问的API也使用一个预设的key. 使用何种秘钥会决定访问PKU Helper后端功能的权限, 因而一旦用户登陆后即可采用用户的key来进行传输. 在开发模式中, 你可以随时伪装访问任何PKU Helper用户账号, 并直接POST JSON内容, 这样方便你进行测试. 你只需要将 `Content-Type` 选择为 `application/json`. 如果你需要伪装一个具体的用户, 则用 `X-PKUHelper-AuthOverride-ID` header 来描述目标用户ID. 注意, 开发模式的服务器是分开的. 部署服务器在任何条件下不会接受 `application/x.pku` 之外的访问类型.

有人会问, 为什么要采取 `X-PKU-Integrity` 来描述NVA/NVB值而不是封装在POST主内容内呢? 问题本身已经包含了答案 -- GET 请求中没有 request body, 而我们也不希望我们的GET请求轻易被replay.

<aside class="notice">
我们建议进行开发的时候讲负责HTTP请求的部分进行抽象封装, 因为你肯定不希望每次都写一大段RSA加密和生成 `X-PKU-Integrity` 的部分, 跟别说同名地进行开发和部署模式的切换了.
</aside>

# Kittens

## Get All Kittens

```ruby
require 'kittn'

api = Kittn::APIClient.authorize!('meowmeowmeow')
api.kittens.get
```

```python
import kittn

api = kittn.authorize('meowmeowmeow')
api.kittens.get()
```

```shell
curl "http://example.com/api/kittens"
  -H "Authorization: meowmeowmeow"
```

```javascript
const kittn = require('kittn');

let api = kittn.authorize('meowmeowmeow');
let kittens = api.kittens.get();
```

> The above command returns JSON structured like this:

```json
[
  {
    "id": 1,
    "name": "Fluffums",
    "breed": "calico",
    "fluffiness": 6,
    "cuteness": 7
  },
  {
    "id": 2,
    "name": "Max",
    "breed": "unknown",
    "fluffiness": 5,
    "cuteness": 10
  }
]
```

This endpoint retrieves all kittens.

### HTTP Request

`GET http://example.com/api/kittens`

### Query Parameters

Parameter | Default | Description
--------- | ------- | -----------
include_cats | false | If set to true, the result will also include cats.
available | true | If set to false, the result will include kittens that have already been adopted.

<aside class="success">
Remember — a happy kitten is an authenticated kitten!
</aside>

## Get a Specific Kitten

```ruby
require 'kittn'

api = Kittn::APIClient.authorize!('meowmeowmeow')
api.kittens.get(2)
```

```python
import kittn

api = kittn.authorize('meowmeowmeow')
api.kittens.get(2)
```

```shell
curl "http://example.com/api/kittens/2"
  -H "Authorization: meowmeowmeow"
```

```javascript
const kittn = require('kittn');

let api = kittn.authorize('meowmeowmeow');
let max = api.kittens.get(2);
```

> The above command returns JSON structured like this:

```json
{
  "id": 2,
  "name": "Max",
  "breed": "unknown",
  "fluffiness": 5,
  "cuteness": 10
}
```

This endpoint retrieves a specific kitten.

<aside class="warning">Inside HTML code blocks like this one, you can't use Markdown, so use <code>&lt;code&gt;</code> blocks to denote code.</aside>

### HTTP Request

`GET http://example.com/kittens/<ID>`

### URL Parameters

Parameter | Description
--------- | -----------
ID | The ID of the kitten to retrieve

