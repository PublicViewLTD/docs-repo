# MessagePack接入指引

接下来演示使用MessagePack数据编排格式以接入本平台。

本文所用实例接口详见API文档: [Hello World](/API/Hello)

## 约定说明

本平台的MessagePack的约定如下：

客户端请求DTO（`Data Transfer Object`，即数据传输对象）中，数据以"Key/Value"形式编排.

在一个在`Body`中携带了验证数据的典型`POST`方法请求中，数据的编排格式如下：

``` lua
--[[
    使用开发者账号信息验证的API
]]
local body = {
    "secret"    = "a490080e777f453b",   -- secret hash
    "nonce"     = 100,                  -- 流水号
    "sign"      = "c720230e678c632e",   -- 数据签名
    "time"      = 1619845237,           -- UNIX时间戳
    "user"      = "9da14f23-766c-3357-bccd-2497463ac82c",   -- 账户ID
    "data"      = {
        -- 具体的数据内容
        "greet" = "Hello, Meow!" -- 这里的数据内容按顺序排序来计算数据签名
    }
}


--[[
    使用设备信息验证的API
]]
local device_body = {
    "secret"    = "a490080e777f453b",   -- 设备 secret hash
    "nonce"     = 100,                  -- 流水号
    "sign"      = "c720230e678c632e",   -- 数据签名
    "time"      = 1619845237,           -- UNIX时间戳
    "device"    = "3fa85f64-5717-4562-b3fc-2c963f66afa6",   -- 设备ID
    "data"      = {
        -- 具体的数据内容
        "greet" = "Hello, Meow!" -- 这里的数据内容按顺序排序来计算数据签名
    }
}
```

服务端向客户端返回的DTO中，数据通常不包含Key，以元组形式编排.

<br>

## 示例 1

示例所用的API[请查阅文档](/API/Hello?id=get-apihello)，示例所使用的编程语言为`Golang`.

``` go
package main

import (
	"fmt"
	"io/ioutil"
	"net/http"
	"strconv"
	"time"
)

func main() {
	// apiSecret := "b1yUWkXZgrT1Of7t"
	apiSecretHash := "88ee325fab0791d7"
	userId := "8dd8c6e3-3634-4bc0-b523-9a0e29c7c40e"
	currentTime := time.Now()
	curTimeStamp := strconv.FormatInt(currentTime.Unix(), 10)

	//创建 http client
	client := &http.Client{}
	req, _ := http.NewRequest("GET", "https://bright.ljflytjl.cn:9101" + "/api/Hello", nil)

	// 设置请求头
	req.Header.Set("Content-Type", "application/x-msgpack") //Content-Type表示自己发送的资源类型
	req.Header.Set("Accept", "text/plain") //Accept表示希望请求到的资源类型
	req.Header.Set("secret", apiSecretHash)
	req.Header.Set("nonce", "10") //实际请求中请确保流水号在两分钟内不重复
	req.Header.Set("time", curTimeStamp)
	req.Header.Set("user", userId)

	//发送请求
	resp, err := client.Do(req)
	if err != nil {
		fmt.Println("Err:" + err.Error())
	}

	//最后关闭连接
	defer resp.Body.Close()

	//读取内容
	body, err := ioutil.ReadAll(resp.Body)
	if err != nil {
		fmt.Println("解析内容出错" + err.Error())
	}

	fmt.Println("服务端返回：" + string(body))
}
```

上述示例中由于请求的是GET类型并且没有传递参数，所以并没有计算参数签名。并且由于该API返回的是纯文本结果，所以可以指定请求头`Accept`为`text/plain`.

<br>

------

<br>

## 示例 2

本示例展示的依然是GET请求，但在URL中携带RESTFUL风格的请求参数，并对其计算签名.

示例所用的API[请查阅文档](/API/Hello?id=get-apihellogreet)，示例所使用的编程语言为`Lua`, 所使用客户端开发框架为`yomunsam/TinaX`, 所使用Lua运行环境为`tencent/xLua`.

