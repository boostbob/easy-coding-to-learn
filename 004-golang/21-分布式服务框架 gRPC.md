gRPC 是谷歌开发的高性能、通用的开源 RPC 框架，使用 http/2 协议设计，默认基于 protobuf 序列化，gRPC 支持众多的编程语言互调。使用之前需要安装 protoc 的编译器。Protocol Buffers 是用 C++ 语言编写的，直接去 https://github.com/protocolbuffers/protobuf/releases 下载二进制版本到本地解压后，移动到系统的 bin 目录，比如 /usr/local/bin 就用了。
![](https://develop-developer.oss-cn-hangzhou.aliyuncs.com/images/zpjzyiEQgsjfAfscx-Bb9P3tF1tx9j_nnC1j89U7gs.png?x-oss-process=style/txt-water)

```bash
Desktop 🍎 protoc --version
libprotoc 3.14.0
```

查看 protoc 的版本则安装成功，要生成对应语言的 proto 代码，还需要安装对应的语言的插件。执行下面的命令安装 go 语言的 protobuf 插件:

```bash
GO111MODULE=off go get github.com/golang/protobuf/protoc-gen-go
```

使用 which 确认一下:

```bash
Desktop 🍎 which protoc-gen-go
/Users/wangbo/.go/bin/protoc-gen-go
```

protobuf 最基本的数据单元叫 message，类似于结构体，不同的语言通过这个 message 定义来生成可以互相调用的代码。

看一个最简单的 hello.proto 的定义:

```
syntax = "proto3";

package hello; // 包名

// 定义一种 String 的类型/结构
message String {
  // 成员  
  string value = 1; // 1 是编码时的编号
}
```

使用 protoc 命令生成 go 的代码:

```bash
hello[master*] 🍎 protoc --go_out=. hello.proto
hello[master*] 🍎 ll
total 8.0K
-rw-r--r-- 1 wangbo staff 2.4K  1 15 11:42 hello.pb.go
-rw-r--r-- 1 wangbo staff   72  1 15 11:42 hello.proto
```

--go_out 表示生成 go 语言的代码，hello.pb.go 就是其生成的文件，里面定义了 String 这个结构体，还增加了一些方法。HellService 现在可以这样写了:

```go
type HelloService struct{}

func (hs *HelloService) Hello(name *String, reply *String) error {
  reply.Value = "hi, " + name.GetValue()
  return nil
}

func startHelloProtoRpcServer() {
  rpc.RegisterName("HelloService", new(HelloService))
  listener, _ := net.Listen("tcp", ":3000")
  conn, _ := listener.Accept()
  rpc.ServeConn(conn)
}
```

服务端和 net/rpc 版本的代码相同，客户端的代码也差不多，换成了 String 类型:

```go
func startHelloProtoRpcClient() {
  var reply String
  client, _ := rpc.Dial("tcp", "127.0.0.1:3000")
  client.Call("HelloService.Say", String{Value: "zhangsan"}, &reply)
  fmt.Println(reply.Value)
}
```

这好像和自己定义一个 String 结构体没有什么区别啊? 确实，上个章节说到如果不需要跨语言用 net/rpc 就很好了，不过它演示了 protobuf 的基本用法。


下面实现一个跨语言版本的 hellorpc，对  hello.proto 增加 Say 方法的定义:

```
syntax = "proto3";

package grpchello;

// 调用参数 
message Person {    
  string name = 1;
}

// 调用结果
message Result {    
  string text = 1;
}

// 调用方法
service HelloService {    
  rpc Say (Person) returns (Result);
}
```

生成 go 语言的定义:

```
protoc --go_out=plugins=grpc:. hello.go.proto
```

打开生成的 hello.pb.go 文件，发现除了定义 Person 和 Result 结构体外，还定义了服务端和客户端的接口:

```go
type HelloServiceServer interface { 
  Say(context.Context, *Person) (*Result, error)
}

type HelloServiceClient interface { 
  Say(ctx context.Context, in *Person, opts ...grpc.CallOption) (*Result, error)
}
```

rpc 的服务方被抽象成了 grpc.Server，客户端其实是包装了 grpc.ClientConn 的对象，还提供了 rpc 的注册函数 RegisterHelloServiceServer，构造一个 grpc.Server 对象，提供一个 HelloServiceServer 的结构体实现 Say 方法就可以启动一个 grpc 的服务端了:

```go 
func startGrpcHelloServer() { 
  grpcServer := grpc.NewServer() 
  RegisterHelloServiceServer(grpcServer, new(HelloServiceServerImpl)) 
  conn, _ := net.Listen("tcp", ":3000") 
  grpcServer.Serve(conn)
}
```

接下来做一个java 的客户端来访问它，首先也得根据 proto 文件生成 java 的代码，和 go 不同的时候，指定了 java 的 package 包名，引入依赖 protobuf-java、grpc-protobuf、grpc-stub，记得要先安装 protoc-gen-grpc-java 插件(编译参考 https://github.com/grpc/grpc-java/blob/master/compiler/README.md，下载参考 https://repo1.maven.org/maven2/io/grpc/protoc-gen-grpc-java/1.35.0/)。

```bash
mkdir -p src/main/java
protoc --java_out=src/main/java hello.java.proto
protoc --plugin=/usr/local/bin/protoc-gen-grpc-java --grpc-java_out=src/main/javahello.java.proto
```

生成的文件:

```
src/main/java
└── com
    └── wangbo
        └── proto
            ├── HelloProto.java
            ├── HelloService.java
            ├── HelloServiceGrpc.java
            ├── Person.java
            ├── PersonOrBuilder.java
            ├── Result.java
            ├── ResultOrBuilder.java
```

启动 java 的客户端访问 go 的服务端:

```java
public class ClientApplication {
    public static void main(String[] args) {
        ManagedChannel channel = ManagedChannelBuilder
                .forAddress("127.0.0.1", 3000)
                .usePlaintext()
                .build();
        HelloServiceGrpc.HelloServiceBlockingStub stub = HelloServiceGrpc.newBlockingStub(channel);
        Person person = Person.newBuilder().setName("zhangsan").build();
        Result result = stub.say(person);
        System.out.println(result.getText());
    }
}
```

调用成功，如果把 go 作为客户端代码也很简单:

```go
func startGrpcHelloClient() { 
  conn, _ := grpc.Dial("127.0.0.1:3000", grpc.WithInsecure()) 
  defer conn.Close()
  stub := NewHelloServiceClient(conn) 
  result, _ := stub.Say(context.Background(), &Person{Name: "zhangsan"}, grpc.EmptyCallOption{}) 
  fmt.Println(result.Text)
}
```

对应的 java 服务端的代码，继承 HelloServiceImplBase 重载 say 方法:

```java
public class ServerApplication {
    public static void main(String[] args) throws InterruptedException, IOException {
        Server server = ServerBuilder.forPort(3000)
                .addService(new HelloServiceGrpc.HelloServiceImplBase() {
                    @Override
                    public void say(com.wangbo.proto.Person person, StreamObserver<com.wangbo.proto.Result> responseObserver) {
                        Result result = Result.newBuilder().setText("hi, " + person.getName()).build();
                        responseObserver.onNext(result);
                        responseObserver.onCompleted();
                    }
                }).build().start();
        server.awaitTermination();
    }
}
```

这两个例子只使用 string 类型，为了跨语言，protobuf 定义了很多种数据类型，更多的用法和例子访问 [https://developers.google.com/protocol-buffers/docs/reference/google.protobuf](https://developers.google.com/protocol-buffers/docs/reference/google.protobuf)，其他语言的 grpc 参考 [https://grpc.io/docs/languages](https://grpc.io/docs/languages)，grpc 还提供了 tls 安全的通讯方案。



本章节的代码 [https://github.com/developdeveloper/go-demo/tree/master/21-distibuted-os-issue](https://github.com/developdeveloper/go-demo/tree/master/21-distibuted-os-issue)
