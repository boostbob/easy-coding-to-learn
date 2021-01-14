RPC 意为远程过程调用或者远程方法调用，这里说的远程可能是本机的另外一个进程，但大多场景是远程的一台 tcp 服务器，Web HTTP Api 访问虽然方便，但是面对复杂的业务的时候封装查询参数往往就很复杂了，RPC 调用在调用方生成动态代理接口对象，调用远程的方法就就像是调用本地方法一样，提高了易用性。


动态代理接口对象的主要工作是:  
1. 识别要访问的远程方法的 IP 和端口  
2. 将调用方法名、参数进行序列化  
3. 将通讯的请求发送给远端服务器  
4. 接受远程服务器返回的调用结果  


这个过程比较重要的是通讯协议和序列化协议，通讯就是 tcp 连接通讯，而序列化是将对象的状态信息转换为可传输和可存储的过程，反序化就是从这些可传输、可存储的数据中恢复对象实例和它的状态，甚至是方法定义。在 go 语言中，这个过程使用 encoding/gob 完成，gob 对 go 就像 Serialization 对 java、pickle 对 python，这些序列化方案都是语言内部的，需要跨语言的描述就需要 xml、json、protocol buffers 序列化方案了。


下面的例子把 person 结构体实例序列化到终端显示，由于序列化的结果并不全是文本，所以显示不规整:

```go
type person struct {
  Name string
  Age  int
}

p1 := person{"zhangsan", 20}
enc1 := gob.NewEncoder(os.Stdout)
enc1.Encode(p1)
```

当然也能存成文件:

```go
file1, _ := os.OpenFile("/tmp/person.gob", os.O_CREATE|os.O_WRONLY, 0644)
enc2 := gob.NewEncoder(file1)
enc2.Encode(p1)
file1.Close()
```

这时候如果把 /tmp/person.gob 通过 qq 传给好友，那么好友也能读取这个文件恢复成一个 person 结构体对象:

```go
var p2 person
file2, _ := os.Open("/tmp/person.gob")
defer file2.Close()
dec := gob.NewDecoder(file2)
dec.Decode(&p2)
```

在 PRC 远程调用的过程中，对象也是通过被序列化后通过网络传输给服务方，服务方把序列化的数据恢复成对应的结构体对象，用传过来的参数调用它的方法后返回结果，调用方只需要知道方法的签名就行了，不必知道它的具体实现过程，这个叫存根(stub)。


go 语言的 net/rpc 包提供了编写 RPC 调用的支持，首先定义一下 rpc 的服务端，它是真正提供实现的一方:

```go
type HelloService struct{}

func (hs *HelloService) Say(name string, reply *string) error {
  *reply = "hi, " + name
  return nil
}

func startHelloRpcServer() {
  rpc.RegisterName("HelloService", new(HelloService))
  listener, _ := net.Listen("tcp", ":3000")

  //使用 for 循环服务多个客户端
  conn, _ := listener.Accept()
  rpc.ServeConn(conn)
}
```

新的方法是 rpc.RegisterName 和 rpc.ServeConn，先注册 rpc 的服务名(RegisterName 用于指定名称，Register 函数利用反射获得名称)，然后使用 conn 连接构造 rpc 服务，事实上 ServeConn 函数接受的参数类型是  io.ReadWriteCloser，F12 查看 ServeConn 的内部实现，可以看到 gob 编解码:

```go
func ServeConn(conn io.ReadWriteCloser) {
  DefaultServer.ServeConn(conn)
}

func (server *Server) ServeConn(conn io.ReadWriteCloser) {
  buf := bufio.NewWriter(conn)
  srv := &gobServerCodec{
    rwc:    conn,
    dec:    gob.NewDecoder(conn),
    enc:    gob.NewEncoder(buf),
    encBuf: buf,
  }
  server.ServeCodec(srv)
}
```

在内部它调用了 ServeCodec 来实现，这个函数接受一个 ServerCodec 对象，表示具体使用哪一种序列化的编解码方案，这里默认的是 gob，而 gob 是 go 语言内部的，它不能跨语言，如果需要以 json 序列化传输对象和参数，就可以直接使用这个函数:

```go
// rpc.ServeConn(conn)
// 使用 json 编码
rpc.ServeCodec(jsonrpc.NewServerCodec(conn))
```

而 json 序列化的实现是: 

```go
func NewServerCodec(conn io.ReadWriteCloser) rpc.ServerCodec { 
  return &serverCodec{  
    dec:     json.NewDecoder(conn),
    enc:     json.NewEncoder(conn),
    c:       conn,
    pending: make(map[uint64]*json.RawMessage),
  }
}
```

有了 rpc 的服务端，客户端就可以调用了，很显然就像网络编程一样，客户端需要先 dial 到服务方建一个连接，然后把调用的方法名和参数都序列化后传过去:

```go
var reply string
client, _ := rpc.Dial("tcp", "127.0.0.1:3000")
client.Call("HelloService.Say", "zhangsan", &reply)
fmt.Println(reply)
client.Call 真正调用了远程方法，第一参数是方法名，第二个是调用参数，它对应着服务方的第一个参数，第三个参数是调用结果，对应服务方的第二个参数，F12 查看 Call 的实现:

func (client *Client) Call(serviceMethod string, args interface{}, reply interface{}) error {
  call := <-client.Go(serviceMethod, args, reply, make(chan *Call, 1)).Done 
  return call.Error
}
```

