---
title: "Golang 第三方库 viper"
description: 
date: 2021-08-05
categories:
  - Golang 第三方库
tags:
  - Golang
series:	
---

此包用于读取各类配置文件。

安装：

```bash
go get github.com/spf13/viper
```

`Viper` 支持的配置文件后缀名如下：

`json`, `toml`, `yaml`, `yml`, `properties`, `props`, `prop`, `hcl`, `dotenv`, `env`, `ini`

## 一、初始化

### 1. 读取配置

设置读取配置的文件，此方法需要显式的指定配置文件的路径、名称和扩展名，使用此方法设定后，`Viper` 将不会再去其他路径寻找配置文件。

```go
func SetConfigFile(in string)
```

设置读取配置的文件名和路径，如果没有设置 `ConfigFile`，那么 `Viper` 会去设置的路径中，寻找所有类型被支持的同名配置文件。

```go
func AddConfigPath(in string)  // 添加路径，可以添加多个
func SetConfigName(in string)
```

配置好路径过后就可以读取配置信息了：

```go
func ReadInConfig() error
```

### 2. 设置默认值

```go
viper.SetDefault("ContentDir", "content")
viper.SetDefault("LayoutDir", "layouts")
viper.SetDefault("Taxonomies", map[string]string{"tag": "tags", "category": "categories"})
```

### 3. 监听配置变化

调用此函数后，`Viper` 会自动监听配置文件的变化，如果配置文件有更改，`Viper` 会更新自己的配置信息。

```go
func WatchConfig()
```

如果希望在配置文件更新时，进行额外的处理，可以这样：

```go
viper.OnConfigChange(func(e fsnotify.Event) {
    fmt.Println("配置发生变更：", e.Name)
})
```

## 二、读取值

### 1. 检查值是否设置

直接使用 `Viper` 去获取值时，如果这个 `key` 在配置文件中没有设置，那么会返回零值。所以如果需要判断值是否设置了这个 `key`，使用以下方法：

```go
// 判断是否有这个设置，包括文件和代码中设置的配置
func IsSet(key string) bool
// 判断是否在配置文件中有这个配置
func InConfig(key string) bool
```

### 2. 获取值

获取单个配置的值：

```go
func Get(key string) interface{}
func GetBool(key string) bool
func GetDuration(key string) time.Duration
func GetFloat64(key string) float64
func GetInt(key string) int
func GetInt32(key string) int32
func GetInt64(key string) int64
func GetIntSlice(key string) []int
func GetSizeInBytes(key string) uint
func GetString(key string) string
func GetStringMap(key string) map[string]interface{}
func GetStringMapString(key string) map[string]string
func GetStringMapStringSlice(key string) map[string][]string
func GetStringSlice(key string) []string
func GetTime(key string) time.Time
func GetUint(key string) uint
func GetUint32(key string) uint32
func GetUint64(key string) uint64
```

### 3. 调试

此方法可以打印注册的所有配置信息。

```go
viper.Debug()
```

## 三、写入配置文件

### 1. 更改设置

在使用中可以更改既定的配置

```go
func Set(key string, value interface{})
```

### 2. 写入配置文件

更改配置之后，可以将当前配置写入配置文件

```go
func WriteConfig() error
func WriteConfigAs(filename string) error
func SafeWriteConfig() error
func SafeWriteConfigAs(filename string) error
```

