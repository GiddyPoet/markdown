#### rpc入门

**RPC是远程过程调用的简称，是分布式系统中不同节点间流行的通信方式。**

##### RPC版"Hello, World"


本质上rpc库提供了一个接口类型，即实现的Hello方法，只能有两个可序列化的参数，第二个参数为指针，并返回一个error类型，并且方法是公开的。类似于实现http的handle方法，采用的参数时一致的。

* HelloService是一个服务空间，在这个服务空间下提供一个Hello服务，供远程调用。

server代码
```golang
type HelloService struct{}

func (p *HelloService) Hello(request string, reply *string) error {
	*reply = "hello:" + request
	return nil
}

func main() {
	rpc.RegisterName("HelloService", new(HelloService))

	listener, err := net.Listen("tcp", ":1234")
	if err != nil {
		log.Fatal("ListenTCP error:", err)
	}

	conn, err := listener.Accept()
	if err != nil {
		log.Fatal("Accept error:", err)
	}

	rpc.ServeConn(conn)
}
```


* 通过rpc.Dail建立连接
* client.Call调用具体的rpc方法。其中第一个参数为rpc服务空间名和方法名，其他为rpc的参数
client代码
```golang
func main() {
	client, err := rpc.Dial("tcp", "localhost:1234")
	if err != nil {
		log.Fatal("dialing:", err)
	}

	var reply string
	err = client.Call("HelloService.Hello", "hello", &reply)
	if err != nil {
		log.Fatal(err)
	}

	fmt.Println(reply)
}
```

##### 更安全的RPC接口

在涉及RPC的应用中，作为开发人员一般至少有三种角色：
1. RPC方法的开发人员
2. 客户端调用RPC方法的人员
3. 服务端和客户端RPC接口规范的设计人员

对上述代码进行重构：
1. 服务的名字
2. 服务要实现的详细方法列表
3. 注册该类型服务的函数
```golang
// 服务空间
const HelloServiceName = "path/to/pkg.HelloService"


type HelloServiceInterface interface {
	Hello(request string, reply *string) error
}

//通过接口类型封装HelloInterface,只需要传进来的参数实现了该Hello接口
func RegisterHelloService(svc HelloServiceInterface) error {
	return rpc.RegisterName(HelloServiceName, svc)
}
```

对客户端进行重构：
1、定义客户端结构体
2、实现连接rpc服务函数
3、实现调用rpc服务函数

```golang
type HelloServiceClient struct {
	Client *rpc.Client
}

func DailHelloService(network, address string) (*HelloServiceClient, error) {
	c , err := rpc.Dail(network,address)
	if err != nil {
		return nil,err
	}
	return &HelloServiceClient{
		
	}
}
```


##### 跨语言的RPC（自带的json解析器）

标准库的RPC默认采用Go语言特有的gob编码，因此从其它语言调用Go语言实现的RPC服务将比较困难。在互联网的微服务时代，每个RPC以及服务的使用者都可能采用不同的编程语言，因此跨语言是互联网时代RPC的一个首要条件。

解决方案：
**通过自带的jsonrpc来实现跨语言的rpc接口调用**

首先是基于json编码重新实现RPC服务：
```golang
func main() {
	rpc.RegisterName("HelloService", new(HelloService))

	listener, err := net.Listen("tcp", ":1234")
	if err != nil {
		log.Fatal("ListenTCP error:", err)
	}

	for {
		conn, err := listener.Accept()
		if err != nil {
			log.Fatal("Accept error:", err)
		}
		// josnrpc的解码器
		go rpc.ServeCodec(jsonrpc.NewServerCodec(conn))
	}
}
```

然后是实现json版本的客户端：
```golang
func main() {
	conn, err := net.Dial("tcp", "localhost:1234")
	if err != nil {
		log.Fatal("net.Dial:", err)
	}
	//基于该链接建立针对客户端的json编解码器
	client := rpc.NewClientWithCodec(jsonrpc.NewClientCodec(conn))

	var reply string
	err = client.Call("HelloService.Hello", "hello", &reply)
	if err != nil {
		log.Fatal(err)
	}

	fmt.Println(reply)
}
```