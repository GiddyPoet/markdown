#### day1

```golang
package http

type Handler interface {
    ServeHTTP(w ResponseWriter, r *Request)
}

func ListenAndServe(address string, h Handler) error
```

上述发现，其中Handler为一个接口，而ListenAndServe第一个入参是路径，而第二个入参是handler,通过自定义handler实现不同的引擎，只要Handler满足接口约定即可


#### go包管理

通常使用go mod进行包管理，具体使用方式：
```golang
go mod init package_name


package_name //此处是文件名，如果包需要公开则可以带url名，如github.com/username/项目名
├── gee //这里是自定义的包类型
│   └── gee.go
├── go.mod 
└── main.go
```

UI下是通过golang封装的http库，仿照gin实现
```golang
package gee

import (
	"fmt"
	"net/http"
)

// HandleFunc defines the requset handler used by gee
type HandleFunc func(http.ResponseWriter, *http.Request)

// Engine is http engine
type Engine struct {
	router map[string]HandleFunc
}

// New is the constructor of gee.Engine
func New() *Engine {
	return &Engine{router: make(map[string]HandleFunc)}
}

func (e *Engine) addRoute(method string, pattern string, handle HandleFunc) {
	key := method + "-" + pattern
	e.router[key] = handle
}

// Get is http get method
func (e *Engine) Get(pattern string, handle HandleFunc) {
	e.addRoute("GET", pattern, handle)
}

// Post is http post method
func (e *Engine) Post(pattern string, handle HandleFunc) {
	e.addRoute("POST", pattern, handle)
}

// Run the engine handle function
func (e *Engine) Run(addr string) error {
	return http.ListenAndServe(addr, e)
}

func (e *Engine) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	key := r.Method + "-" + r.URL.Path
	if handler, ok := e.router[key]; ok {
		handler(w, r)
	} else {
		fmt.Fprintf(w, "404 NOT FOUND: %s\n", r.URL)
	}
}
```


#### day2

##### 设计Context

* 必要性：对于web来说无非就是根据 *http.Request 来构造http.ResponseWriter，但是每次对于上述对象进行操作，其实较为复杂，因此应该将上述对象操作进行封装，避免冗余的操作，如果不进行有效的封装，则用户要进行大量的重复操作。因此根据常用场景进行封装。

封装前后对比：
```golang
// berfore
obj = map[string]interface{}{
    "name": "GiddyPoet",
    "password": "1234",
}

w.Header().Set("Content-Type", "application/json")
w.WriteHeader(http.StatusOk)

// func NewEncoder(w io.Writer) *Encoder NewEncoder创建一个将数据写入w的*Encoder。 其实就相当于fd与流关联了
encoder := json.NewEncoder(w)
// 将数据写入w
if err := encoder.Encode(obj); err !=nil {
    http.Error(w,err.Error(),500)
}

//after
c.JSON(http.StatusOk,gee.H{
    "username":c.PostForm("username"),
    "password":c.PostForm("password"),
})
```

* 针对使用场景，封装*http.Request和http.ResponseWriter的方法，简化相关接口的调用，只是设计 Context 的原因之一。对于框架来说，还需要支撑额外的功能。例如，将来解析动态路由/hello/:name，参数:name的值放在哪呢？再比如，框架需要支持中间件，那中间件产生的信息放在哪呢？Context 随着每一个请求的出现而产生，请求的结束而销毁，和当前请求强相关的信息都应由 Context 承载。Context承载了所有相关的信息