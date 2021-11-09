---
title: "Go net/http client 客户端"
description: 
date: 2019-01-01 00:00:00
categories:
  - Go标准库
tags:
  - Go
series:	
---

Go 中的`net`包封装了大部分网络相关的功能，我们基本不需要借助其他库就能实现我们的爬虫需求。

<!--more--> 

## 简单请求

其中最为常用的是 `http` 和 `url`，使用前可以根据我们的需要进行导入：

```go
import (
	"net/http"
	"net/url"
)
```

`http`提供了一些非常方便的接口，可以实现最简单的请求，例如Get、Post、Head：

```go
resp, err := http.Get("http://example.com/")
...
resp, err := http.Post("http://example.com/upload", "image/jpeg", &buf)
...
resp, err := http.PostForm("http://example.com/form",
	url.Values{"key": {"Value"}, "id": {"123"}})
```

可以看到，我们非常简单的就发起了请求并获得了响应，这里需要注意一点的是，获得的响应body需要我们手动关闭：

```go
resp, err := http.Get("http://example.com/")
if err != nil {
	// 处理异常
}
defer resp.Body.Close()  // 函数结束时关闭Body
body, err := ioutil.ReadAll(resp.Body)  // 读取Body
// ...
```

这样的请求方式是非常方便的，但是当我们需要定制我们请求的其他参数时，就必须要使用其他组件了。

## Client

`Client`是`http`包内部发起请求的组件，使用它，我们才可以去控制请求的超时、重定向和其他的设置。以下是Client的定义：

```go
type Client struct {
	Transport     RoundTripper
	CheckRedirect func(req *Request, via []*Request) error
	Jar           CookieJar
	Timeout       time.Duration // Go 1.3
}
```

首先是生成Client对象：

```go
client := &http.Client{}
```

Client也有一些简便的请求方法，如：

```go
resp, err := client.Get("http://example.com")
```

但这种方法与直接使用`http.Get`没多大差别，我们需要使用另一个方法来定制请求的Header、请求体、证书验证等参数，这就是`Request`和`Do`。

### 设置超时

这是一张说明Client超时的控制范围的图：

![img](assets/client-timeout.png)

这其中，设置起来最方便的是`http.Client.Timeout`，可以在创建Client时通过字段设置，其计算的范围包括连接(Dial)到读完response body为止。

`http.Client`会自动跟随重定向，重定向时间也会记入`http.Client.Timeout`，这点一定要注意。

```go
client := &http.Client{
    Timeout: 15 * time.Second
}
```

还有一些更细粒度的超时控制：

- `net.Dialer.Timeout` 限制建立TCP连接的时间
- `http.Transport.TLSHandshakeTimeout` 限制 TLS握手的时间
- `http.Transport.ResponseHeaderTimeout` 限制读取response header的时间
- `http.Transport.ExpectContinueTimeout` 限制client在发送包含 `Expect: 100-continue`的header到收到继续发送body的response之间的时间等待。

如果需要使用这些超时，需要到`Transport`中去设置，方法如下所示：

```go
c := &http.Client{  
    Transport: &http.Transport{
        DialContext: (&net.Dialer{
                Timeout:   30 * time.Second,
                KeepAlive: 30 * time.Second,
        }).DialContext,
        TLSHandshakeTimeout:   10 * time.Second,
        ResponseHeaderTimeout: 10 * time.Second,
        ExpectContinueTimeout: 1 * time.Second,
    },
}
```

可以看到这其中没有单独控制`Do`方法超时时间的设置，如果需要的话可以使用`context`自行实现。

### 控制重定向

在Client的字段中，有一个`CheckRedirect`，此字段就是用来控制重定向的函数，如果没有定义此字段的话，将会使用默认的`defaultCheckRedirect`方法。

默认的转发策略是最多转发10次。

在转发的过程中，某一些包含安全信息的Header，比如`Authorization`、`WWW-Authenticate`、`Cookie`等，如果转发是跨域的，那么这些Header不会复制到新的请求中。

`http`的重定向判断会默认处理以下状态码的请求：

- 301 (Moved Permanently)
- 302 (Found)
- 303 (See Other)
- 307 (Temporary Redirect)
- 308 (Permanent Redirect)

301、302和303请求将会改用Get访问新的请求，而307和308会使用原有的请求方式。

