## Uber Go 语言编码规范

### 指导原则

#### interface 合理性验证

- 如果 `*Handler` 与 `http.Handler` 的接口不匹配, 那么语句 `var _ http.Handler = (*Handler)(nil)` 将无法编译通过.

- 赋值的右边应该是断言类型的零值。 对于指针类型（如 `*Handler`）、切片和映射，这是 `nil`； 对于结构类型，这是空结构。 var _ http.Handler = LogHandler{}

### 接收器 (receiver) 与接口

- 值对象只可以使用值接收器方法集
- 指针对象可以使用 值接收器方法集 + 指针接收器方法集

### 接收 Slices 和 Maps

请记住，当 map 或 slice 作为函数参数传入时，如果您存储了对它们的引用，则用户可以对其进行修改。

| Bad                                                          | Good                                                         |
| ------------------------------------------------------------ | :----------------------------------------------------------- |
| func (d *Driver) SetTrips(trips []Trip) {  d.trips = trips<br/>}<br/><br/>trips := ...<br/>d1.SetTrips(trips)<br/><br/>// 你是要修改 d1.trips 吗？<br/>trips[0] = ... | func (d *Driver) SetTrips(trips []Trip) {<br/>  d.trips = make([]Trip, len(trips))<br/>  copy(d.trips, trips)<br/>}<br/><br/>trips := ...<br/>d1.SetTrips(trips)<br/><br/>// 这里我们修改 trips[0]，但不会影响到 d1.trips<br/>trips[0] = ... |

#### 使用 `time.Time` 表达瞬时时间

- bad

```go
func isActive(now, start, stop int) bool {
  return start <= now && now < stop
}
```
- good

```go
func isActive(now, start, stop time.Time) bool {
  return (start.Before(now) || start.Equal(now)) && now.Before(stop)
}
```

### 错误类型使用

- [`errors.New`](https://golang.org/pkg/errors/#New) 对于简单静态字符串的错误
- [`fmt.Errorf`](https://golang.org/pkg/fmt/#Errorf) 用于格式化的错误字符串
- 实现 `Error()` 方法的自定义类型
- 用 [`"pkg/errors".Wrap`](https://godoc.org/github.com/pkg/errors#Wrap) 的 Wrapped errors
- [Don't just check errors, handle them gracefully](https://dave.cheney.net/2016/04/27/dont-just-check-errors-handle-them-gracefully). 
- 处理类型断言失败(t, ok := i.(string))

### 避免可变全局变量

- 使用选择依赖注入方式避免改变全局变量。 既适用于函数指针又适用于其他值类型

### 避免使用 `init()`

### 追加时优先指定切片容量

### 优先使用 strconv 而不是 fmt



## 规范

### 相似的声明放在一组

### 包名

当命名包时，请按下面规则选择一个名称：

- 全部小写。没有大写或下划线。
- 大多数使用命名导入的情况下，不需要重命名。
- 简短而简洁。请记住，在每个使用的地方都完整标识了该名称。
- 不用复数。例如`net/url`，而不是`net/urls`。
- 不要用“common”，“util”，“shared”或“lib”。这些是不好的，信息量不足的名称。

另请参阅 [Package Names](https://blog.golang.org/package-names) 和 [Go 包样式指南](https://rakyll.org/style-packages/).

### 函数名

我们遵循 Go 社区关于使用 [MixedCaps 作为函数名](https://golang.org/doc/effective_go.html#mixed-caps) 的约定

### 导入别名

如果程序包名称与导入路径的最后一个元素不匹配，则必须使用导入别名。

```go
import (
  "net/http"

  client "example.com/client-go"
  trace "example.com/trace/v2"
)
```

### 对于未导出的顶层常量和变量，使用_作为前缀

```go
// foo.go

const (
  _defaultPort = 8080
  _defaultUser = "user"
)
```

### nil 是一个有效的 slice

- 您不应明确返回长度为零的切片。应该返回`nil` 来代替。
- 要检查切片是否为空，请始终使用`len(s) == 0`。而非 `nil`。
- 零值切片（用`var`声明的切片）可立即使用，无需调用`make()`创建。

### 缩小变量作用域

```go
if err := ioutil.WriteFile(name, data, 0644); err != nil {
 return err
}
```

### 避免参数语义不明确(Avoid Naked Parameters)

```go
printInfo("foo", true /* isLocal */, true /* done */)
```

Gooder deal

```go
type Region int

const (
  UnknownRegion Region = iota
  Local
)

type Status int

const (
  StatusReady Status= iota + 1
  StatusDone
  // Maybe we will have a StatusInProgress in the future.
)

func printInfo(name string, region Region, status Status)
```

### 字符串 string format

```go
const msg = "unexpected values %v, %v\n"
fmt.Printf(msg, 1, 2)
```

## 编程模式

### 表驱动测试

```
// func TestSplitHostPort(t *testing.T)

tests := []struct{
  give     string
  wantHost string
  wantPort string
}{
  {
    give:     "192.0.2.0:8000",
    wantHost: "192.0.2.0",
    wantPort: "8000",
  },
  {
    give:     "192.0.2.0:http",
    wantHost: "192.0.2.0",
    wantPort: "http",
  },
}

for _, tt := range tests {
  t.Run(tt.give, func(t *testing.T) {
    host, port, err := net.SplitHostPort(tt.give)
    require.NoError(t, err)
    assert.Equal(t, tt.wantHost, host)
    assert.Equal(t, tt.wantPort, port)
  })
}
```

我们遵循这样的约定：将结构体切片称为`tests`。 每个测试用例称为`tt`。此外，我们鼓励使用`give`和`want`前缀说明每个测试用例的输入和输出值。

```go
tests := []struct{
  give     string
  wantHost string
  wantPort string
}{
  // ...
}

for _, tt := range tests {
  // ...
}
```

### 功能选项



### Linting

我们建议至少使用以下linters，因为我认为它们有助于发现最常见的问题，并在不需要规定的情况下为代码质量建立一个高标准：

- [errcheck](https://github.com/kisielk/errcheck) 以确保错误得到处理
- [goimports](https://godoc.org/golang.org/x/tools/cmd/goimports) 格式化代码和管理 imports
- [golint](https://github.com/golang/lint) 指出常见的文体错误
- [govet](https://golang.org/cmd/vet/) 分析代码中的常见错误
- [staticcheck](https://staticcheck.io/) 各种静态分析检查

### Lint Runners

我们推荐 [golangci-lint](https://github.com/golangci/golangci-lint) 作为go-to lint的运行程序，这主要是因为它在较大的代码库中的性能以及能够同时配置和使用许多规范。这个repo有一个示例配置文件[.golangci.yml](https://github.com/uber-go/guide/blob/master/.golangci.yml)和推荐的linter设置。

golangci-lint 有[various-linters](https://golangci-lint.run/usage/linters/)可供使用。建议将上述linters作为基本set，我们鼓励团队添加对他们的项目有意义的任何附加linters。
