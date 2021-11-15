# golang config options设计

经常我们在阅读golang源码配置的时候，经常会

### java中Builder模式

```java
User user = new User.Builder()
  .name("jack")
  .build();
```

### 类似golang中实现builder模式

```go
//使用一个builder类来做包装
type UserBuilder struct {
  User
}

func (u *UserBuilder) WithName(Name string) *UserBuilder {
  u.User.Name = Name 
  return u
}

func (u *UserBuilder) WithSex(Sex int) *UserBuilder {
  u.User.Sex = Sex 
  return u
}

func (u *UserBuilder) WithAge(Sex int) *UserBuilder {
  u.User.Age = Age 
  return u
}

func (u *UserBuilder) Build() (User) {
  return  u.User
}
```
### 使用
```go
u := UserBuilder{}
user := u.WithName("jack").
  WithSex(0).
  WithAge(30).
  Build()
```

上述也是通常实现配置的方式，下面我们来看看，grpc中的config的实现方式，也是golang中大家一般都会这样实现的方式Functional Options

### grpc中的实现

```go
var opts []grpc.ServerOption
// 自己添加想要修改的参数
opts = append(opts, grpc.ReadBufferSize(128*1024))
s := grpc.NewServer(opts...)
go s.Serve(info.Listener)
```

#### 创建NewServer中，带入默认配置

```go
type ServerOption interface {
	apply(*serverOptions)
}

var defaultServerOptions = serverOptions{
	maxReceiveMessageSize: defaultServerMaxReceiveMessageSize,
	maxSendMessageSize:    defaultServerMaxSendMessageSize,
	connectionTimeout:     120 * time.Second,
	writeBufferSize:       defaultWriteBufSize,
	readBufferSize:        defaultReadBufSize,
}
// opt回调，传入可变参数
func NewServer(opt ...ServerOption) *Server {
	opts := defaultServerOptions
	for _, o := range opt {
		o.apply(&opts)
	}
  ...
}
```

#### apply实现

```go
type funcServerOption struct {
	f func(*serverOptions)
}

func (fdo *funcServerOption) apply(do *serverOptions) {
	fdo.f(do)
}

func newFuncServerOption(f func(*serverOptions)) *funcServerOption {
	return &funcServerOption{
		f: f,
	}
}
// 注册回调函数
func ReadBufferSize(s int) ServerOption {
   return newFuncServerOption(func(o *serverOptions) {
      o.readBufferSize = s
   })
}
```

好处：

