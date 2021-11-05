---
title: "Golang 第三方库 zap"
description: 
date: 2021-08-05
categories:
  - Golang 第三方库
tags:
  - Golang
series:	
---

此包用于结构化记录日志。

安装：

```bash
go get -u go.uber.org/zap
```

导入：

```go
import "go.uber.org/zap"
```

优点：

- 性能高，与 Zerolog 同一水平。

## 一、创建 Logger

### 1. New

创建 `Logger` 最基础的方式是使用 `New` 方法：

```go
func New(core zapcore.Core, options ...Option) *Logger
```

此方法需要传入 `zapcore` 和 `option` 来进行构造。通常需要深度定制时使用此方法。

例子：

```go
EncoderConfig := zapcore.EncoderConfig{
    MessageKey:     "msg",
    LevelKey:       "level",
    TimeKey:        "time",
    NameKey:        "zap",
    CallerKey:      "caller",
    StacktraceKey:  "stacktrace",
    LineEnding:     zapcore.DefaultLineEnding,
    EncodeLevel:    zapcore.LowercaseLevelEncoder,
    EncodeTime:     zapcore.ISO8601TimeEncoder,
    EncodeDuration: zapcore.StringDurationEncoder,
    EncodeCaller:   zapcore.ShortCallerEncoder,
    EncodeName:     zapcore.FullNameEncoder,
}
core := zapcore.NewCore(
	zapcore.NewJSONEncoder(EncoderConfig),
    zapcore.Lock(os.Stdout),
    zap.InfoLevel,
)
logger := zap.New(
	core,
    zap.AddCaller(),
    zap.AddStacktrace(zap.ErrorLevel),
)
defer logger.sync()
```

### 2. Option

可以用于对 Logger 时进行一些配置，Zap 提供的 Option 如下：

```go
func AddCaller() Option  // 添加调用信息
func AddCallerSkip(skip int) Option  // 报告调用信息时，跳过指定调用层数
func WithCaller(enabled bool) Option  // 是否添加调用信息
func AddStacktrace(lvl zapcore.LevelEnabler) Option  // 异常时堆栈追踪
func Development() Option  // 开发模式
func ErrorOutput(w zapcore.WriteSyncer) Option  //异常输出位置
func Fields(fs ...Field) Option  // 添加字段
func Hooks(hooks ...func(zapcore.Entry) error) Option  // 添加回调
func IncreaseLevel(lvl zapcore.LevelEnabler) Option  // 仅可以提高记录等级，不能降低
func WrapCore(f func(zapcore.Core) zapcore.Core) Option  // 用于包裹或替换 Core
```

### 3. 内置的 Logger

由于 `New` 方法创建 Logger 过于复杂，`zap` 包中提供了几个内建的 Logger。

- `NewDevelopment`

  此 Logger 以人性化的格式记录 Debug 级别以上的日志和标准的异常信息。

  Warn 和以上级别的日志会自动包含堆栈追踪。

  此方法等价于 `NewDevelopmentConfig().Build(...Option)`。

  ```go
  func NewDevelopment(options ...Option) (*Logger, error)
  ```

- `NewProduction`

  此 Logger 以 Json 格式记录 Info 级别以上的日志和标准的异常信息。

  Error 和以上级别的日志会自动包含堆栈追踪。

  此方法等价于 `NewProductionConfig().Build(...Option)`。

  ```go
  func NewProduction(options ...Option) (*Logger, error)
  ```

- `NewExample`

  此 Logger 是一个常用于测试用例的 Logger。

  它以 Json 格式记录 Debug 级别以上的日志和标准的异常信息。但是删去了时间戳和调用函数，以简短日志输出。

  ```go
  func NewExample(options ...Option) *Logger
  ```

## 二、使用  Config 创建 Logger

除了使用 New 自定义创建外，使用 Config 来创建 Logger 会更加方便。

### 1. Config