那么，我们如何去控制重定向的次数，甚至是禁止重定向呢？这里其实就需要我们自己去实现一个`CheckRedirect`函数了，首先我们来看看默认的`defaultCheckRedirect`方法：

```go
func defaultCheckRedirect(req *Request, via []*Request) error {
	if len(via) >= 10 {
		return errors.New("stopped after 10 redirects")
	}
	return nil
}
```

第一个参数`req`是即将转发的request，第二个参数 `via`是已经请求过的requests。可以看到其中的逻辑是判断请求过的request数量，大于等于10的时候返回一个`error`，这也说明默认的最大重定向次数为10次，当此函数返回`error`时，即是重定向结束的时候。

所以如果需要设置重定向次数，那么复制一份这个函数，修改函数名字和其中if判断的数字，然后在生成Client时设定到Client即可：

```go
client := &http.Client{
    CheckRedirect: yourCheckRedirect,
}
```

或者：

```go
client := &http.Client{}
client.CheckRedirect = yourCheckRedirect
```

禁止重定向则可以把判断数字修改为0。最好相应地修改errors中提示的信息。

### CookieJar管理

可以看到Client结构体中还有一个`Jar`字段，类型为`CookieJar`，这是Client用来管理Cookie的对象。

如果在生成Client时，没有给这个字段赋值，使其为`nil`的话，那么之后Client发起的请求将只会带上Request对象中指定的Cookie，请求响应中由服务器返回的Cookie也不会被保存。所以如果需要自动管理Cookie的话，我们还需要生成并设定一个CookieJar对象：

```go
options := cookiejar.Options{
    PublicSuffixList: publicsuffix.List
}
jar, err := cookiejar.New(&options)
client := &http.Client{
    Jar: jar,
}
```

这里的`publicsuffix.List`是一个域的公共后缀列表，是一个可选的选项，设置为`nil`代表不启用。但是不启用的情况下会使Cookie变得不安全：意味着foo.com的HTTP服务器可以为bar.com设置cookie。所以一般来说最好启用。

如果嫌麻烦不想启用`PublicSuffixList`，可以将其设置为`nil`，如下即可：

```go
jar, err := cookiejar.New(nil)
client := &http.Client{
    Jar: jar,
}
```

而`publicsuffix.List`的实现位于golang.org/x/net/publicsuffix，需要额外下载，使用的时候也需要导入：

```go
import "golang.org/x/net/publicsuffix"
```

## Request

这是Go源码中Request定义的字段，可以看到非常的多，有兴趣的可以去源码或者官方文档看有注释的版本，本文只介绍一些比较重要的字段。

```go
type Request struct {
	Method           string
	URL              *url.URL
	Proto            string // "HTTP/1.0"
	ProtoMajor       int    // 1
	ProtoMinor       int    // 0
	Header           Header
	Body             io.ReadCloser
	GetBody          func() (io.ReadCloser, error)
	ContentLength    int64
	TransferEncoding []string
	Close            bool
	Host             string
	Form             url.Values
	PostForm         url.Values
	MultipartForm    *multipart.Form
	Trailer          Header
	RemoteAddr       string
	RequestURI       string
	TLS              *tls.ConnectionState
	Cancel           <-chan struct{}
	Response         *Response
}
```

在这里不推荐直接生成Request，而应该使用http提供的`NewRequest`方法来生成Request，此方法中做了一些生成Request的默认设置，以下是`NewRequest`的函数签名：

```go
func NewRequest(method, url string, body io.Reader) (*Request, error)
```

参数中`method`和`url`两个是必备参数，而`body`参数，在使用没有body的请求方法时，传入`nil`即可。

配置好Request之后，使用Client对象的`Do`方法，就可以将Request发送出去，以下是示例：

```go
req, err := NewRequest("GET", "https://www.baidu.com", nil)
resp, err := client.Do(req)
```

### Method

请求方法，必备的参数，如果为空字符则表示Get请求。

注：Go的HTTP客户端不支持`CONNECT`请求方法。

### URL

一个被解析过的url结构体。

### Proto

HTTP协议版本。

在Go中，HTTP请求会默认使用`HTTP1.1`，而HTTPS请求会默认首先使用`HTTP2.0`，如果目标服务器不支持，握手失败后才会改用`HTTP1.1`。

如果希望强制使用`HTTP2.0`的协议，那么需要使用`golang.org/x/net/http2`这个包所提供的功能。

### 发起Post请求

