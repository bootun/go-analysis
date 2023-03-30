不要复制我的结构体!
===

## 不允许复制的结构体
sync包中的许多结构都是不允许拷贝的，比如`sync.Cond`,`sync.WaitGroup`,`sync.Pool`, 以及sync包中的各种锁，因为它们自身存储了一些状态(比如等待者的数量)，如果你尝试复制这些结构体:
```go
var wg1 sync.WaitGroup
wg2 := wg1 // 将 wg1 复制一份，命名为 wg2
wg2.Wait()
```

那么你将在你的 IDE 中看到一个醒目的警告:

```go
assignment copies lock value to wg2: sync.WaitGroup contains sync.noCopy
```

IDE是如何实现这一点的呢？我们自己又能否利用这一机制来告诉别人，不要拷贝某个结构体呢？

## 实现原理

大部分编辑器/IDE都会在你的代码上运行`go vet`,vet是Go官方提供的**静态分析工具**，我们刚刚得到的提示信息就是vet分析代码后告诉我们的。vet的实现在Go源码的`cmd/vet`中,里面注册了很多不同类型的分析器，其中`copylock`这个分析器会检查实现了`Lock`和`Unlock`方法的结构体是否被复制。

`copylock Analyser`在`cmd/vet`中注册，具体实现代码在`golang.org/x/tools/go/analysis/passes/copylock/copylock.go`中, 这里只摘抄部分核心代码进行解释:
```go
var lockerType *types.Interface

func init() {
    //...
    methods := []*types.Func{
        types.NewFunc(token.NoPos, nil, "Lock", nullary),
        types.NewFunc(token.NoPos, nil, "Unlock", nullary),
    }
    // Locker 结构包括了 Lock 和 Unlock 两个方法
    lockerType = types.NewInterface(methods, nil).Complete()
}
```

`init`函数中把包级别的全局变量`lockerType`进行了初始化，`lockerType`内包含了两个方法: `Lock`和`Unlock`, 只有实现了这两个方法的结构体才是`copylock Analyzer`要处理的对象。
```go
// lockPath 省略了参数部分，只保留了最核心的逻辑，
// 用来检测某个类型是否实现了Locker接口(Lock和Unlock方法)
func lockPath(...) typePath {
    // ...
    // 如果传进来的指针类型实现了Locker接口, 就返回这个类型的信息
    if types.Implements(types.NewPointer(typ), lockerType) && !types.Implements(typ, lockerType) {
        return []string{typ.String()}
    }
    // ...
}

// checkCopyLocksAssign 检查赋值操作是否复制了一个锁
func checkCopyLocksAssign(pass *analysis.Pass, as *ast.AssignStmt) {
    for i, x := range as.Rhs {
        // 如果等号右边的结构体里有字段实现了Lock/Unlock的话,就输出警告信息
        if path := lockPathRhs(pass, x); path != nil {
            pass.ReportRangef(x, "assignment copies lock value to %v: %v", analysisutil.Format(pass.Fset, as.Lhs[i]), path)
        }
    }
}
```
在`copylock.go`中不仅有赋值检查的逻辑，其他可能会导致结构体被复制的方式它也进行了处理，例如 函数传参、函数调用等，这里就不一一解释了，感兴趣的同学可以自行查看源码。

## 结论
只要你的IDE会帮你运行`go vet`(目前主流的VSCode和GoLand都会自动帮你运行)，你就能通过这个机制来尽量避免复制结构体。  

如果你的结构体也因为某些原因，不希望使用者复制，你也可以使用该机制来警告使用者:

定义一个实现了`Lock`和`Unlock`的结构体
```go
type noCopy struct{}

func (*noCopy) Lock()   {}
func (*noCopy) Unlock() {}
```
将其嵌入你的结构体中:
```go
// Foo 代表你不希望别人复制的结构体
type Foo struct {
    noCopy
    // ...
}
```
或直接让你的结构体实现`Lock`和`Unlock`方法:
```go
type Foo struct {
    // ...
}

func (*Foo) Lock()   {}
func (*Foo) Unlock() {}
```

这样别人在复制`Foo`的时候，就会得到IDE的警告信息了。