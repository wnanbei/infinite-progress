---
title: "Golang 第三方库 grpc"
description: 
date: 2021-08-05 00:00:00
categories:
  - Golang 第三方库
tags:
  - Golang
series:	
---

`gRPC` 是一个高性能、通用的开源 RPC 框架，由 Google 主要面向移动应用开发并基于 `HTTP/2` 协议标准而设计，基于 `ProtoBuf(Protocol Buffers)` 序列化协议开发，且支持众多开发语言。

使用 gRPC， 可以在一个 `.proto` 文件中定义服务，并使用任何支持它的语言去实现客户端和服务端。使用 gRPC定义一个服务，指定一个可以远程调用的带有参数和返回类型的的方法，客户端可以像调用本地方法一样直接调用服务端的方法。gRPC 解决了不同语言及环境间通信的复杂性。	

使用 `protocol buffers` 还能获得其他好处：

- 包括高效的序列号
- 简单的 IDL
- 容易进行接口更新。

使用 gRPC 能更容易编写跨语言的分布式代码。

<!--more-->

## 安装

### 安装 grpc 包

```sh
$ go get google.golang.org/grpc
```

导入：

```go
import "google.golang.org/grpc"
```

### 安装 protocol buffer 编译器

1. 到此地址根据系统下载编译好的编译器：

   https://github.com/protocolbuffers/protobuf/releases

2. 解压文件。

3. 将编译器放到环境变量中。

4. 安装 protoc 的 Go 插件

   ```sh
   $ go get -u github.com/golang/protobuf/protoc-gen-go
   ```

## 用法

`gRPC` 开发流程：

1. 编写`.proto`文件，生成指定语言源代码。
2. 编写服务端代码。
3. 编写客户端代码。

### .proto 文件

```protobuf
syntax = "proto3"; // 版本声明，使用Protocol Buffers v3版本

package pb; // 包名

// 定义一个打招呼服务
service Greeter {
    // SayHello 方法
    rpc SayHello (HelloRequest) returns (HelloReply) {}
}

// 包含人名的一个请求消息
message HelloRequest {
    string name = 1;
}

// 包含问候语的响应消息
message HelloReply {
    string message = 1;
}
```

执行命令，生成 Go 语言源代码：

```sh
$ protoc -I helloworld/ helloworld/pb/helloworld.proto --go_out=plugins=grpc:helloworld
```

在 `gRPC_demo/helloworld/pb` 目录下会生成 `helloworld.pb.go` 文件。

### server

```go
package main

import (
	"fmt"
	"net"

	pb "gRPC_demo/helloworld/pb"
	"golang.org/x/net/context"
	"google.golang.org/grpc"
	"google.golang.org/grpc/reflection"
)

type server struct{}

func (s *server) SayHello(ctx context.Context, in *pb.HelloRequest) (*pb.HelloReply, error) {
	return &pb.HelloReply{Message: "Hello " + in.Name}, nil
}

func main() {
	// 监听本地的8972端口
	lis, err := net.Listen("tcp", ":8972")
	if err != nil {
		fmt.Printf("failed to listen: %v", err)
		return
	}
	s := grpc.NewServer() // 创建gRPC服务器
	pb.RegisterGreeterServer(s, &server{}) // 在gRPC服务端注册服务

	reflection.Register(s) //在给定的gRPC服务器上注册服务器反射服务
	// Serve方法在lis上接受传入连接，为每个连接创建一个ServerTransport和server的goroutine。
	// 该goroutine读取gRPC请求，然后调用已注册的处理程序来响应它们。
	err = s.Serve(lis)
	if err != nil {
		fmt.Printf("failed to serve: %v", err)
		return
	}
}
```

### client

```go
package main

import (
	"context"
	"fmt"

	pb "gRPC_demo/helloworld/pb"
	"google.golang.org/grpc"
)

func main() {
	// 连接服务器
	conn, err := grpc.Dial(":8972", grpc.WithInsecure())
	if err != nil {
		fmt.Printf("faild to connect: %v", err)
	}
	defer conn.Close()

	c := pb.NewGreeterClient(conn)
	// 调用服务端的SayHello
	r, err := c.SayHello(context.Background(), &pb.HelloRequest{Name: "q1mi"})
	if err != nil {
		fmt.Printf("could not greet: %v", err)
	}
	fmt.Printf("Greeting: %s !\n", r.Message)
}
```