Config 包含了绝大部分常见的设置。其中部分设置需要使用 `zapcore.EncoderConfig` 来进行配置。

```go
type Config struct {
    // 输出的日志级别，使用 Config.Level.SetLevel 可以原子性的更改输出的日志级别
    Level AtomicLevel `json:"level" yaml:"level"`
    // 是否开启开发模式，开启后会改变 DPanicLevel 的行为，栈追踪也会更自由
    Development bool `json:"development" yaml:"development"`
    // 是否关闭调用信息，默认状况下，所有的日志都会带有文件名和文件行数
    DisableCaller bool `json:"disableCaller" yaml:"disableCaller"`
    // 禁用自动堆栈追踪，默认情况下，堆栈追踪在开发模式抓取 WarnLevel 以上级别的日志，
    // 在生产模式抓取 ErrorLevel 以上级别的日志
    DisableStacktrace bool `json:"disableStacktrace" yaml:"disableStacktrace"`
    // 采样策略，作用是限制日志在每秒钟内的输出数量, 以防止CPU和IO被过度占用。
    // 如果为 nil 则禁用采样
    Sampling *SamplingConfig `json:"sampling" yaml:"sampling"`
    // Encoding 设置日志的编码格式，可选的有 json 和 console，也可以使用 RegisterEncoder
    // 来注册第三方编码方式
    Encoding string `json:"encoding" yaml:"encoding"`
    // 注册的 Encoder 的设置，详见 zapcore.EncoderConfig 
    EncoderConfig zapcore.EncoderConfig `json:"encoderConfig" yaml:"encoderConfig"`
    // 日志输出的位置，如果是控制台，则是 stdout，或者是日志文件路径
    // 详见 Open
    OutputPaths []string `json:"outputPaths" yaml:"outputPaths"`
    ErrorOutputPaths []string `json:"errorOutputPaths" yaml:"errorOutputPaths"`
    // 添加到根 logger 的字段
    InitialFields map[string]interface{} `json:"initialFields" yaml:"initialFields"`
}
```

在创建 Config 之后，使用 `Build()` 方法，即可根据 Config 创建相应的 Logger。

例子：

```go
cfg := zap.Config{
    Level:             zap.NewAtomicLevelAt(zap.DebugLevel),
    Development:       false,
    DisableCaller:     false,
    DisableStacktrace: false,
    Sampling:          nil,
    Encoding:          "json",
    EncoderConfig: zapcore.EncoderConfig{
        MessageKey:     "msg",
        LevelKey:       "level",
        TimeKey:        "time",
        NameKey:        "zap",
        CallerKey:      "caller",
        StacktraceKey:  "stacktrace",
        LineEnding:     zapcore.DefaultLineEnding,
        EncodeLevel:    zapcore.LowercaseLevelEncoder,
        EncodeTime:     zapcore.ISO8601TimeEncoder,
        EncodeDuration: zapcore.StringDurationEncoder,
        EncodeCaller:   zapcore.ShortCallerEncoder,
        EncodeName:     zapcore.FullNameEncoder,
    },
    OutputPaths:      []string{"stdout"},
    ErrorOutputPaths: []string{"stderr"},
    InitialFields:    map[string]interface{}{"app": "zapdex"},
}

logger, err := cfg.Build()
if err != nil {
	panic(err)
}
defer logger.Sync()

logger.Info("logger construction succeeded")
```

### 2. AtomicLevel

这是 `Zap` 中用来表示日志级别的对象，可以原子性的修改。

创建 AtomicLevel：

```go
func NewAtomicLevel() AtomicLevel
func NewAtomicLevelAt(l zapcore.Level) AtomicLevel
```

更改 AtomicLevel：

```go
func (lvl AtomicLevel) SetLevel(l zapcore.Level)
```

创建或更改特定的日志级别需要 `zapcore.Level`，以下是其定义：

