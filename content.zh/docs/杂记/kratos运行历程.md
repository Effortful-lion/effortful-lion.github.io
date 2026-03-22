---
title: "Kratos运行历程"
weight: 1
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: true
# bookComments: false
# bookSearchExclude: false
# bookHref: ''
# bookIcon: ''
---

# Kratos运行历程

## 调用和开发顺序

![](/assets/kratos运行历程-1769083033433.png)

## 初始化 kratos 工程
```shell
kratos new demo1
```
![](/assets/kratos运行历程-1769065693505.png)

## cmd运行目录

用 Goland 打开，编辑运行配置：
![](/assets/kratos运行历程-1769065722129.png)

开始断点调试

## main 入口
```go
package main

import (
	"flag"
	"os"

	"demo1/internal/conf"

	"github.com/go-kratos/kratos/v2"
	"github.com/go-kratos/kratos/v2/config"
	"github.com/go-kratos/kratos/v2/config/file"
	"github.com/go-kratos/kratos/v2/log"
	"github.com/go-kratos/kratos/v2/middleware/tracing"
	"github.com/go-kratos/kratos/v2/transport/grpc"
	"github.com/go-kratos/kratos/v2/transport/http"

	_ "go.uber.org/automaxprocs"
)

// go build -ldflags "-X main.Version=x.y.z"
var (
	// Name is the name of the compiled software.
	Name string
	// Version is the version of the compiled software.
	Version string
	// flagconf is the config flag.
	flagconf string

	id, _ = os.Hostname()
)

func init() {
	flag.StringVar(&flagconf, "conf", "../../configs", "config path, eg: -conf config.yaml")
}

func newApp(logger log.Logger, gs *grpc.Server, hs *http.Server) *kratos.App {
	return kratos.New(
		kratos.ID(id),
		kratos.Name(Name),
		kratos.Version(Version),
		kratos.Metadata(map[string]string{}),
		kratos.Logger(logger),
		kratos.Server(
			gs,
			hs,
		),
	)
}

func main() {
	// 解析命令行参数 —— 应该是kratos内置很多命令行参数有其他作用（暂且不管）
	flag.Parse()
	// 初始化 logger
	logger := log.With(log.NewStdLogger(os.Stdout),
		"ts", log.DefaultTimestamp,
		"caller", log.DefaultCaller,
		"service.id", id,
		"service.name", Name,
		"service.version", Version,
		"trace.id", tracing.TraceID(),
		"span.id", tracing.SpanID(),
	)
	// 多配置源指定
	c := config.New(
		config.WithSource(
			// 指定配置文件路径，可以 load 可以 watch
			file.NewSource(flagconf),
		),
	)
	defer c.Close()

	// 加载多源配置 + 合并配置 + 监听配置变更
	if err := c.Load(); err != nil {
		panic(err)
	}

	// 将配置文件中的配置信息读取到内存
	var bc conf.Bootstrap
	if err := c.Scan(&bc); err != nil {
		panic(err)
	}

	// 注入配置到 app
	app, cleanup, err := wireApp(bc.Server, bc.Data, logger)
	if err != nil {
		panic(err)
	}
	defer cleanup()

	// start and wait for stop signal
	if err := app.Run(); err != nil {
		panic(err)
	}
}

```

### 多配置源指定
```go
// 多配置源指定
	c := config.New(
		config.WithSource(
			// 指定配置文件路径，可以 load 可以 watch
			file.NewSource(flagconf),
		),
	)
```

### conf.Bootstrap
```go
// 将配置文件中的配置信息读取到内存
	var bc conf.Bootstrap
	if err := c.Scan(&bc); err != nil {
		panic(err)
	}
```
```go
type Bootstrap struct {
	state         protoimpl.MessageState
	sizeCache     protoimpl.SizeCache
	unknownFields protoimpl.UnknownFields

	Server *Server `protobuf:"bytes,1,opt,name=server,proto3" json:"server,omitempty"`
	Data   *Data   `protobuf:"bytes,2,opt,name=data,proto3" json:"data,omitempty"`
}
```
```go
type Server struct {
	state         protoimpl.MessageState
	sizeCache     protoimpl.SizeCache
	unknownFields protoimpl.UnknownFields

	Http *Server_HTTP `protobuf:"bytes,1,opt,name=http,proto3" json:"http,omitempty"`
	Grpc *Server_GRPC `protobuf:"bytes,2,opt,name=grpc,proto3" json:"grpc,omitempty"`
}
```
```go
type Data struct {
	state         protoimpl.MessageState
	sizeCache     protoimpl.SizeCache
	unknownFields protoimpl.UnknownFields

	Database *Data_Database `protobuf:"bytes,1,opt,name=database,proto3" json:"database,omitempty"`
	Redis    *Data_Redis    `protobuf:"bytes,2,opt,name=redis,proto3" json:"redis,omitempty"`
}
```

