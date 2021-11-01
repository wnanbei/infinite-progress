## 一、OS 文件操作

导入模块

```go
import "os"
```

### 1. 打开文件

函数：

```go
func Open(name string) (*File, error) // 只能用于读取
func OpenFile(name string, flag int, perm FileMode) (*File, error)
```

- `name` - 文件路径
- `flag` - 打开文件模式
- `perm` - 权限控制

文件打开模式：

```go
// 此三项模式必须至少指定一个
O_RDONLY int = syscall.O_RDONLY // 只读模式
O_WRONLY int = syscall.O_WRONLY // 只写模式
O_RDWR   int = syscall.O_RDWR   // 读写模式
// 这些行为的选项是可选的
O_APPEND int = syscall.O_APPEND // 添加模式
O_CREATE int = syscall.O_CREAT  // 如果文件不存在则创建新文件
O_EXCL   int = syscall.O_EXCL   // 与 O_CREATE 一起使用, 文件必须不存在
O_SYNC   int = syscall.O_SYNC   // 同步方式打开，即不使用缓存，直接写入硬盘
O_TRUNC  int = syscall.O_TRUNC  // 打开并清空文件
```

例子：

```go
file, err := os.OpenFile("example.txt", os.O_RDWR|os.O_CREATE, 0664)
if err != nil {
    ...
}
defer file.Close()
```

### 2. File 对象

#### 读取

```go
// 读取数据到指定的 []byte 中
func (f *File) Read(b []byte) (n int, err error)
// 从 off 开始读取数据到指定的 []byte 中
func (f *File) ReadAt(b []byte, off int64) (n int, err error)
// 获取文件名
func (f *File) Name() string
// 返回描述文件的 FileInfo 类型
func (f *File) Stat() (FileInfo, error)
```

#### 写入

```go
// 写入文件
func (f *File) Write(b []byte) (n int, err error)
// 指定位置写入文件
func (f *File) WriteAt(b []byte, off int64) (n int, err error)
// 写入字符串
func (f *File) WriteString(s string) (n int, err error)
```

#### 操作

```go
// 关闭文件
func (f *File) Close() error
// 将文件裁剪到 size 大小，多余部分会被丢弃，size 为 0 则清空文件
func (f *File) Truncate(size int64) error
// 将文件系统的最近写入的数据从内存中的拷贝刷新到硬盘中稳定保存
func (f *File) Sync() error
// 更改文件操作的位置
func (f *File) Seek(offset int64, whence int) (ret int64, err error)
// 更改文件权限
func (f *File) Chmod(mode FileMode) error
// 更改文件所有者
func (f *File) Chown(uid, gid int) error
// 获取目录的内容的 FileInfo 类型
func (f *File) Readdir(n int) ([]FileInfo, error)
// 获取目录的内容的名称
func (f *File) Readdirnames(n int) (names []string, err error)
// 设置读写文件超时
func (f *File) SetDeadline(t time.Time) error
func (f *File) SetReadDeadline(t time.Time) error
func (f *File) SetWriteDeadline(t time.Time) error
```

### 3. FileInfo 对象

描述文件信息的接口对象

```go
type FileInfo interface {
    Name() string       // 文件名
    Size() int64        // 文件大小
    Mode() FileMode     // 文件权限
    ModTime() time.Time // 上次更改时间
    IsDir() bool        // 是否是目录
    Sys() interface{}   // 底层数据源
}
```

## 二、OS 常用方法

### 1. 文件操作

```go
// 创建文件
func Create(name string) (*File, error)
// 创建目录
func Mkdir(name string, perm FileMode) error
// 创建路径中所有目录
func MkdirAll(path string, perm FileMode) error
// 删除文件或目录
func Remove(name string) error
// 删除目录及所有子目录内容
func RemoveAll(path string) error
// 重命名或移动文件或目录
func Rename(oldpath, newpath string) error
// 裁剪文件大小
func Truncate(name string, size int64) error
// 获取文件或目录信息
func Stat(name string) (FileInfo, error)
```

### 2. 环境变量

```go
func Clearenv()  // 清除所有环境变量
func Environ() []string  // 返回所有环境变量
func Getenv(key string) string  // 获取指定环境变量值
func Setenv(key, value string) error  // 设置环境变量
func Unsetenv(key string) error  // 取消某个环境变量
```

### 3. 系统信息

```go
func Getwd() (dir string, err error)  // 获取当前路径的绝对路径
func Hostname() (name string, err error)  // 获取当前主机名
func Getpid() int  // 获取调用者进程 ID
func Getppid() int  // 获取调用者父进程 ID
func Getgid() int  // 获取调用者组 ID
func Getegid() int  // 获取调用者有效的组 ID
func Getgroups() ([]int, error)  // 获取调用者所属的所有组
func Getuid() int  // 获取调用者用户 ID
func Geteuid() int  // 获取调用者有效的用户 ID
```

### 4. 检查异常

```go
func IsExist(err error) bool  // 检查是否是文件存在异常
func IsNotExist(err error) bool  // 检查是否是文件不存在异常
func IsPermission(err error) bool  // 检查是否是权限异常
func IsTimeout(err error) bool  // 检查是否是超时异常
```