```go
const (
    DebugLevel Level = iota - 1  // production 环境会被禁用
    InfoLevel                    // 最常见的级别
    WarnLevel                    
    ErrorLevel
    DPanicLevel                  // DpanicLevel 在 development 环境中会记录日志并 panic
    PanicLevel                   // PanicLevel 会记录日志并 panic
    FatalLevel                   // PanicLevel 会记录日志并调用 os.Exit(1)
)
```

## 三、Logger

### 1. 输出日志

```go
func (log *Logger) Debug(msg string, fields ...Field)
func (log *Logger) Info(msg string, fields ...Field)
func (log *Logger) Warn(msg string, fields ...Field)
func (log *Logger) Error(msg string, fields ...Field)
func (log *Logger) DPanic(msg string, fields ...Field)
func (log *Logger) Panic(msg string, fields ...Field)
func (log *Logger) Fatal(msg string, fields ...Field)
```

### 2. 全局 Logger

使用 `ReplaceGlobals` 方法可以将 logger 设置为全局的 logger：

```go
func ReplaceGlobals(logger *Logger) func()
```

设置为全局 logger 后，使用 `L` 方法就可以在任何位置获取这个全局的 logger：

```go
func L() *Logger
```

### 3. SugaredLogger

logger 在输出日志时，字段需要指定类型，较为麻烦，使用 `SugaredLogger` 则可以在输出日志时较为方便，但性能较低。获取 `SugaredLogger`：

```go
func (log *Logger) Sugar() *SugaredLogger
```

## 四、ZapCore

`ZapCore` 是 Zap 中的核心部分，在需要细致且底层的配置时使用。

导入：

```go
import "go.uber.org/zap/zapcore"
```

### 1. Core

创建 Core 主要使用 `NewCore` 方法：

```go
func NewCore(enc Encoder, ws WriteSyncer, enab LevelEnabler) Core
```

参数：

- `enc Encoder` - 使用 `EncoderConfig` 创建的 Encoder，Zap 提供了最常见的两种：

  ```go
  func NewConsoleEncoder(cfg EncoderConfig) Encoder
  func NewJSONEncoder(cfg EncoderConfig) Encoder
  ```

- `ws WriteSyncer` - 表示日志写入位置的接口，可以使用 `AddSync` 创建：

  ```go
  func AddSync(w io.Writer) WriteSyncer
  ```

  注：`*os.File`、`os.Stderr`、`os.Stdout` 已经实现了此接口。

  在并发情况下，需要给 WriteSyncer 加锁，可以使用 `Lock` 方法：

  ```go
  func Lock(ws WriteSyncer) WriteSyncer
  ```