如果要使用Request发起Post请求，提交表单的话，可以用到它的`PostForm`字段，这是一个类型为`url.Values`的字段，以下为示例：

```go
req, err := NewRequest("Post", "https://www.baidu.com", nil)
req.PostForm.Add("key", "value")
```

如果你Post提交的不是表单数据，那么你需要将其封装成`io.Reader`类型，并在`NewRequest`函数中传递进去。

### 设置Header

Header的类型是`http.Header`，其中包含着之前请求中返回的header和client发送的header。

可以使用这种方式设置Header：

```go
req, err := NewRequest("Get", "https://www.baidu.com", nil)
req.Header.Add("key", "value")
```

Header还有一些`Set`、`Del`等方法可以使用。

### 添加Cookie

前文我们已经介绍了如何在Client中启用一直使用的CookieJar，使用它可以自动管理获得的Cookie。

但很多时候我们也需要给特定的请求手动设置Cookie，这个时候就可以使用Request对象的`AddCookie`方法，这是其函数签名：

```go
func (r *Request) AddCookie(c *Cookie)
```

要注意的是，其传入的参数是Cookie类型，，以下是此类型包含的属性：

```go
type Cookie struct {
    Name       string
    Value      string
    Path       string
    Domain     string
    Expires    time.Time
    RawExpires string
    MaxAge     int
    Secure     bool
    HttpOnly   bool
    Raw        string
    Unparsed   []string
}
```

其中只有`Name`和`Value`是必须的，所以以下是添加Cookie的示例：

```go
c := &http.Cookie{
    Name:  "key",
    Value: "value",
}
req.AddCookie(c)
```

## Transport

`Transport`是`Client`中的一个类型，用于控制传输过程，是Client实际发起请求的底层实现。如果没有给这个字段初始化相应的值，那么将会使用默认的`DefaultTransport`。

Transport承担起了Client中连接池的功能，它会将建立的连接缓存下来，这可能会在访问大量不同网站时，留下太多打开的连接，这可以使用Transport中的方法进行关闭。

首先来看一下`Transport`的定义：

```go
type Transport struct {
	Proxy                  func(*Request) (*url.URL, error)
	DialContext            func(ctx context.Context, network, addr string) (net.Conn, error) // Go 1.7
	Dial                   func(network, addr string) (net.Conn, error)
	DialTLS                func(network, addr string) (net.Conn, error) // Go 1.4
	TLSClientConfig        *tls.Config
	TLSHandshakeTimeout    time.Duration // Go 1.3
	DisableKeepAlives      bool
	DisableCompression     bool
	MaxIdleConns           int // Go 1.7
	MaxIdleConnsPerHost    int
	MaxConnsPerHost        int                                                         // Go 1.11
	IdleConnTimeout        time.Duration                                               // Go 1.7
	ResponseHeaderTimeout  time.Duration                                               // Go 1.1
	ExpectContinueTimeout  time.Duration                                               // Go 1.6
	TLSNextProto           map[string]func(authority string, c *tls.Conn) RoundTripper // Go 1.6
	ProxyConnectHeader     Header                                                      // Go 1.8
	MaxResponseHeaderBytes int64                                                       // Go 1.7
}
```

由于`Transport`是`Client`内部请求的实际发起者，所以内容会比较多，1.6之后的版本也添加了许多新的字段，这里我们来讲解常见的一些字段。

### 拨号

由于Client中设置的Timeout范围比较宽，而在生产环境中我们可能需要更为精细的超时控制，在`Dial`拨号中可以设置几个超时时间。

在较新的版本中，`Dial`这个字段已经不再被推荐使用，取而代之的是`DialContext`，设置这个字段，需要借助于`net.Dialer`，以下是其定义：

```go
type Dialer struct {
	Timeout time.Duration
	Deadline time.Time
	LocalAddr Addr
	DualStack bool
	FallbackDelay time.Duration
	KeepAlive time.Duration
	Resolver *Resolver
	Cancel <-chan struct{}
	Control func(network, address string, c syscall.RawConn) error
}
```

这其中需要我们设置的并不多，主要是Timeout和KeepAlive。Timeout是Dial这个过程的超时时间，而KeepAlive是连接池中连接的超时时间，如下所示：

```go
trans := &http.Transport{
    DialContext: (&net.Dialer{
        Timeout: 30 * time.Second,
        KeepAlive: 30 * time.Second,
    }).DialContext,
}
```

### 设置代理

