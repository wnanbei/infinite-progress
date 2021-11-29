---
title: "Go 语言规范"
date: 2019-01-01 00:00:00
categories:
  - Go语言规范
tags:
  - Go
series:
---

Go 语言规范指导性原则：

1. 简单性
2. 可读性
3. 生产力

<!--more-->

## 命名

| **概括性词汇** | **精确清晰的词汇**                                 |
| -------------- | -------------------------------------------------- |
| send           | deliver, dispatch, announce, distribute, route     |
| find           | search, extract, locate, recover                   |
| start          | launch, create, begin, open                        |
| make           | create, set up, build, generate, compose, add, new |

### 项目名

- 项目名(仓库名）的命名可以使用字母、数字。

- 多个单词建议采用中划线分隔，目前github中大多数项目都是使用用中划线。
- 不建议采用驼峰式分隔，不要使用下划线( kubernetes 中的组件名称不允许使用下划线)

正确:

> user、user-api、user-service,product、product-search、redis-go,druid、zeus、kubernetes.

错误:

> user_api、Product

### 包与导入路径

- 包名应和目录名一致
- 包路径应该尽可能简洁
- 避免在包路径中使用任何大写字母

### 常量

- 全大写，使用 `_` 分隔

- 如果是枚举类型的常量，需要先创建相应类型。

    ```go
     type Scheme string
    
     const (
        HTTP  Scheme = "http"
        HTTPS Scheme = "https"
     )
    ```

### 变量

指导原则：

1. 勿在变量名称中包含类型名称。
2. 常量应该描述它们持有的值，而不是该如何使用。
3. 循环和分支使用单字母变量，参数和返回值使用单个词，函数和包级别声明使用多个单词。
4. 方法、接口和包使用单个词。
5. 包的名称是调用者用来引用名称的一部分，要好好利用这一点。

#### 声明样式

使用一致的声明样式：

- 声明变量但没有初始化时，使用 `var`。
- 在声明和初始化时，使用 `:=`。

#### 长度

> The greater the distance between a name’s declaration and its uses, the longer the name should be.
>
> 名字的声明与其使用之间的距离越大，名字应该越长。
>
> ​                                          — Andrew Gerrand

- 声明和使用的距离越短，变量命名可以越短。如 `i` 代替 `index`。

- 长变量名称需要证明自己的合理性。

### 参数

参数默认具有文档的功能。

- 当参数类型具有描述性的时候，参数名就应该尽可能短小：

   ```go
   func AfterFunc(d Duration, f func()) *Timer
   func Escape(w io.Writer, s []byte)
   ```

- 当参数类型比较模糊的时候，参数名就应当具有文档的功能：

   ```go
   func Unix(sec, nsec int64) Time
   func HasPrefix(s, prefix []byte) bool
   ```

### 返回值

- 在外部可见的函数中，返回值的名称应当可以作为文档参考。

    ```go
    func Copy(dst Writer, src Reader) (written int64, err error)
    func ScanBytes(data []byte, atEOF bool) (advance int, token []byte,
     err error)
    ```

### Receiver

- 方法接收者的名字在同一类型的不同方法中应该保持统一。

- 由于方法接收者在函数内部经常出现，因此它经常采用一两个字母来标识方法接收者的类型。

    ```go
    func (b *Buffer) Read(p []byte) (n int, err error)
    func (sh serverHandler) ServeHTTP(rw ResponseWriter, req *Request)
    func (r Rectangle) Size() Point
    ```

### 接口类型

- 只含有一个方法的接口类型通常以函数名加上`er`后缀作为名字

    ```go
    type Reader interface {
        Read(p []byte) (n int, err error)
    }
    ```

- 有时候可能导致蹩脚的英文，但别管他，能看懂就好

    ```go
    type Execer interface {
        Exec(p []byte) (n int, err error)
    }
    ```

- 有时候可以适当调整一下英文单词的顺序，增加可读性：

    ```go
    type ByteReader interface {
        ReadByte(p []byte) (n int, err error)
    }
    ```

- 当接口含有多个方法的时候，还是要选取一个能够精准描述接口目的的名字，譬如`net.Conn`、`http/ResponseWriter`。

### Error

- Error 类型写成 `FooError` 形式

  ```go
  type ExitError struct {}
  ```

- Error 变量写成 `ErrFoo` 形式

  ```go
  var ErrFormat = errors.New("unknown format")
  ```

## 注释

注释应该做至少三件事中的一件：

1. 注释应该解释其作用。
2. 注释应该解释其如何做的。
3. 注释应该解释其原因。

### 包级别注释

- 包级别的注释就是对包的介绍，只需在同个包的任一源文件中说明即可有效。

- 对于 `main` 包，一般只有一行简短的注释用以说明包的用途，且以项目名称开头：

  ```go
   // Gogs (Go Git Service) is a painless self-hosted Git Service.
   package main
  ```

- 对于一个复杂项目的子包，一般情况下不需要包级别注释，除非是代表某个特定功能的模块。

- 对于简单的非 `main` 包，可用一行注释概括。

- 对于相对功能复杂的非 `main` 包，一般都会增加一些使用示例或基本说明，且以 `Package <name>` 开头：

  ```go
   /*
   Package regexp implements a simple library for regular expressions.
  
   The syntax of the regular expressions accepted is:
  
       regexp:
           concatenation { '|' concatenation }
       concatenation:
           { closure }
       closure:
           term [ '*' | '+' | '?' ]
       term:
           '^'
           '$'
           '.'
           character
           '[' [ '^' ] character-ranges ']'
           '(' regexp ')'
   */
   package regexp
  ```

- 特别复杂的包说明，可单独创建 `doc.go` 文件来加以说明。

### 特别声明

- `TODO:`当某个部分等待完成时，使用此开头的注释来提醒维护人员。
- `FIXME:`当某个部分存在已知问题进行需要修复或改进时，使用此开头的注释来提醒维护人员。
- `NOTE:`当需要特别说明某个问题时，使用此开头的注释。

## 版权声明

作为开源项目，必须有相应的开源许可证才能算是真正的开源。在选择了一个开源许可证之后，需要在源文件中进行相应的版权声明才能生效。

### Apache License Version 2.0

该许可证要求在所有的源文件中的头部放置以下内容才能算协议对该文件有效。

```License
// Copyright [yyyy] [name of copyright owner]
//
// Licensed under the Apache License, Version 2.0 (the "License"): you may
// not use this file except in compliance with the License. You may obtain
// a copy of the License at
//
//     http://www.apache.org/licenses/LICENSE-2.0
//
// Unless required by applicable law or agreed to in writing, software
// distributed under the License is distributed on an "AS IS" BASIS, WITHOUT
// WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the
// License for the specific language governing permissions and limitations
// under the License.
```

`[yyyy]` 表示该源文件创建的年份。

`[name of copyright owner]`，即版权所有者。如果为个人项目，就写个人名称。若为团队项目，则宜写团队名称。

### MIT License

使用 MIT 授权的项目，需在源文件头部增加以下内容。

```License
// Copyright [yyyy] [name of copyright owner]. All rights reserved.
// Use of this source code is governed by a MIT-style
// license that can be found in the LICENSE file.
```

年份和版权所有者的名称填写规则与 Apache License Version 2.0 的一样。