## Protocol Buffers

示例：

```protobuf
syntax = "proto3";

package routeguide;

service SearchService {
  rpc Search(SearchRequest) returns (SearchResponse);
}

message SearchRequest {
  required string query = 1;
  optional int32 page_number = 2;
  optional int32 result_per_page = 3;
}

message SearchResponse {
 ...
}
```

- `syntax = "proto3";` 是我们使用的协议版本。
- `message` 的每个字段都有个 index，范围在 1~15 之间时编码只需要一个字节，所以性能要求更高的字段尽量使用这个范围内的 index。此 index 后续不要修改。

### 字段类型

这是 `grpc` 的类型与 Python、Go 类型的对应表。

| .proto Type | Notes                                                        | Python Type | Go Type |
| :---------- | :----------------------------------------------------------- | :---------- | :------ |
| double      |                                                              | float       | float64 |
| float       |                                                              | float       | float32 |
| int32       | Uses variable-length encoding. Inefficient for encoding negative numbers – if your field is likely to have negative values, use sint32 instead. | int         | int32   |
| int64       | Uses variable-length encoding. Inefficient for encoding negative numbers – if your field is likely to have negative values, use sint64 instead. | int/long    | int64   |
| uint32      | Uses variable-length encoding.                               | int/long    | uint32  |
| uint64      | Uses variable-length encoding.                               | int/long    | uint64  |
| sint32      | Uses variable-length encoding. Signed int value. These more efficiently encode negative numbers than regular int32s. | int         | int32   |
| sint64      | Uses variable-length encoding. Signed int value. These more efficiently encode negative numbers than regular int64s. | int/long    | int64   |
| fixed32     | Always four bytes. More efficient than uint32 if values are often greater than 228. | int/long    | uint32  |
| fixed64     | Always eight bytes. More efficient than uint64 if values are often greater than 256. | int/long    | uint64  |
| sfixed32    | Always four bytes.                                           | int         | int32   |
| sfixed64    | Always eight bytes.                                          | int/long    | int64   |
| bool        |                                                              | bool        | bool    |
| string      | A string must always contain UTF-8 encoded or 7-bit ASCII text, and cannot be longer than 232. | str/unicode | string  |
| bytes       | May contain any arbitrary sequence of bytes no longer than 232. | str         | []byte  |

### 字段规则

在字段类型前可以定义字段的规则：

- `repeated`: 此字段可以出现多次，等同于一个数组

### reserved

保留字段，用于声明某些 `index` 不能使用，也可以声明字段名。

```protobuf
message Foo {
  reserved 2, 15, 9 to 11;
  reserved "foo", "bar";
}
```

### enum

枚举类型

```protobuf
message SearchRequest {
  required string query = 1;
  optional int32 page_number = 2;
  optional int32 result_per_page = 3 [default = 10];
  enum Corpus {
    UNIVERSAL = 0;
    WEB = 1;
    IMAGES = 2;
    LOCAL = 3;
    NEWS = 4;
    PRODUCTS = 5;
    VIDEO = 6;
  }
  optional Corpus corpus = 4 [default = UNIVERSAL];
}
```

### import

Protocol Buffers 支持从多个文件导入已经写好的代码。

```protobuf
import "myproject/other_protos.proto";
```

### 嵌套 Message

```protobuf
message SearchResponse {
  message Result {
    required string url = 1;
    optional string title = 2;
    repeated string snippets = 3;
  }
  repeated Result result = 1;
}
```

如果需要在父 Message 之外复用这个子 Message：

```protobuf
message SomeOtherMessage {
  optional SearchResponse.Result result = 1;
}
```

### Package

用于声明当前文件的包名，这样在其他文件导入此文件时，可以使用包名来索引到具体 message：

```protobuf
package foo.bar;
message Open { ... }
```

导入文件:

```protobuf
message Foo {
  ...
  required foo.bar.Open open = 1;
  ...
}
```

### 生成 Go 代码

生成使用 grpc 的代码。

```bash
protoc -I=$SRC_DIR --go_out=plugins=grpc:$DST_DIR $SRC_DIR/addressbook.proto
```