Transport第一个`Proxy`字段是用来设置代理，支持HTTP、HTTPS、SOCKS5三种代理方式，首先我们来看看如何设置HTTP和HTTPS代理：

```go
package main

import (
	"net/url"
    "net/http"
)

func main() {
    proxyURL, _ := url.Parse("https://127.0.0.1:1080")
    trans := &http.Transport{
        Proxy: http.ProxyURL(proxyURL),
    }
    client := &http.Client{
        Transport: trans,
    }
    client.Get("https://www.google.com")
}
```

设置SOCKS5代理则需要借助`golang.org/x/net/proxy`：

```go
package main

import (
	"net/url"
	"net/http"
	"golang.org/x/net/proxy"
)

func main() {
	dialer, err := proxy.SOCKS5("tcp", "127.0.0.1:8080",
        &proxy.Auth{User:"username", Password:"password"},
        &net.Dialer {
            Timeout: 30 * time.Second,
            KeepAlive: 30 * time.Second,
        },
    )
    trans := &http.Transport{
        DialContext: dialer.DialContext
    }
    client := &http.Client{
        Transport: trans,
    }
    client.Get("https://www.google.com")
}
```

这里的`proxy.SOCKS5`函数将会返回一个`Dialer`对象，其传入的参数分别为协议、IP端口、账号密码、`Dialer`，如果代理不需要账号密码验证的话，第三个字段可以设置为`nil`。

### 连接控制

众所周知，HTTP1.0协议使用的是短连接，而HTTP1.1默认使用的是长连接，使用长连接则可以复用连接，减少建立连接的开销。

`Transport`中实现了连接池的功能，可以将连接保存下来以便下次访问此域名，其中也对连接的数量做出了一定的限制。

`DisableKeepAlives`这个字段可以用来关闭长连接，默认值为false，如果有特殊的需求，需要使用短连接，可以设置此字段为true：

```go
trans := &http.Transport{
    ...
    DisableKeepAlives: true,
}
```

除此之外，还可以控制连接的数量和保持时间：

1. `MaxConnsPerHost int` - 每个域名下最大连接数量，包括正在拨号的、活跃的、空闲的的连接。

   值为0表示不限制数量。

2. `MaxIdleConns int` - 空闲连接的最大数量。

   DefaultTransport中的默认值为100，在需要发起大量连接时偏小，可以根据需求自行设定。

   值为0表示不限制数量。

3. `MaxIdleConnsPerHost int` - 每个域名下空闲连接的最大数量。

   值为0则会使用默认的数量，每个域名下只能有两个空闲连接。在对单个网站发起大量连接时，两个连接可能会不够，可以酌情增加此数值。

4. `IdleConnTimeout time.Duration` - 空闲连接的超时时间，从每一次空闲开始算。DefaultTransport中的默认值为90秒。

   值为0表示不限制。

由于Transport负担起了连接池的功能，所以在并发使用时，最好将Transport与Client一起复用，不然可能会造成发起过量的长连接，浪费系统资源。

## 其他

### 设置url参数

在Go的请求方式中，没有给我们提供可以直接设置url参数的方法，所以需要我们自己在url地址中进行拼接。

`url`包中提供了一个`url.Values`类型，其本质的类型为：`map[string][]string`，可以让我们拼接参数更加简单，如下所示：

```go
URL := "http://httpbin.org/get"
params := url.Values{
    "key1": {"value"},
    "key2": {"value2", "value3"},
}
URL = URL + "&" + params.Encode()
fmt.Println(URL)
// 输出为：http://httpbin.org/get&key1=value&key2=value2&key2=value3
```

## 示例

以下是发起Get请求的一个例子：

```go
// 生成client客户端
client := &http.Client{}
// 生成Request对象
req, err := http.NewRequest("Get", "http://httpbin.org/get", nil)
if err != nil {
    fmt.Println(err)
}
// 添加Header
req.Header.Add("User-Agent", "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/74.0.3729.108 Safari/537.36")
// 发起请求
resp, err := client.Do(req)
if err != nil {
    fmt.Println(err)
}
// 设定关闭响应体
defer resp.Body.Close()
// 读取响应体
body, err := ioutil.ReadAll(resp.Body)
if err != nil {
    fmt.Println(err)
}
fmt.Println(string(body))<https://colobu.com/2016/07/01/the-complete-guide-to-golang-net-http-timeouts/>)
```
