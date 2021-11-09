---
title: "Gin 配置 Swagger 接口文档"
description: 
date: 2020-08-05 00:00:00
categories:
  - Go第三方库
tags:
  - Go
  - Gin
series:	
  - Go Web 开发
---

此包用于自动化生成 API 文档。

<!--more-->

下载 `swag` 工具：

```sh
$ go get -u github.com/swaggo/swag/cmd/swag
```

下载 `gin-swagger` 库：

```sh
$ go get -u github.com/swaggo/gin-swagger
$ go get -u github.com/swaggo/files
```

## 使用

### 启用文档

1. 首先需要在项目代码中根据需求编写相应的注释。

2. 使用 `swag` 生成 `swagger` 所需的 Json 文件。

   ```sh
   $ swag init
   ```

   此命令会在当前目录下生成 `docs` 目录以及里面的部分文件。

3. 在项目的 `main.go` 文件中配置 `swagger` 的启用方式。

   ```go
   import (
   	"github.com/gin-gonic/gin"
   	swaggerFiles "github.com/swaggo/files"
   	ginSwagger "github.com/swaggo/gin-swagger"
   
   	"projectName/docs"
   )
   
   func main() {
   	r := gin.New()
   	r.GET("/swagger/*any", ginSwagger.WrapHandler(swaggerFiles.Handler))
   	r.Run()
   }
   ```

4. 启动后，就可以在 `http://localhost:8080/swagger/index.html` 看到 API 文档了。

### 禁用文档

很多时候我们不需要开启文档，这时候可以使用 `DisablingWrapHandler`。

```go
r.GET("/swagger/*any", ginSwagger.DisablingWrapHandler(swaggerFiles.Handler, "NAME_OF_ENV_VARIABLE"))
```

只要在运行程序前，将 `NAME_OF_ENV_VARIABLE` 环境变量设置成任何值，都将会禁用 `Swagger` 文档。

## API 整体文档

此注释用于声明整个 API 文档的信息，注释的位置在项目的 `main` 函数位置

```go
// @title Swagger Example API
// @version 1.0
// @description This is a sample server celler server.
// @termsOfService http://swagger.io/terms/
func main() {}
```

### 字段

| 字段名                   | 说明                                                         | 示例                                            |
| ------------------------ | ------------------------------------------------------------ | ----------------------------------------------- |
| @title                   | **必填** 应用程序的名称                                      | Swagger Example API                             |
| @version                 | **必填** 提供应用程序API的版本                               | 1.0                                             |
| @description             | 应用程序的简短描述                                           | This is a sample server celler server.          |
| @tag.name                | 标签的名称                                                   | This is the name of the tag                     |
| @tag.description         | 标签的描述                                                   | Cool Description                                |
| @tag.docs.url            | 标签的外部文档的URL                                          | https://example.com                             |
| @tag.docs.description    | 标签的外部文档说明                                           | Best example documentation                      |
| @termsOfService          | API的服务条款                                                | http://swagger.io/terms/                        |
| @contact.name            | 公开的API的联系信息                                          | API Support                                     |
| @contact.url             | 联系信息的URL，必须采用网址格式                              | http://www.swagger.io/support                   |
| @contact.email           | 联系人/组织的电子邮件地址。 必须采用电子邮件地址的格式       | support@swagger.io                              |
| @license.name            | **必填** 用于API的许可证名称                                 | Apache 2.0                                      |
| @license.url             | 用于API的许可证的URL，必须采用网址格式                       | http://www.apache.org/licenses/LICENSE-2.0.html |
| @host                    | 运行API的主机（主机名或IP地址）                              | localhost:8080                                  |
| @BasePath                | 运行API的基本路径                                            | /api/v1                                         |
| @query.collection.format | 请求URI query里数组参数的默认格式：csv，multi，pipes，tsv，ssv。 如果未设置，则默认为csv | multi                                           |
| @schemes                 | 用空格分隔的请求的传输协议                                   | http https                                      |
| @x-name                  | 扩展的键必须以x-开头，并且只能使用json值                     | {"key": "value"}                                |

### Markdown

如果文档中的短字符串不足以完整表达，或者需要展示图片，代码示例等类似的内容，则需要使用 `Markdown` 描述。

| 字段名                    | 说明                                                 |
| ------------------------- | ---------------------------------------------------- |
| @description.markdown     | API 的简短描述，从 `api.md` 文件中解析               |
| @tag.name                 | tag 的名称                                           |
| @tag.description.markdown | tag 的描述，该描述将从名为 `tagname.md` 的文件中读取 |

## API 接口文档

此注释用于声明单个 API 接口的信息，注释的位置在具体的 `controller` 函数位置

```go
// FindUserByName 查询用户信息
//
// @Summary 查询用户信息
// @Description 根据用户名查询用户信息
// @Tags 用户接口
// @Accept application/x-www-form-urlencoded
// @Produce application/json
// @Param name path string true "用户账户名"
// @Success 200 {object} response.BasicResponse "基础响应类型"
// @Router /user/{name} [get]
func FindUserByName(c *gin.Context) {}
```

### 字段

| 字段名                | 描述                                                |
| --------------------- | --------------------------------------------------- |
| @summary              | API 行为的简短摘要                                  |
| @description          | API 行为的详细说明                                  |
| @description.markdown | API 行为的详细说明，从 `endpointname.md` 文件中读取 |
| @id                   | 用于标识 API 的唯一字符串，在所有 API 中必须唯一    |
| @tags                 | 每个 API 操作的 tag 列表，以逗号分隔                |
| @accept               | API 可以接收的 MIME 类型的列表                      |
| @produce              | API 可以返回的 MIME 类型的列表                      |
| @param                | API 可以接收的参数                                  |
| @security             | API 操作的安全性。                                  |
| @success              | 访问 API 成功的响应内容                             |
| @failure              | 访问 API 失败的响应内容                             |
| @response             | 与 success、failure 作用相同                        |
| @header               | 头字段                                              |
| @router               | 此 API 的路由路径                                   |
| @x-name               | 扩展字段，必须以 `x-` 开头，并且只能使用 json 值    |