``` lua
local M = {}
M.UserId = "8dd8c6e3-3634-4bc0-b523-9a0e29c7c40e";
M.ApiSecret = "b1yUWkXZgrT1Of7t";
M.ApiSecretHash = "88ee325fab0791d7";

--- 获取字符串格式的UTC Unix时间戳
function M.GetUnixTimeStampStr()
    return tostring(os.time(os.date("!*t")));
end

--- 获取数据签名
function M.GetSignature(...)
    local paramsStr = "";
    for i , v in ipairs({...}) do
        paramsStr = paramsStr .. tostring(v);
    end
    local secretParams = paramsStr .. M.ApiSecret;
    return CS.TinaX.EncryUtil.GetMD5(secretParams, true, true); --调用TinaX原生层Util
end

function M.GetHelloWorld(greet)
    if greet == nil or type(greet) ~= "string" then 
        error("greet is invalid.")
    end
    local md5 = M.GetSignature(greet);
    local req = XCore.Http.MakeRequest({
        uri = "https://bright.ljflytjl.cn:9101/api/Hello/" .. greet,
        header = {
            { key = "Content-Type", value = "application/x-msgpack" },
            { key = "Accept", value = "application/x-msgpack" },
            { key = "secret", value = M.ApiSecretHash },
            { key = "nonce", value = 10 },
            { key = "time", value = M.GetUnixTimeStampStr()},
            { key = "sign", value = M.GetSignature(greet)},
            { key = "user", value = M.UserId}
        }
    });
    XCore.Http.GetAsync(req, function(resp, err)
        -- onFinish callback
        if err ~= nil then
            error(err.Message)
        end
        print(resp:ReadAsString());
    end)
end

return M;
```


<br>

------

<br>

## 示例 3

本示例展示的是Hello World中的`POST`请求，该请求涉及DTO处理，即`MessagePack`的序列化和反序列化操作。

示例所用的API[请查阅文档](/API/Hello?id=post-apihello)，示例所使用的编程语言为`C#`, 开发平台为`.NET 5`, 所使用HTTP客户端库为`Flurl.Http`,所使用MessagePack序列化库`MessagePack`

本实例中，我们需要为请求数据创建DTO类，为了方便复用，将Dto抽象出部分公用基类：

``` csharp
using MessagePack;

namespace PublicView.Dtos
{
    [MessagePackObject]
    public class ApiBaseDto
    {
        [Key("secret")]
        public string SecretHash { get; set; }
        
        [Key("nonce")]
        public int Nonce { get; set; }

        [Key("sign")]
        public string Signature { get; set; }
        
        [Key("time")]
        public long TimeStamp { get; set; }

        [Key("user")]
        public string UserId { get; set; }
    }

    [MessagePackObject]
    public class ApiBaseDto<T> : ApiBaseDto
    {
        [Key("data")]
        public T Data { get; set; }
    }
}
```

并编写示例中Api所需的业务相关的Dto:

``` csharp
[MessagePackObject]
public class HelloC2SDto //客户端向服务端请求的DTO
{
    [Key("greet")]
    public string Greet { get; set; }
}

[MessagePackObject]
public class HelloS2CDto //服务端向客户端返回结果的DTO
{
    [Key(0)] //服务端向客户端返回的DTO以元组形式编排参数，所以Key标注为index，从0开始.
    public string Message { get; set; }

    [Key(1)]
    public int Code { get; set; }
}
```
!> 如果客户端需要同时接入`MessagePack`、`XML`和`Protocol Buffer`等多种序列化格式，则上述Dto写法可能产生冲突，请根据实际业务变通处理。

HTTP客户端发送请求的代码如下:

``` csharp
using System;
using System.Net.Http;
using System.Threading.Tasks;
using Flurl;
using Flurl.Http;
using MessagePack;
using Nekonya;
using PublicView.Dtos;

namespace PublicView
{
    class Program
    {
        const string UserId = "8dd8c6e3-3634-4bc0-b523-9a0e29c7c40e";
        const string ApiSecret = "adg1zLXjVLzBrNQh";
        const string ApiSecretHash = "ac1645453ff9a6a4";

        static async Task Main(string[] args)
        {
            var hello_dto = new HelloC2SDto { Greet = "Meow" };
            var c2s_dto = new ApiBaseDto<HelloC2SDto>
            {
                Data = hello_dto,
                SecretHash = ApiSecretHash,
                Nonce = 11,
                Signature = GetUserApiSign(hello_dto.Greet),
                TimeStamp = GetNowUnixTimeStamp(),
                UserId = UserId
            };

            try
            {
                var response = await "https://localhost:5001".AppendPathSegment("/api/Hello")
                    .WithHeader("Content-Type", "application/x-msgpack")
                    .WithHeader("Accept", "application/x-msgpack")
                    .PostAsync(new ByteArrayContent(MessagePackSerializer.Serialize(c2s_dto)));

                var s2c_dto = MessagePackSerializer.Deserialize<HelloS2CDto>(await response.GetBytesAsync());
                Console.WriteLine(s2c_dto.Message);
                Console.WriteLine(s2c_dto.Code.ToString());
            }
            catch(FlurlHttpException e) //此处处理诸如400 Bad request等异常
            {
                Console.WriteLine(await e.GetResponseStringAsync());
            }
        }

        public static string GetUserApiSign(params string[] dataParams) { } //篇幅所限，内容省略，与前例一致

        public static long GetNowUnixTimeStamp() { } //篇幅所限，内容省略，与前例一致
    }
}
```