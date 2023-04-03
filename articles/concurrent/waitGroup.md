sync.WaitGroup
===
## 使用方式
WaitGroup的注释已经很好的描述了它的用法：

> `WaitGroup`用来等待goroutine集合完成。主goroutine调用`Add`来设置等待的数量。然后每一个goroutine在运行完成时调用`Done`。`Wait`在所有goroutine都完成之前会一直阻塞。

通俗点说，就是你有**一组任务**要并发的执行，你需要等这些任务都完成，才能够继续做接下来的事情，这时候你就可以考虑使用WaitGroup来等待所有任务的完成。
```go
func main() {
    var wg sync.WaitGroup
    for i:=0;i<100;i++{
        wg.Add(1) // 或直接在循环外wg.Add(100)
        go func(wg *wg.WaitGroup) {
            defer wg.Done()
            // do something
        }(&wg)
    }
    wg.Wait()
}
```

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
`noCopy`是一个空结构体,实现了`Locker`接口，用来避免WaitGroup被复制，感兴趣的读者可以查看我之前[分析noCopy的文章](/articles/go-vet/copylock.md),这里就不做解释了。

剩余两个字段分别是WaitGroup的状态`state`和用作唤醒的信号量`sema`。

`state`字段是`atomic`包下的Uint64类型，这意味着所有对该字段的操作(包括更新、读取等)都是原子性的。