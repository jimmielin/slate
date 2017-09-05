---
title: PKU Helper 后端指南

language_tabs:
  - shell

toc_footers:
  - 用Slate制作
  - 2017-09-03 Brasilia, BRA

includes:
  - errors

search: true
---

# 介绍

PKU Helper 新一代后端于2017年8月在一个[僻静的机场](http://www.gcmap.com/airport/SBBR)中被创作, 主要想法源自于 [@luolc](https://github.com/luolc) 与 [@jimmielin](https://github.com/jimmielin) 的一些讨论. 其特点为:

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
'{ "hello": "world"}' | 
openssl rsautl -encrypt -pubin -inkey preshared_public_key.pub |
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

> 这里的访问结果应该是成功的. 无论你进行明文还是密文传输, `POST /test` 都会给你一个成功的返回值, 你可以用它来测试.

PKU Helper的上行信息全部采用密文传输, 即便是公开访问的API也使用一个预设的key. 使用何种秘钥会决定访问PKU Helper后端功能的权限, 因而一旦用户登陆后即可采用用户的key来进行传输. 

<aside class="warning">
<strong>在开发模式中</strong>, 你可以随时伪装访问任何PKU Helper用户账号, 并直接POST JSON内容, 这样方便你进行测试. 你只需要将 `Content-Type` 选择为 `application/json`. 如果你需要伪装一个具体的用户, 则用 `X-PKUHelper-AuthOverride-ID` header 来描述目标用户ID. 

注意, 开发模式的服务器是分开的. 部署服务器在任何条件下不会接受 `application/x.pku` 之外的访问类型.
</aside>

有人会问, 为什么要采取 `X-PKU-Integrity` 来描述NVA/NVB值而不是封装在POST主内容内呢? 问题本身已经包含了答案 -- GET 请求中没有 request body, 而我们也不希望我们的GET请求轻易被replay.

<aside class="notice">
我们建议进行开发的时候讲负责HTTP请求的部分进行抽象封装, 因为你肯定不希望每次都写一大段RSA加密和生成 `X-PKU-Integrity` 的部分, 跟别说同名地进行开发和部署模式的切换了.
</aside>

# 动态PKU API访问系统

## 概述

鉴于北京大学API的极度不稳定性, 我们希望能够在不发布版本更新的前提下 (因为版本更新只应该用于引入面向用户的功能性改变), 对于各个网页的访问指令进行即时的更新.

这个访问指令库 ("ISA", Instruction Set) 可以独立于PKU Helper的主程序版本进行工作, 只需要满足以下前提:

- **前后端约定对于一个给定的操作, 存在一系列具有稳定结果的指令.** 
比如访问 `dean` 中的成绩, 首先访问 `dean` 需要获取验证码和登陆, 从而 `dean` 指令集中包含了 `getCaptcha` 和 `login` 两个操作. 
这两个操作的具体方式被ISA所抽象, 客户端只需要按照ISA中的描述进行HTTP请求, 并将HTTP请求上报给服务器, 服务器会返回**一个稳定的结果**. 因此无论北京大学计算中心如何修改他们的返回结果, 只需要后端进行parser的更新, 客户端得到的结果一定是干净的.

<aside class="notice">
注意ISA文件由客户端抓取的时候是JSON格式返回, 只是我们写JSON没有YAML方便, 因而用YAML格式编写由服务器翻译成JSON提供给客户端.
</aside>

> 我们来看一个典型的ISA描述文件示例 (YAML), 配以相应的注释. 

```yaml
dean: # 属于 "dean" 系列
  login: # 的 "login" 功能
    - method: "POST" # GET/POST/UPDATE/DELETE/PUT
      url:    "http://dean.pku.edu.cn/student/authenticate.php, sno, password, captcha, submit=%B5%C7%C2%BC" # URL
      headers:
        Accept: "text/html"
        Content-Type: "application/x-www-form-urlencoded"
        Host: "dean.pku.edu.cn"
        Origin: "http://dean.pku.edu.cn"
        Referer: "http://dean.pku.edu.cn/student/"
      # 请求body可以为一个hash table, 也可以为单纯的string, 其中可替换部分 (已经约定好了dean/login的参数是哪些, 这些都是相对稳定的) 用 {} 描述
      body:
        sno:      "{student_number}"
        password: "{password}"
        captcha:  "{captcha}"
        submit:   "%B5%C7%C2%BC"
      # 期待的结果, 客户端可以用这个来判断.
      expects:
        # 成功的话应该收到如下信息
        success:
          headers:
            Connection: "close"
            Content-Type: "text/html"
          # HTTP请求结果body中包含如下信息
          bodyContains: '&lt;html&gt;&lt;body&gt;&lt;script Language ="JavaScript"&gt;parent.location.href="student_index.php?'
        fail:
          # 失败的错误. 我们可以描述一系列可能出现的错误 (不详尽), 以提供给用户更精确的信息. 如果不满足success或fail任何一条则结果一定是fail, 只是是 "未知错误"
          captchaIncorrect:
            bodyContains: "图形校验码错误，请重新登录"
          credentialsIncorrect:
            bodyContains: "学号不存在，或者密码有误。-1"
      # 需要进行的操作 (登录不需要后端处理, 这里只是做个例子)
      action:
        success: null
        fail: null
```

对于一个操作ISA的描述可以包括一系列HTTP请求 (装在一个数组里). 每个请求由如下参数描述:

<aside class="success">
如无特别描述, 一套指令 (e.g. `dean`, `elective`) 可以共用cookie, 每次默认都保存收到的cookie并发送收到的cookie.
</aside>

参数 | 类别 | 描述
--- | ----- | ------
method | string | 请求method (GET/POST/...)
url | string | 目标URL
headers | array | 请求headers
body | array/string | 如为GET则忽略, array则需要urlencode, string则直接发送
expects | array | 期待结果, 可能包含 `success`, `fail` 结果
action | array | 进行的进一步操作描述

其中 `expects` 部分可以包含 `success` 或者 `fail` 结果, 成功只有一个, 但是失败的可能是多种多样的. 它们各自*可能*包括:

参数 | 类别 | 描述
--- | ----- | ------
headers | array | 需要match的headers
body | string | 返回body的精确值
bodyContains | string | 返回body需要包含的值

其中 `action` 部分包含 `success` 或者 `fail` 结果, 内部会包含一个 `action` 字符串, 具有以下可能的结果和参数:

<aside class="warning">
如果 `fail` 的具体类型已经被上面描述 (一般都是面向用户可以解决的错误) 则直接客户端处理即可, 无需参照这里的 `fail` 部分. 这里的 `fail` 部分是处理未知情况所用, 一般都是牵扯到API改了的问题.
</aside>

action | 描述
--- | ------
nothing | 啥也不做 (一般就省略掉这个section即可, 不需要明示)
parse | 将结果上报到同级 `parser` 属性所指定的URL才能得到这个操作的结果
report | 将结果上报到同级 `parser` 属性所指定的URL, 操作终止
notify | 告诉用户发生了错误, 具体错误信息*可能*在 `message` 属性中

## 获取ISA

`GET /manifest/latestisa`

获取ISA很简单. 每次客户端启动时, 你需要按照 RFC 7232 $3.3 标准进行一个有条件GET请求, 即需要包含一个 `If-Modified-Since` header (其中日期的格式随意), 包含了你上次更新ISA的时间. 如果ISA没有更新, 你得到的就会是 `HTTP 304 Not Modified`, 无需进行更新. 否则你会得到更新的JSON版ISA.

```shell
curl 
  -H "Content-Type: application/x.pku"
  -H "If-Modified-Since: ..."
  -H "Authorization: 8faa2562030bc5be6b9160e6ddb3290dc0c4e1cf4fe47ae0a8d459baaa1ddd47"
  -H "X-PKU-Integrity: ..."
  -X GET
  "https://api.pkuhelper/manifest/latestisa"
```

> 注意为了防止他人随意抓取ISA, `X-PKU-Integrity` 的要求仍然如旧, 即使这是 `GET` 操作.

## Parser数据结果上报

如果操作结果所对应的 `action` 为 `parse`, 你需要按照如下的统一格式上报服务器**最后一次收到的返回信息**:

参数 | 描述
---- | ----
status | HTTP Status Code (e.g. 200)
headers | 收到的所有HTTP Headers
cookies | 做出request时的所有HTTP Cookie
body | 返回的request body

一般来讲不需要客户端对于数据做出判断 (因为客户端级别的数据识别很可能带来安全隐患, 或者闪退的可能), 从而这个脏活交由服务器来完成. 注意, 服务器并不是万能的. **它有可能返回失败的结果**, 如果parse出的数据不符合我们以往对这个接口返回值的经验, 或者有其他的异常.

这种保守性可以确保返回值对客户端是可信的, 而不会造成其他的后果.

> 一个成功的数据parse结果通常看起来像这样 (所有的返回数据都在 `payload` 对象内)

```json
{
  "status": "success",
  "payload": {
    "coursesSelected": [
      {
        "id": 10002830,
        "name": "飞行与魔法",
        "department": "物理学院",
        "slots": 30,
        "selected": 47,
        "type": 1,
        "assignPoints": 98
      }
    ]
  }
}
```

> 一个失败的数据parse:

```json
{
  "status": "error"
}
```

<aside class="success">
总而言之, parser的返回结果都是可信的而稳定的, 即便它倾向于返回不成功的值. 这是为了确保如果你收到 <code>success</code> 状态, 它真的是 <code>success</code> 而不是什么乱七八糟的regex给你的错误的结果.
</aside>

<aside class="warning">
对于失败的parser尝试, 后端会进行数据的适当记录, 因为这一般意味着接口变动了或者我们遇到了没考虑的情况 (比如一个人有一些比较奇特的成绩或者课程)
</aside>

## 具体的功能

### Dean "北京大学教务部" (本科)

Dean作为一个多年来相对稳定的功能, 其操作描述变动不会特别大, 作为描述文件的熟悉工具再好不过了. 目前Dean支持以下操作:

**基本操作**

- `getCaptcha` 获取验证码图片.
- `login` 登录流程, 获取登录cookie. 在此之前, 你需要运行 `getCaptcha`. 接受参数: `{student_number}`, `{password}`, `{captcha}`.

**功能性操作**
所有功能性操作都应当在 `login` 完成之后进行.

- `getGrades` 获取成绩. 其数据上报服务器处理, 处理结果如右图.

> `getGrades` 的正常parser为 `POST /dean/parse_new_grade`. 其有两个模式, 你可以通过更改 `format` 参数的值来指定做出哪种返回.

> `format` 为 `ordered` 时

```json
{
  "status": "success",
  "payload": 
    [
      {
          "year": "15-16",
          "term": "1",
          "id": "00130201",
          "type": "专业必修",
          "name": "高等数学 (B) (一) Advanced Mathematics (B) (1)",
          "credit": "5",

          "status": "success",
          "grade": "89",
          "gp": "3.77"
      },
      {
          "year": "15-16",
          "term": "1",
          "id": "00130202",
          "type": "专业必修",
          "name": "高等数学 (B) (二) Advanced Mathematics (B) (2)",
          "credit": "5",

          "status": "withdraw",
          "grade": null,
          "gp": null
      }, 
      ...
     ]
}
```

> `format` 为 `assoc` 时返回的则是Hash Table:

```json
{
  "status": "success",
  "payload": 
    {
      "00130201": {
          "year": "15-16",
          "term": "1",
          "id": "00130201",
          "type": "专业必修",
          "name": "高等数学 (B) (一) Advanced Mathematics (B) (1)",
          "credit": "5",

          "status": "unsure",
          "grade": null,
          "gp": null,
          "gradeUpperBound": 100,
          "gradeLowerBound": 94
      },
      ...
    }
}
```

> 课程 `status` 可能是 `success`, `withdraw` 或者 `unsure`. 对应的参数变化如上.