- `enab LevelEnabler` - 用于决定记录什么级别的日志，[AtomicLevel](#2. AtomicLevel) 已实现这一接口。

#### 创建多个 Core

有些时候我们需要同时将日志输出到多个位置，这样的情况下则需要我们创建多个 `Core`，并合并成一个：

```go
func NewTee(cores ...Core) Core
```

### 2. EncoderConfig

`zapcore.EncoderConfig` 用于配置 `zap.Config` 中的 Encoder。

```go
type EncoderConfig struct {
    // 用于设置默认字段的字段名，如果值为空，则记录日志时忽略此字段
    MessageKey    string `json:"messageKey" yaml:"messageKey"`
    LevelKey      string `json:"levelKey" yaml:"levelKey"`
    TimeKey       string `json:"timeKey" yaml:"timeKey"`
    NameKey       string `json:"nameKey" yaml:"nameKey"`
    CallerKey     string `json:"callerKey" yaml:"callerKey"`
    StacktraceKey string `json:"stacktraceKey" yaml:"stacktraceKey"`
    // 每行日志的分隔符
    LineEnding    string `json:"lineEnding" yaml:"lineEnding"`
	// 日志级别的编码
    EncodeLevel    LevelEncoder    `json:"levelEncoder" yaml:"levelEncoder"`
    // 时间格式的编码
    EncodeTime     TimeEncoder     `json:"timeEncoder" yaml:"timeEncoder"`
    EncodeDuration DurationEncoder `json:"durationEncoder" yaml:"durationEncoder"`
    // 调用信息的编码
    EncodeCaller   CallerEncoder   `json:"callerEncoder" yaml:"callerEncoder"`
    // 与上面几种编码器不同，此编码器是可选的，默认会使用 FullNameEncoder
    EncodeName NameEncoder `json:"nameEncoder" yaml:"nameEncoder"`
}
```

#### 内置 Encoder

Zap 中提供了几个编码器类型，可以用于自定义编码。分别是：

```go
type LevelEncoder func(Level, PrimitiveArrayEncoder)
type CallerEncoder func(EntryCaller, PrimitiveArrayEncoder)
type DurationEncoder func(time.Duration, PrimitiveArrayEncoder)
type NameEncoder func(string, PrimitiveArrayEncoder)
type TimeEncoder func(time.Time, PrimitiveArrayEncoder)
```

Zap 也提供了一部分常用的编码器实现，可以根据需要使用：

- LevelEncoder

  ```go
  func CapitalColorLevelEncoder(l Level, enc PrimitiveArrayEncoder)  // 大写彩色
  func CapitalLevelEncoder(l Level, enc PrimitiveArrayEncoder)  // 大写
  func LowercaseColorLevelEncoder(l Level, enc PrimitiveArrayEncoder)  // 小写彩色
  func LowercaseLevelEncoder(l Level, enc PrimitiveArrayEncoder)  // 小写
  ```

- TimeEncoder

  ```go
  func EpochTimeEncoder(t time.Time, enc PrimitiveArrayEncoder)  // 时间戳
  func EpochMillisTimeEncoder(t time.Time, enc PrimitiveArrayEncoder)  // 毫秒级时间戳
  func EpochNanosTimeEncoder(t time.Time, enc PrimitiveArrayEncoder)  // 纳秒级时间戳
  func RFC3339TimeEncoder(t time.Time, enc PrimitiveArrayEncoder)  // 秒级标准格式
  func ISO8601TimeEncoder(t time.Time, enc PrimitiveArrayEncoder)  // 毫秒级标准格式
  func RFC3339NanoTimeEncoder(t time.Time, enc PrimitiveArrayEncoder)  // 纳秒级标准格式
  ```

- CallerEncoder

  ```go
  func FullCallerEncoder(caller EntryCaller, enc PrimitiveArrayEncoder)  // 长路径
  func ShortCallerEncoder(caller EntryCaller, enc PrimitiveArrayEncoder)  // 短路径
  ```

- NameEncoder

  ```go
  func FullNameEncoder(loggerName string, enc PrimitiveArrayEncoder)
  ```

- DurationEncoder

  ```go
  func MillisDurationEncoder(d time.Duration, enc PrimitiveArrayEncoder)
  func NanosDurationEncoder(d time.Duration, enc PrimitiveArrayEncoder)
  func SecondsDurationEncoder(d time.Duration, enc PrimitiveArrayEncoder)
  func StringDurationEncoder(d time.Duration, enc PrimitiveArrayEncoder)
  ```


## 五、日志文件切割归档

使用第三方库 `Lumberjack` 实现。安装：

```bash
$ go get -u github.com/natefinch/lumberjack
```

### 1. 接入 Zap

```go
func logWriter() zapcore.WriteSyncer {
	lumberJackLogger := &lumberjack.Logger{
		Filename:   "./test.log",
		MaxSize:    10,
		MaxBackups: 5,
		MaxAge:     30,
		Compress:   false,
	}
	return zapcore.AddSync(lumberJackLogger)
}
```

Lumberjack 参数：

- Filename: 日志文件的位置
- MaxSize：在进行切割之前，日志文件的最大大小（以MB为单位）
- MaxBackups：保留旧文件的最大个数
- MaxAges：保留旧文件的最大天数
- Compress：是否压缩/归档旧文件