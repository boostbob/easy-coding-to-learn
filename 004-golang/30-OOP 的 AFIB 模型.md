从 oop 的角度来看，完整的系统几乎都包含了这四个部分(简称为AIFB):

1. 抽象 Abstract: 对要解决的问题建立一定的机制，为类似的问题提供通用的模型。
2. 实现 Implemention: 有了抽象的模型，但没有提供实际解决问题的方法，所谓实现就是用来提供具体达到目标的方法。
3. 框架 Framework: 有了抽象和实现已经可以解决定义的问题了，但是怎么去使用呢？这就需要提供一定的使用流程来连接。
4. 业务 Business: 有了框架已经可以正常去调用这个系统了，把它应用到具体的场景中，这就是一套面向业务的流程，来解决真正的客户的需求。

从一个很微小的例子来理解这4个层次，假设客户的需求是：用手机给张三打电话。

第一层抽象: 很显然核心的要点是打电话，至于打给谁，用什么手机，这些不是关键点，这个系统如果只能给张三打电话，显然会显得很鸡肋，下次你要给李四打电话，你会重新做一个系统吗? 不会，对不对；如果系统做成只能使用评估手机打电话，那下次调用的时候，调用者没有苹果手机，只有华为手机，那怎么办? 这个系统还不能用了，又得重新做一个，对不对? 所以针对这个需求，可以适当的抽象出: 可以用任意能打电话的手机，给 XXX 打电话。定义一个打电话的行为，这就是适当的抽象: 

```go
// 抽象 Abstract

type Phone interface {
  Call(name string) 
}
```

第二层实现: 有了接口对打电话的行为做出了具体的定义，那么具体由什么手机来打呢? 不同的手机应该有不同的实现。对于苹果手机和华为的手机分别来提供打电话这个行为的方法:

```go
// 实现 Implention

// Apple 苹果手机得这么打
type ApplePhone struct {} 

func (phone *ApplePhone) Call(name string) {
  fmt.Println("apple call " + name)
}

// Huawei 华为手机这么打
type HuaweiPhone struct {} 

func (phone *HuaweiPhone) Call(name string) {
  fmt.Println("huawei call " + name)
}
```

第三层框架: 有了苹果和华为的手机，就能实现打电话这个行为了，但是具体打电话的过程还是没有的，为了让这个行为更易使用，需要一个框架来整合一下，但是框架其实不关心具体打电话的过程内部是怎么实现的，它只完成打电话这个动作:

```go
// 框架 Framework

// phone 参数: 能打电话设备
// names 参数: 要打给谁谁谁
func doCall(phone Phone, names... string) {
  for _, name := range names {
    phone.Call(name)
  }
}
```

第四层业务: 系统功能都具备了，终于可以使用了，在具体场景中去使用吧，给张三打个电话，可以用苹果手机，也能用华为手机:

```go
// 业务  Business

func main() {
  var phone Phone 

  phone = &ApplePhone{}
  doCall(phone, "zhangsan", "lisi")

  phone = &HuaweiPhone{}
  doCall(phone, "zhangsan", "lisi")
}
```

如果将来你换了 Oppo 的手机，你只需要提供一个 OppoPhone 的实现层就行了，代码清晰易改动:

```go
// Oppo 手机得这么打
type OppoPhone struct { }

func (phone *OppoPhone) Call(name string) {
  fmt.Println("oppo call " + name)
}
```

这个例子足够小，但是体现了 oop 设计的理念和常用的模型，不管使用什么语言，什么场景，这套基本的方法流程是通用的。