### MIME 类型

`swag` 接受所有格式正确的 MIME 类型, 即使匹配 `*/*`。

除此之外，`swag` 还接受某些 MIME 类型的别名，如下所示：

| 别名                  | MIME 类型                         |
| --------------------- | --------------------------------- |
| json                  | application/json                  |
| xml                   | text/xml                          |
| plain                 | text/plain                        |
| html                  | text/html                         |
| mpfd                  | multipart/form-data               |
| x-www-form-urlencoded | application/x-www-form-urlencoded |
| json-api              | application/vnd.api+json          |
| json-stream           | application/x-json-stream         |
| octet-stream          | application/octet-stream          |
| png                   | image/png                         |
| jpeg                  | image/jpeg                        |
| gif                   | image/gif                         |

### response

声明响应，主要有 `success`, `failure`, `response` 三类，格式一致：

```go
@Success {return code} {param type} {data type} comment
@Failure {return code} {param type} {data type} comment
@Response {return code} {param type} {data type} comment
@Header {return code} {param type} {data type} comment
```

示例：

```go
@Success 200 {array} model.Account
@Header 200 {string} Token "qwerty"
@Failure 400,404 {object} httputil.HTTPError
@Failure 500 {object} httputil.HTTPError
@Failure default {object} httputil.DefaultError
```

### router

声明 API 的路由:

```go
@Router path [httpMethod]
```

**多路径参数:**

```go
@Param group_id path int true "Group ID"
@Param account_id path int true "Account ID"
@Router /examples/groups/{group_id}/accounts/{account_id} [get]
```

## Param

此字段用于声明 API 接收的数据字段，用空格分隔，如下所示：

```go
@param name param_type data_type is_mandatory comment attribute(optional)
```

示例：

```go
@Param enumstring query string false "string enums" Enums(A, B, C)
@Param enumint query int false "int enums" Enums(1, 2, 3)
@Param enumnumber query number false "int enums" Enums(1.1, 1.2, 1.3)
@Param string query string false "string valid" minlength(5) maxlength(10)
@Param int query int false "int valid" minimum(1) maximum(10)
@Param default query string false "string default" default(A)
@Param collection query []string false "string collection" collectionFormat(multi)
```

### 直接声明

1. `name` - 字段名
2. `param_type` - 字段类型，说明此字段的类型
   - query - url 参数中的字段
   - path - url 路径中的字段
   - header - 请求头
   - formData - post 表单类型字段
   - body - 请求体中的内容
3. `data_tape` - 字段的数据类型
   - string (string)
   - integer (int, uint, uint32, uint64)
   - number (float32)
   - boolean (bool)
   - user defined struct
4. `is_mandatory` - 此字段是否是必须的
5. `comment` - 注释，通常是字段的描述
6. `attribute` - 额外的属性，此部分为可选

### attribute

直接声明中的字段都是必须字段，而 `attribute` 有部分额外功能

| 字段名           | 类型      | 描述                     |
| ---------------- | --------- | ------------------------ |
| default          | *         | 服务器将使用的默认参数值 |
| maximum          | `number`  | int 最大值               |
| minimum          | `number`  | int 最小值               |
| maxLength        | `integer` | 字符串最大长度           |
| minLength        | `integer` | 字符串最小长度           |
| enums            | [*]       | 枚举类型                 |
| format           | `string`  | 格式                     |
| collectionFormat | `string`  | 指定query数组参数的格式  |

- 注1：`default` 对于必需的参数没有意义
- 注2：与 JSON 模式不同，`default` 务必符合此参数的定义类型

## 结构体字段

逐个编写 Param，Response 等字段是非常麻烦的，且不利于维护，所以通常使用 Struct 来定义字段的类型。

```go
type JSONResult struct {
    Code    int          `json:"code" `   // ID this is userid
    Message string       `json:"message"` // This is Name
    Data    interface{}  `json:"data"`
}

type Order struct { //in `proto` package
    ...
}

@param result formData jsonresult.JSONResult{data=proto.Order} true "登录参数"
@success 200 {object} jsonresult.JSONResult{data=proto.Order} "desc"
@success 200 {object} jsonresult.JSONResult{data=[]proto.Order} "desc"
@success 200 {object} jsonresult.JSONResult{data=string} "desc"
@success 200 {object} jsonresult.JSONResult{data=[]string} "desc"
```

- struct 字段后方的注释，会被读取为这个字段的描述

### Tag

在使用 struct 作为数据的定义时，主要使用 struct 的 tag 作为声明的方式。

| Tag           | 描述             | 示例                              |
| ------------- | ---------------- | --------------------------------- |
| example       | 声明字段示例值   | example:"account name"            |
| binding       | 声明字段为必须   | binding:"required"                |
| swaggertype   | 重新声明字段类型 | swaggertype:"array,number"        |
| swaggerignore | 不展示某些字段   | swaggerignore:"true"              |
| extensions    | 扩展信息         | extensions:"x-nullable,x-abc=def" |

- 除此之外，还包括 Param 的 Attribute 也可以使用到 Tag 中。

### 重命名模型

在展示时更改模型名称

```go
type Resp struct {
    Code int
}//@name Response
```

