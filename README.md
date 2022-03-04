## 目录

## 介绍



## 命名
- 需要注释来补充说明的命名不是好的命名；

### 变量
- 所有变量或常量都是用驼峰格式，非导出的变量或常量首字母采用小写；
```go
var (
	startOnce sync.Once
	exporting = &atomic.Bool{}
)

const  ConnectionMaxSize = 100
```
- 如果函数块较短（小于50行），本地变量尽量使用单字母命名，如：
  ```go
    func OpenFile(name string, flag int, perm FileMode) (*File, error) {
        testlog.Open(name)
        f, err := openFileNolog(name, flag, perm)
        if err != nil {
            return nil, err
        }
        f.appendMode = flag&O_APPEND != 0
    
        return f, nil
    }
  ```
- 如果函数块较长导致本地变量定义与引用位置较远或者本地变量较多，可以使用非单字母变量名；

### 结构体
- 命名使用驼峰格式，非导出结构体首字母采用小写，如：
```go
// RestyClient a rest client implement by Resty
type RestyClient struct {
	client *resty.Client
}
```
- 使用名词，不要使用动词，如："Exporter", "Protocol"；
- 在没有冲突的前提下，结构体前缀尽量避免与包名重复，比如："dubbo" 包下，避免定义 "DubboExporter"，而是使用 "Export"；
- 结构体的导出字段需要放在前，非导出字段在后；
- 结构体字段的命名可以参考 [变量](#变量)

### 函数名
- 函数命名使用驼峰格式，非导出函数首字母采用小写，如："NewResponse", "removePendingResponse"

### 文件名
- 文件名如果有多个单词，需要以下划线分割；

### 其他情况
- bool 类型变量，名称应以 Has, Is, Can 或 Allow 开头；
- 包含常见特有名词的命名，特有名字不能使用驼峰格式，如：
```
// bad
ApiClient, HttpRequest, IpAddress
// good
APIClient/apiClient, HTTPRequest/httpRequest, IPAddress/ipAddress
```

### 包名
- 采用小写，不能使用大写或中划线；
- 命名保持简短清晰，尽量使用单个单词的名词，如："config", "protocol", "registry"；
- 命名如需使用多个单词，可以下划线分割，如："config_center"；
- 不能使用基础库库名，如："http", "cmd", "math"；
- 不要使用复数，如："urls"；
- 不要使用信息量不足的单词，比如："common", "global", "lib", "util"；
- 尽量不要对导入的包进行重命名，以下两种情况必须使用别名导入：
    - 包名与导入路径最后一个单词不匹配，必须使用别名导入：
  ```go
    import (
        "net/http"

        client "example.com/client-go"
        trace "example.com/trace/v2"
    )
  ```
    - 导入的包名有冲突，必须使用别名导入：
  ```go
    import (
        "fmt"
        "os"
        "runtime/trace"

        nettrace "golang.net/x/trace"
    )
  ```

参考:[Package names](https://go.dev/blog/package-names)

## 注释
- 编码阶段需要同步维护好变量、函数以及包注释；
- 注释必须是完整句子，并以句号结束；
- 尽量采用英文注释，以避免各种编码问题；
- 所有导出的变量、常量、函数、结构体都必须有注释；
- 没有用的代码直接删除，不要通过注释保留下来；

### 函数注释
- 函数注释需要以函数名开头，详细描述函数的入参、功能、返回结果、错误类型等信息；

### 结构体（接口）注释
- 每个结构体（或接口）都需要在定义前一行添加注释；
- 结构体字段命名应该能够清晰体现字段含义，如果需要补充说明可以在字段定义前添加行注释或者块注释，如：
```go
// ServiceDiscoveryConfig will be used to create
type ServiceDiscoveryConfig struct {
	// Protocol indicate which implementation will be used.
	// for example, if the Protocol is nacos, it means that we will use nacosServiceDiscovery
	Protocol string `yaml:"protocol" json:"protocol,omitempty" property:"protocol"`
	// Group, usually you don't need to config this field.
	// you can use this to do some isolation
	Group string `yaml:"group" json:"group,omitempty"`
	// RemoteRef is the reference point to RemoteConfig which will be used to create remotes instances.
	RemoteRef string `yaml:"remote_ref" json:"remote_ref,omitempty" property:"remote_ref"`
}
```
- 接口下的所有方法都需要有注释，如：
```go
// Protocol extension for protocol
type Protocol interface {
	// Export service for remote invocation
	Export(invoker Invoker) Exporter
	// Refer a remote service
	Refer(url *common.URL) Invoker
	// Destroy will destroy all invoker and exporter, so it only is called once.
	Destroy()
}
```

### 包注释
- 每个包都应该有一个包注释，位于package子句之前的块注释或行注释。包如果有多个go文件，只需要出现在一个go文件中即可，如：
```go
// Package hash provides interfaces for hash functions.
package hash
```
- 如果包注释较长，建议在包下新建一个 doc.go 文件单独放置包注释；

## 控制结构

### if 语句
- 使用 if 语句初始化一次性的本地变量，减少不必要的本地变量定义，如：
```go
// good
if err := dir.registry.Subscribe(url, dir); err != nil {
    logger.Error("registry.Subscribe(url:%v, dir:%v) = error:%v", url, dir, err)
}

// bad
err := dir.registry.Subscribe(url, dir)
if err != nil {
    logger.Error("registry.Subscribe(url:%v, dir:%v) = error:%v", url, dir, err)
}
```

- 避免不必要的 else 语句，如：
```go
// good
a := 10
if b {
    a = 100
}

// bad
var a int
if b {
    a = 100
} else {
    a = 10
}
```

- 代码应该优先处理错误、异常或特殊情况，通过尽早返回或者继续来减少代码嵌套：

```go
// good
for _, e := range employees {
    if !isWellDone(e.Id) {
        log.Printf("not good enough this time")
        continue
    }

    if newSalary, err := adjustSalary(e.Id); err != nil {
        e.Salary = newSalary
    } else {
        log.Printf("adjust salary failed")
        return err
    }
}

// bad
for _, e := range employees {
    if isWellDone(e.Id) {
        if newSalary, err := adjustSalary(e.Id); err != nil {
            e.Salary = newSalary
        } else {
            log.Printf("adjust salary failed")
            return err
        }
    } else {
        log.Printf("not good enough this time")
    }
}

```

## 编码风格
- 强制使用gofmt自动格式化代码，所有代码格式风格与官方推荐保持一致；
- 单行代码不超过 120 个字符；

## 其他
### Panic
- 生产环境中运行的代码必须避免 panic；
- 只允许在程序初始化阶段通过 panic 终止程序运行；

### 单元测试
- 单元测试文件命名要以 _test 为后缀，如：application_config_test.go；

## 参考资料
- [Effective Go](https://go.dev/doc/effective_go#package-names)
- [Package names](https://go.dev/blog/package-names)