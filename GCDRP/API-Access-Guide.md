# 开放平台API接入指南

## 前言

1. 该服务内部使用`Protocol buffer 3`描述Api, 即：无论使用WebAPI、GRPC等各种形式的方式接入服务，请牢记，Api参数是有次序关系的。
2. 所有时间相关内容，无特殊说明均为UTC标准时。

<br>

## 接入方式

开放平台当前支持支持WebAPI和GRPC接入方式。

当前阶段对外开放接入标准为：`Web API Restful Json`

<br>

## Web API接入方式

?> Web API仅支持`HTTPS`形式接入，不支持HTTP。

### application/json 格式RestFul Web Api 结构描述

| 数据项   | 数据类型        | 描述       | 必须字段 | 备注 |
| -------- | --------------- | ---------- | -------- | ---- |
| `app`    | `string`/`guid` | App ID     | `true`   |      |
| `secret` | `string`        | App Secret Hash | `true`   | Hash处理后的AppSecret，可在开发者面板查看 |
| `nonce` | `number` (`int32`) | 临时流水号 | `true`   | 请确保该值在2分钟内唯一，否则系统会拒绝请求(HTTP 400) |
| `sign` | `string` | 数据签名 | `true` | 签名规则请见下方说明 |
| `time` | `number` (`int64`) | Unix时间戳 | `true` | 数据发送时的时间戳，2分钟内有效。接入方请注意时间戳的千年虫问题（Unix 2038问题），服务器不会出现该问题。 |
| `data` | `object` （泛型） | **数据主体** | `true` | 数据本体 |

<br>

### Web Api请求签名规则

**动态参数** ： 将`数据主体` 内的所有**必须字段**的数据**按照次序**拼接为字符串。

1. 获取`动态参数`字符串
2. 在`动态参数`字符串后方拼接`AppSecret`（Hash之前的原始文本），得到新字符串，记作“Secret参数”
3. 将`Secret参数`计算获取其**小写16位**MD5，得到`签名`

**签名参数计算注意事项**：

1. 可选参数不计入运算，此外备注中如有特殊标注的参数亦不计入运算。
2. 在运算数字格式时，如数字无小数，请不要携带小数点后位数。（部分编程语言中，`float`格式的`10`默认转字符串会得到`"10.00"`，这会导致签名校验失败，应改为`"10"`）

> 用于开发调试用途，该页面提供了计算签名的计算器：[https://gcdrp.zhongshihudong.com/Developer/Utils/SignCalculator](https://gcdrp.zhongshihudong.com/Developer/Utils/SignCalculator ':ignore :target=_blank')

### WebAPI 版本问题

同一个API可能存在多个版本，默认调用情况下会访问最新版本。

如需手动指定访问版本，请在HTTP请求头中添加版本信息`meow-api-version`, 例如：

```
GET api/hello HTTP/1.0
host: localhost
meow-api-version: 1.0
```

```
HTTP/1.1 200 OK
host: localhost
content-type: text/plain
content-length: 12

Hello world! 1.0
```

> 具体版本信息及对应Api内容请以Swagger为准


### 通过Header传递验证信息

为了规范实施Restful，有时候我们不方便在Body中使用Json传递基础验证信息。如在使用GET方法时，在Body中携带Json是不规范的。

为了应对这种情况，在部分API是，我们在Http Header中传递验证信息(`app`,`secret`,`nonce`,`sign`, `time`).

> 如果Swagger中某个Api在`Request body`中包含了验证信息，则使用json方式传递验证信息。否则使用`header`传递验证信息。