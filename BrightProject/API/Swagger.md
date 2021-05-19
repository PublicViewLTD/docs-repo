# Swagger

由于API文档的完善工作量较大，为了让各开发者能够尽快入手开发，本系统临时开放Swagger页面.

Swagger页面地址为: [https://bright.ljflytjl.cn:9101/swagger](https://bright.ljflytjl.cn:9101/swagger)

<br>

相关事项说明如下：
1. 随着API文档的完善，Swagger页面随时关闭.
2. 由于Swagger是第三方工具，且默认不支持本平台内部设计的Dto描述格式，因此Swagger自动生成的内容可能与平台实际运作的API接口冲突，包括如下已知问题：
    - 部分接口不会出现在Swagger页面上
    - Swagger页面的调试功能完全不可用
    - Swagger页面所描述的`Media type`与实际支持类型有出入
    - Swagger页面所显示的Dto在部分数据格式下与实际返回内容不符，如分页查询API返回的分页数据，在`XML`模式时为`index`、`total`、`count`等，而`MessagePack`模式则返回为`PageIndex`、`TotalPages`、`ItemCount`等.
3. Swagger仅供参考，实际请以API文档说明为准。
4. 使用Protocol buffer格式接入时，按照swagger所呈现的dto格式编写`.proto`文件即可，版本为`syntax = "proto3";`，平台内部没有使用`.proto`文件而使用特殊方法`JIT`生成了相关内容，故无法提供`.proto`文件.