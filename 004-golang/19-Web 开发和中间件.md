回顾一下前面在网络编程中启动 http 服务器的代码，其实只需要一行:

```go
http.ListenAndServe(":3000", nil)
```

访问本机的 3000 端口，你会收到: 

```go
404 page not found
```

收到 404 http 的状态码，说明服务器已经正常启动，但是没有任何的路由处理器，所以返回了 404，看看 http.ListenAndServe 函数的签名和实现:

```go
func ListenAndServe(addr string, handler Handler) error {
  server := &Server{Addr: addr, Handler: handler}
  return server.ListenAndServe()
}
```

构建了一个 http.Server 对象，Server 结构第一个是 Addr 对象(string)，第二个是 Handler 对象。

```go
type Server struct {
  Addr string
  Handler Handler // handler to invoke, http.DefaultServeMux if nil
  ......
}
```

Addr 是一个字符串表示地址和端口，Handler 是什么呢? 继续 F12 查看:

```go
type Handler interface { 
  ServeHTTP(ResponseWriter, *Request)
}
```

原来是一个实现了 ServeHTTP 函数的接口对象，这个接口恰好就是对 http 请求和响应的封装，而且 Server 的结构有个注释，如果第二个参数是 nil，那么对应的 Handler 对象就是 http.DefaultServeMux，查看  Server 的 ServeHTTP 方法，源代码如下:

```go
func (sh serverHandler) ServeHTTP(rw ResponseWriter, req *Request) {
  handler := sh.srv.Handler
  if handler == nil {
    handler = DefaultServeMux
  }
  if req.RequestURI == "*" && req.Method == "OPTIONS" {
    handler = globalOptionsHandler{}
  }
  handler.ServeHTTP(rw, req)
}
```

继续跟踪 DefaultServeMux 发现它是 &defaultServeMux 的引用:

```go
var DefaultServeMux = &defaultServeMux
var defaultServeMux ServeMux
```

继续跟踪 ServeMux 发现它的 ServeHTTP 方法实现最后被 NotFoundHandler 所处理，难怪是  404:

```go
func NotFound(w ResponseWriter, r *Request) { Error(w, "404 page not found", StatusNotFound) }
func NotFoundHandler() Handler { return HandlerFunc(NotFound) }
```

直接调用 http.HandleFunc 也是作用在这个 defaultServeMux上。 所以要对请求做处理，其实做一个实现了 ServeHTTP 方法的接口对象就行了，它可以安全被转换为 http.Handler 对象，作为一个适配器，http.HandlerFunc 是 http.Handler 的单函数类型适配，也就是说除了让结构体实现 ServeHTTP 方法外，你可以直接提供一个签名为 http.HandlerFunc 的函数作为处理器:

```go
type HandlerFunc func(ResponseWriter, *Request)
func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) {
  f(w, r)
}
```

小结一下，第一种是提供 http.Handler 对象，然后使用 http.Handler(string,Handler) 来挂载处理器；第二种是提供 http.HandlerFunc(ResponseWriter, *Request) 签名的函数，它们共同的目的都是为了得到一个 ServeHTTP(ResponseWriter, *Request) 函数。


内置的  mux 对路由的支持很简单，在 API 的开发中，很多都遵守 RESTFUL 的设计理念，这时候解析 *Request  的路由就成了很大的体力活儿，需要借助 github.com/julienschmidt/httprouter 模块来帮忙，httprouter 并不是框架，先安装它。

```go
go get github.com/julienschmidt/httprouter
go: downloading github.com/julienschmidt/httprouter v1.3.0
go: github.com/julienschmidt/httprouter upgrade => v1.3.0
```

它的用法很简单:

```go
router := httprouter.New()
router.GET("/", func(w http.ResponseWriter, r *http.Request, ps httprouter.Params) {})
```

用 httprouter 封装后的处理，相比 ServeHTTP 多了 httprouter.Params 对象，它的定义如下:

```go
type Param struct { 
  Key   string 
  Value string
}

type Params []Param
```

定义如下的路由:

```go
func main() {
  router := httprouter.New()
  router.GET("/:name", func(w http.ResponseWriter, r *http.Request, ps httprouter.Params) {
    fmt.Fprintln(w, fmt.Sprintf("%s 你好!", ps.ByName("name")))
  })
  http.ListenAndServe(":3000", router)
}
```

其中 :name 成为路径命令参数，访问 http://localhost:3000/zhangsan 显示 "zhangsan 你好!"，查看 GET 的定义，发现 httpRouter 定义了自己的 Handle 处理器:

```go
type Handle func(http.ResponseWriter, *http.Request, Params)
```

Params 的作用就是解析各种路由参数，包括通配符匹配等，其内部使用了压缩字典树结构，这种数据结构的特点是可以高效的做字符串检索，很多 web 框架都使用了 httprouter，比如 https://github.com/gin-gonic/gin，实现和前面相同功能的例子:

