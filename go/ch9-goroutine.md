### 第九章　基于共享变量的并发


#### 9.1 竞争条件
**并发安全:** 如果在并发的情况下，这个函数依然可以正确地工作的话，那么我们就说这个函数是并发安全的，并发安全的函数需要额外的同步工作。

**根据上述定义，有三种方式可以避免数据竞争：**
* 第一种方法是不要去写变量，此方法只有第一次才会去填写数据

```golang
var icons = make(map[string]image.Image)
func loadIcon(name string) image.Image
// NOTE: not concurrency-safe!
func Icon(name string) image.Image {
    icon, ok := icons[name]
    if !ok {
        icon = loadIcon(name)
        icons[name] = icon
    }
    return icon
}
```
* 第二种避免数据竞争的方法是，避免从多个goroutine访问变量

```golang
package bank
var deposits = make(chan int) // send amount to deposit
var balances = make(chan int) // receive balance
func Deposit(amount int) { deposits <- amount }
func Balance() int { return <-balances }
func teller() {
    var balance int // balance is confined to teller goroutine
    for {
        select {
            case amount := <-deposits: //if desposits have data the aad amount
            balance += amount
            case balances <- balance: //if balance has data then get the balances
        }
    }
}
func init() {
    go teller() // start the monitor goroutine
}
```

* 第三种是一种互斥的方式实现的


#### 9.2 sync.Mutex互斥锁

* 通过信号量实现互斥的方式：

```golang
var (
    sema = make(chan struct{}, 1) // a binary semaphore guarding balance
    balance int
)
func Deposit(amount int) {
    sema <- struct{}{} // acquire token 此时sema已满，如没人读，所有写操作都阻塞
    balance = balance + amount
    <-sema // release token 写操作释放，其他goroutine可以获取到该线程
}
func Balance() int {
    sema <- struct{}{} // acquire token
    b := balance
    <-sema // release token
    return b
}
```

* 通过sync.Mutex实现互斥锁
  * 锁之间的区域叫做临界区
  * defer unlock可以隐式的延伸至函数作用域的最后

```golang
import "sync"
var (
    mu sync.Mutex // guards balance
    balance int
)
func Deposit(amount int) {
    mu.Lock()
    balance = balance + amount
    mu.Unlock()
}
func Balance() int {
    mu.Lock()
    b := balance
    mu.Unlock()
    return b
}
```

#### 9.3 sync.RWMutex读写锁（多读单写锁）

```golang
var mu sync.RWMutex
```

#### 9.4 内存同步

```golang
null
```

#### 9.5 sync.Once初始化
* 读时不锁，写时锁
* 写时锁了还要再次读是否为nil，避免在读取的时候其他goroutine写了该变量
```golang
var mu sync.RWMutex // guards icons
var icons map[string]image.Image
// Concurrency-safe.
func Icon(name string) image.Image {
    mu.RLock()
    if icons != nil {
        icon := icons[name]
        mu.RUnlock()
        return icon
    }
    mu.RUnlock()
    // acquire an exclusive lock
    mu.Lock()
    if icons == nil { // NOTE: must recheck for nil
        loadIcons()
    }
    icon := icons[name]
    mu.Unlock()
    return icon
}
```
* 上述代码可以通过sync.Once来更为简单的实现
  * sync.Once试用于一次性的初始化（通常这种初始化方式是否可以通过在父进程中初始化完成呢，避免多个goroutine去初始化该变量

```golang
var loadIconsOnce sync.Once
var icons map[string]image.Image
// Concurrency-safe.
func Icon(name string) image.Image {
    loadIconsOnce.Do(loadIcons)
    return icons[name]
}
```

* 原理：概念上来讲，一次性的初始化需要一个互斥量mutex和一个boolean变量来记录初始化是不是已经完成了；互斥量用来保护boolean变量和客户端数据结构。
* 每一次对Do(loadIcons)的调用都会锁定mutex，并会检查boolean变量。在第一次调用时，变量的值是false，Do会调用loadIcons并会将boolean设置为true。

#### 9.6 竞争条件检测

* 只要在go build，go run或者go test命令后面加上-race的flag，就会使编译器创建一个你的应用的“修改”版或者一个附带了能够记录所有运行期对共享变量访问工具的test，并且会记录下每一个读或者写共享变量的goroutine的身份信息。

```golang
go run main.go -race
```


#### 9.7 并发的非阻塞缓存

```golang
null
```

#### 9.8 Goroutines和线程

##### 9.8.1 动态栈

goroutine和os线程之间的区别：
1. os线程都会有一个固定大小的内存块（一般是2MB）来做栈。
2. goroutine会以一个很小的栈开始起生命周期，一般只需要2KB，goroutine的栈大小会动态地伸缩。最大可以占用1GB


##### 9.8.2 Goroutine调度
os线程特定：
* os线程被操作系统内核调度，每隔几毫秒，一个硬件计时器会中断处理器，通过scheduler的内核函数，挂起当前执行的线程，并保存内存中它的寄存器内容，检查线程列表并决定下一个线程执行的顺序。
* 上述操作涉及到操作系统的上下文切换，保存一个用户线程的状态到内存，恢复另一个线程到寄存器，然后更新调度器的数据结构，上述操作会消耗大量的内存访问时间。

goroutine特点：
* 