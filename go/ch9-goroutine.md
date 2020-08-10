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
* 