它在内部通过  Go 函数实现，返回了一个通道一直等待，所以 Call 是一个同步调用，如果希望异步就可直接调用 Go 方法自己拿到通道进行处理:


```go
done := client.Go("HelloService.Say", "zhangsan", &reply, nil).Done
// 继续做其它的事情
<-done
```

这个例子只有一个参数，如果调用需要多个参数则么办? 直接在 Say 方法中增加参数是行不通的，go 语言对 rpc 远程方法做了规定:  
1. 只允许有 2 个参数，第二个参数必须是指针类型  
2. 必须返回 error 类型  


所有要支持多个参数，第一个参数必须修改成结构体类型，看下面的 math 的例子:

```go
type MathService struct{}

func (ms *MathService) Calc(expr Expr, reply *int) error {
  switch expr.Method {
  case "add":
    *reply = expr.Left + expr.Right
  case "mul":
    *reply = expr.Left * expr.Right
  }
  return nil
}

func startMathRpcServer() {
  rpc.RegisterName("MathService", new(MathService))
  listener, _ := net.Listen("tcp", ":3000")

  //使用 for 循环服务多个客户端
  conn, _ := listener.Accept()
  rpc.ServeConn(conn)
}
```

这里的 Calc 是暴露的远程方法，它的第一个参数 expr 是结构体类型，定义了运算符号和操作数:

```go
type Expr struct {
  Method string
  Left   int
  Right  int
}
```

客户端的调用示例:

```go
expr := Expr{"add", 1, 2}
client.Call("MathService.Calc", expr, &reply)
fmt.Println(reply)
```

这两个例子的服务方都是 tcp 的监听方，其实远程方法的提供者只是基于 io.ReadWriteCloser 实例，这里是 tcp.Conn 对象，所以其实 rpc 的服务方也可以是 tcp 的客户端对象，比如:

```go
func startProxyRpcServer() {
  rpc.Register(new(HelloService))

  for {
    // 反过来拨号到外网的 ip 地址上
    conn, err := net.Dial("tcp", "127.0.0.1:3000")
    // 外网客户端还未监听连接失败
    if err != nil {
      time.Sleep(1 * time.Second)
      continue
    }

    rpc.ServeConn(conn)
    conn.Close()
  }
}
```

这个 tcp 一直尝试去连接本机的 3000  端口，它是 tcp 客户端，但是当连接上了，它也使用 conn 对象来提供 rpc 调用服务，完成后关闭 conn 连接，这时候客户端 rpc 的调用其实是开启 tcp 监听，它是调用方但是它不主动，等待被连接，它也不知道哪个 rpc 提供方会来服务，这个过程相当于反向代理:

```go
func startProxyRpcClient() {
  var reply string

  // 外网的客户端主动提供 tcp 服务等待连接
  listener, _ := net.Listen("tcp", ":3000")
  conn, _ := listener.Accept()

  // 构建 rpc 客户端对象
  client := rpc.NewClient(conn)
  defer client.Close()
  client.Call("HelloService.Say", "zhangsan", &reply)
  fmt.Println(reply)
}
```

和前面 rpc 调用方的区别是，首先使用 rpc.NewClient 构建一个 rpc 客户端对象，使用 Call 发起调用。


如果你已经写了一个 rpc 服务，现在需要提供 http 的版本怎么办? 是不是在 http 内部再调用一下 rpc 服务方呢，显然有点麻烦。能不能把在 http 的方法内直接嫁接到 rpc 服务方呢? 可以，不过这个前提是使用 xml、json 的序列化方案，go rpc 提供 ServeRequest 函数来嫁接:

```go
func startHttpJsonRpcServer() {
  rpc.RegisterName("HelloService", new(HelloService))

  http.HandleFunc("/say", func(writer http.ResponseWriter, request *http.Request) {
    var conn io.ReadWriteCloser = struct {
      io.Writer
      io.ReadCloser
    }{
      writer,
      request.Body,
    }

    rpc.ServeRequest(jsonrpc.NewServerCodec(conn))
  })

  // curl localhost:3000/say -X POST --data '{"method":"HelloService.Say","params":["zhangsan"],"id":0}'
  http.ListenAndServe(":3000", nil)
}
```

调用这个 http 方法的时候，需要传递序列化的调用语义: 

```go
{"method":"HelloService.Say","params":["zhangsan"],"id":0}
```

这里的 id 是调用的表示符，因为网络传输的原因，先发起的调用很可能后返回结果，需要一个 id 来鉴别是本次调用。


go rpc 提供了简洁的实现方案，在正式项目中常常需要跨语言的 rpc 调用，而 json 序列化的结果是全文本，传输效果过低，因此都大多使用 Protobuf 的方案，它是一个中间的描述语言，定义了调用的数据结构和消息(方法)，使用编号来绑定数据，序列化后的数据字节数更少，调用方和服务方靠这个中间文件各自生成对应语言的代码，这样即实现了高效调用也实现了跨语言，具体参考 github.com/golang/protobuf/protoc-gen-go。基于 Protobuf 谷歌开发了 gRPC 开源框架，基于 http/2 协议提供服务。请注意如果不需要跨语言调用，go 自带的 net/rpc 是非常好的方案。


本章节的代码  [https://github.com/developdeveloper/go-demo/tree/master/20-rpc-distibuted-os](https://github.com/developdeveloper/go-demo/tree/master/20-rpc-distibuted-os)
