sync.WaitGroup
===
## 使用
WaitGroup的注释已经很好的描述了它的用法：

> `WaitGroup`用来等待goroutine集合完成。主goroutine调用`Add`来设置等待的数量。然后每一个goroutine在运行完成时调用`Done`。`Wait`在所有goroutine都完成之前会一直阻塞。

## 内部实现
WaitGroup的实现比较简单，整个`waitgroup.go`带上注释也只有100多行。  
WaitGroup结构体的定义如下：
```go
type WaitGroup struct {
	noCopy noCopy

	state atomic.Uint64 // high 32 bits are counter, low 32 bits are waiter count.
	sema  uint32
}
```