```go
func startGinServer() {
  router := gin.New() // 干净的没有中间件
  // router := gin.Default() // 会包含 logger 和 recover 中间件
  router.GET("/:name", func(c *gin.Context) {
    c.String(http.StatusOK, "%s 你好!", c.Params.ByName("name"))
  })
  router.Run(":3000")
}
```

框架 gin 进一步把 Server 的概念抽象成了 gin.Engine，把请求的上下文抽象成了 gin.Context，提供了输入、输出、插件、验证、安全等很多实用的功能，如果你要做一个正式的 web 项目，可以考虑从 gin 开始。


有了 http.HandlerFunc 和 ServeHTTP 的知识，就可以写 http 中间件了。因为 Handler 的原理是提供函数 func(ResponseWriter, *Request)，所以可以提供一个中间件返回这样的函数就行了，其实就是返回 http.Handler，又因为对别的 Handler 的处理其实就是调用 ServeHTTP 方法，所以中间件的函数可以层层包装，在内部实现对 ServeHTTP 的调用，顺便干一点别的事情。简言之，中间件函数的输入是一个 http.Handler，返回也是一个 http.Handler，下面的中间件会在真正要处理的 handler 调用前和完成后打印一个时间点:

```go
func loggerMiddleware(real http.Handler) http.Handler {
  return http.HandlerFunc(func(writer http.ResponseWriter, request *http.Request) {
    fmt.Fprintln(writer, "before "+time.Now().String())
    real.ServeHTTP(writer, request)
    fmt.Fprintln(writer, "\nafter "+time.Now().String())
  })
}
```

它的用法如下:

```go
func startMiddlewareServer() {
  serve1 := func(w http.ResponseWriter, r *http.Request) {
    w.Write([]byte("我前后有时间"))
  }
  handler := http.HandlerFunc(serve1)
  http.Handle("/1", loggerMiddleware(handler))

  serve2 := func(w http.ResponseWriter, r *http.Request) {
    w.Write([]byte("我前后也有时间"))
  }
  http.Handle("/2", loggerMiddleware(http.HandlerFunc(serve2)))
  //http.HandleFunc("/2", loggerMiddleware(http.HandlerFunc(serve2)).ServeHTTP)

  http.ListenAndServe(":3000", nil)
}
```

访问 [http://localhost:3000/1](http://localhost:3000/1) 输出:

```go
before 2021-01-12 11:56:11.26169 +0800 CST m=+233.747613357
我前后有时间
after 2021-01-12 11:56:11.261703 +0800 CST m=+233.747626890
```

访问 [http://localhost:3000/2](http://localhost:3000/2) 输出:

```go
before 2021-01-12 11:56:50.768871 +0800 CST m=+273.255964104
我前后也有时间
after 2021-01-12 11:56:50.768895 +0800 CST m=+273.255988769
```

中间件运行的顺序是:

```go
request -> logger-before => real.ServeHTTP => logger-after -> response
```

如果再包一层中间件:

```go
request -> newMiddleware-before => logger-before => real.ServeHTTP => logger-after => newMiddleware-after -> response
```

所以中间件是一个洋葱皮的模型，一层一层的，洋葱皮的中心就是被最终包装的 ServeHTTP，外层两边都只是辅助打酱油的，进一步把中间件的类型重新定义一下:

```go
type middleware func(handler http.Handler) http.Handler
```

查看 gin.Default() 函数发现它加载中间价的写法是:

```go
engine := New()
engine.Use(Logger(), Recovery())
```

Logger 和 Recovery 函数都返回 HandlerFunc，Use 函数的参数是 ...HandlerFunc，它在内部使用 append 把所有的 HandlerFunc 都 append 到列表 Handlers 上，形成了新的 HandlersChain 调用链:

```go
type HandlersChain []HandlerFunc
```

最后通过 gin.Engine  的 addRoute 方法映射到路径上:

```go
func (engine *Engine) addRoute(method, path string, handlers HandlersChain) { 
  // ...
}
```

这个 engine 最后怎么被使用的呢? 它其实就是 http.ListenAndServe 的第二个参数，没错，它本身也作为一个 http.Handler 介入，在 ServeHTTP 里完成分发请求和响应。 

```go
// router.Run(":3000")
func (engine *Engine) Run(addr ...string) (err error) { 
  defer func() { 
    debugPrintError(err) 
  }() 
  address := resolveAddress(addr) 
  debugPrint("Listening and serving HTTP on %s\n", address) 
  err = http.ListenAndServe(address, engine) 
  return
}

// Engine as http.Handler
func (engine *Engine) ServeHTTP(w http.ResponseWriter, req *http.Request) { 
  c := engine.pool.Get().(*Context) // 使用了 sync.Pool 减少分配和回收，并发安全
  c.writermem.reset(w) 
  c.Request = req 
  c.reset() 

  engine.handleHTTPRequest(c) // 最后调用点集大成者在此

  engine.pool.Put(c)
}
```

到目前为止，应该可以基于 httprouter 写一个能支持 middleware 的 web 简略框架了。


本章节的代码 [https://github.com/developdeveloper/go-demo/tree/master/19-web-and-middleware](https://github.com/developdeveloper/go-demo/tree/master/19-web-and-middleware)
