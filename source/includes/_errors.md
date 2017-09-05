# 错误

请求错误用HTTP Error Code和JSON返回值的组合报回客户端. 为了参照, 常见的HTTP错误代码如下:


Error Code | 解释
---------- | -------
400 | Bad Request -- 你的请求格式有错误.
401 | Unauthorized -- 不允许访问该资源.
403 | Forbidden -- 不允许访问该资源.
404 | Not Found
405 | Method Not Allowed -- 不允许该类HTTP Request访问此资源 (GET/POST混了)
410 | Gone -- 资源已被删除
412 | Precondition Failed -- 一致性header (`X-PKU-Integrity`) 错误
418 | I'm a teapot -- 我是一个茶壶
429 | Too Many Requests
451 | Fahrenheit 451 -- 有关部门责令删除
500 | Internal Server Error
503 | Service Unavailable

> 一般的错误返回格式:
```json
{
	"status":   "error",
	"action":   "fatal",
	"error":    "客户端版本过低, 请使用新的客户端",
	"devError": "请求格式错误 (请尝试preshared密钥)"
}
```

错误返回的参数一般包括以下字符串:
参数 | 解释
--- | -------
status | 状态, 为 `error`
action | 错误的严重程度和采取的活动. `fatal` 为毁灭性错误 (终止客户端并通知用户), `notify` 则通知用户 (客户端生产模式时, 用 `error` 信息, 客户端开发模式时可以用更详尽的 `devError`), `silent` 则不采取行动
error | 面向前端一般用户的错误信息
devError | 面向开发者的详尽错误信息

如有开发需要, 返回参数也可能包含 `debug` 参数, 仅用于debug并且只在 `IN_DEV` (后端开发模式) 开启的时候产生.