而他们的配置来源：demo1/internal/conf/conf.proto 其实就是某个服务的具体 http/grpc 配置
```protobuf
syntax = "proto3";
package kratos.api;

option go_package = "demo1/internal/conf;conf";

import "google/protobuf/duration.proto";

message Bootstrap {
  Server server = 1;
  Data data = 2;
}

message Server {
  message HTTP {
    string network = 1;
    string addr = 2;
    google.protobuf.Duration timeout = 3;
  }
  message GRPC {
    string network = 1;
    string addr = 2;
    google.protobuf.Duration timeout = 3;
  }
  HTTP http = 1;
  GRPC grpc = 2;
}

message Data {
  message Database {
    string driver = 1;
    string source = 2;
  }
  message Redis {
    string network = 1;
    string addr = 2;
    google.protobuf.Duration read_timeout = 3;
    google.protobuf.Duration write_timeout = 4;
  }
  Database database = 1;
  Redis redis = 2;
}
```

证明：分别是读取的数据
![](/assets/kratos运行历程-1769066123742.png)
![](/assets/kratos运行历程-1769066135412.png)

## wireApp

```go
func wireApp(confServer *conf.Server, confData *conf.Data, logger log.Logger) (*kratos.App, func(), error) {
    dataData, cleanup, err := data.NewData(confData)
    if err != nil {
        return nil, nil, err
    }
	// 创建服务专门的 repo usecase service
    greeterRepo := data.NewGreeterRepo(dataData, logger)
    greeterUsecase := biz.NewGreeterUsecase(greeterRepo)
    greeterService := service.NewGreeterService(greeterUsecase)
	// 创建对应服务的 grpc服务器 http服务器
    grpcServer := server.NewGRPCServer(confServer, greeterService, logger)
    httpServer := server.NewHTTPServer(confServer, greeterService, logger)
    app := newApp(logger, grpcServer, httpServer)
	// 注入到 app 返回
    return app, func() {
        cleanup()
    }, nil
}
```

执行后：app的servers属性中有我们注册的grpc服务器和http服务器
![](/assets/kratos运行历程-1769066231453.png)

### repo、usecase、service
```text
1. repo: 操作数据库，实现操作
2. usecase：纯业务对象，实现业务逻辑
3. service：调用usecase层/rpc调用（调用api+转换响应结果）
```

## Run
```go
// Run会执行所有已向应用程序的生命周期注册的OnStart钩子。
func (a *App) Run() error {
	// 创建 应用程序 实例
	instance, err := a.buildInstance()
	if err != nil {
		return err
	}
	a.mu.Lock()
	a.instance = instance
	a.mu.Unlock()
	// 创建 context
	sctx := NewContext(a.ctx, a)
	eg, ctx := errgroup.WithContext(sctx)
	wg := sync.WaitGroup{}

	// 运行所有 preStart 钩子
	for _, fn := range a.opts.beforeStart {
		if err = fn(sctx); err != nil {
			return err
		}
	}
	
	// 继承键值对 + 新键值对
	octx := NewContext(a.opts.ctx, a)
	// servers：grpc 服务器 + http 服务器
	for _, srv := range a.opts.servers {
		server := srv
		// 统一收集错误
		eg.Go(func() error {
			<-ctx.Done() // wait for stop signal
			stopCtx := context.WithoutCancel(octx)
			if a.opts.stopTimeout > 0 {
				var cancel context.CancelFunc
				stopCtx, cancel = context.WithTimeout(stopCtx, a.opts.stopTimeout)
				defer cancel()
			}
			return server.Stop(stopCtx)
		})
		wg.Add(1)
		eg.Go(func() error {
			wg.Done() // here is to ensure server start has begun running before register, so defer is not needed
			return server.Start(octx)
		})
	}
	wg.Wait()
	if a.opts.registrar != nil {
		rctx, rcancel := context.WithTimeout(ctx, a.opts.registrarTimeout)
		defer rcancel()
		if err = a.opts.registrar.Register(rctx, instance); err != nil {
			return err
		}
	}
	
	// 运行所有 afterEnd 的钩子
	for _, fn := range a.opts.afterStart {
		if err = fn(sctx); err != nil {
			return err
		}
	}

	// 优雅关闭
	c := make(chan os.Signal, 1)
	signal.Notify(c, a.opts.sigs...)
	eg.Go(func() error {
		select {
		case <-ctx.Done():
			return nil
		case <-c:
			return a.Stop()
		}
	})
	if err = eg.Wait(); err != nil && !errors.Is(err, context.Canceled) {
		return err
	}
	err = nil
	for _, fn := range a.opts.afterStop {
		err = fn(sctx)
	}
	return err
}
```
