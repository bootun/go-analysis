内联优化
===
内联优化是一种常见的编译器优化策略，通俗来讲，就是把函数在它被调用的地方展开，这样可以减少函数调用所带来的开销(栈的创建、参数的拷贝等)。

当函数/方法被内联时，具体是什么样的表现呢？  

## 观察内联
举个例子，现在有以下代码
```go
// ValidateName 验证给定的用户名是否合法
//
//go:noinline
func ValidateName(name string) bool { // AX: 字符串指针  BX: 字符串长度
    if len(name) < 1 {
        return false
    } else if len(name) > 12 {
        return false
    }
    return true
}

//go:noinline
func (s *Server) CreateUser(name string, password string) error {
    if !ValidateName(name) {
        return errors.New("invalid name")
    }
    // ...
    return nil
}

type Server struct{}
```
为了便于理解，我为函数和方法增加了`//go:noinline`注释。Go编译器在遇到该注释时，不会将函数/方法进行内联处理。我们先看一下禁止内联时，该段代码生成的汇编指令:
```go
// ...

// ValidateName函数 
// 此时:
// AX寄存器: 指向name字符串数组的指针
// BX寄存器: name字符串的长度
TEXT github.com/bootun/example/user.ValidateName(SB) github.com/bootun/example/user/user.go
    user.go:9   0x4602c0  MOVQ AX, 0x8(SP) // 保存name字符串的指针到栈上（后面没有用到）
    user.go:10  0x4602c5  TESTQ BX, BX     // BX & BX, 用来检测BX是否为0, 等价于:CMPQ 0, BX
    user.go:10  0x4602c8  JE 0x4602d9      // 如果为0则跳转到0x4602d9
    user.go:12  0x4602ca  CMPQ $0xc, BX    // 比较常数12和name的长度
    user.go:12  0x4602ce  JLE 0x4602d3     // 小于等于12则跳转到0x4602d3
    user.go:13  0x4602d0  XORL AX, AX      // return false
    user.go:13  0x4602d2  RET
    user.go:15  0x4602d3  MOVL $0x1, AX    // return true
    user.go:15  0x4602d8  RET
    user.go:11  0x4602d9  XORL AX, AX      // return false
    user.go:11  0x4602db  RET

// CreateUser方法
TEXT github.com/bootun/example/user.(*Server).CreateUser(SB) /github.com/bootun/example/user/user.go
    // 省略了一些函数调用前的准备工作(寄存器赋值等操作)
    user.go:20    0x460300  CALL user.ValidateName(SB)
    user.go:20    0x460305  TESTL AL, AL
    user.go:20    0x460307  JE 0x460317
    user.go:24    0x460309  XORL AX, AX
    user.go:24    0x46030b  XORL BX, BX
    user.go:24    0x46030d  MOVQ 0x10(SP), BP
    user.go:24    0x460312  ADDQ $0x18, SP
    user.go:24    0x460316  RET
    errors.go:62  0x460317  LEAQ 0x9302(IP), AX
    errors.go:62  0x46031e  NOPW
    errors.go:62  0x460320  CALL runtime.newobject(SB)
    // ...
```
上面的汇编里只截取了最关键的两段: `ValidateName`函数和`CreateUser`方法。

看不懂汇编的同学也没关系，注意看`CreateUser`方法内有一行`user.go:20` `CALL user.ValidateName`, 说明在`CreateUser`方法内调用了`ValidateName`函数，刚好和我们的代码能够对应的上。

现在让我们去掉源代码`ValidateName`函数上的`//go:noinline`再次编译后查看生成的汇编指令：

> 如果你想使用文章里的代码进行尝试，请不要删除`CreateUser`方法上的`//go:noinline`,因为例子中的`CreateUser`太简短了，编译器会把它也内联优化掉，不方便我们进行试验和观察

```go
// CreateUser函数
// 此时:
// AX寄存器: 方法Recever,即Server结构体
// BX寄存器: name字符串的指针
// CX寄存器: name字符串的长度
TEXT github.com/bootun/example/user.(*Server).CreateUser(SB) /github.com/bootun/example/user/user.go

    // ...              
    user.go:18    0x4602d4  MOVQ BX, 0x28(SP)    // 保存name字符串的指针到栈上
    user.go:19    0x4602d9  TESTQ CX, CX         // 验证name的长度是否为0
    user.go:9     0x4602dc  JE 0x4602e6          // 为0则跳转到0x4602e6
    user.go:9     0x4602de  NOPW
    user.go:11    0x4602e0  CMPQ $0xc, CX        // 比较常数12和字符串的长度
    user.go:11    0x4602e4  JLE 0x460318         // 小于等于则跳转到0x460318继续执行(name合法)
    
    errors.go:62  0x4602e6  LEAQ 0x9333(IP), AX  // 构造错误返回                   
    errors.go:62  0x4602ed  CALL runtime.newobject(SB)                  
    errors.go:62  0x4602f2  MOVQ $0xc, 0x8(AX)                    
    // ...
    user.go:23    0x460318  XORL AX, AX      // AX = 0
    user.go:23    0x46031a  XORL BX, BX      // BX = 0				
    user.go:23    0x46031c  MOVQ 0x10(SP), BP  // 恢复BP寄存器
    user.go:23    0x460321  ADDQ $0x18, SP     // 增加栈指针, 减小栈空间
    user.go:23    0x460325  RET                // return
    // ...
```
观察这一次的代码可以发现，`ValidateName`函数的逻辑直接被内嵌到了`CreateUser`方法里展开了。我们在生成的汇编代码里也搜索不到`ValidateName`相关的符号了。
现在的代码等价于:
```go
func (s *Server) CreateUser(name string, password string) error {
    if len(name) < 1 {
        return errors.New("invalid name")
    } else if len(name) > 12 {
        return errors.New("invalid name")
    }
    return nil
}
```

## 什么样的函数会被内联？
内联相关的代码在`cmd/compile/internal/inline/inl.go`里,属于编译器的一部分。在该文件的最上面有这样一段注释, 里面很好的概括了内联的控制和规则:
```go
// The Debug.l flag controls the aggressiveness. Note that main() swaps level 0 and 1,
// making 1 the default and -l disable. Additional levels (beyond -l) may be buggy and
// are not supported.
//      0: disabled
//      1: 80-nodes leaf functions, oneliners, panic, lazy typechecking (default)
//      2: (unassigned)
//      3: (unassigned)
//      4: allow non-leaf functions
//
// At some point this may get another default and become switch-offable with -N.
//
// The -d typcheckinl flag enables early typechecking of all imported bodies,
// which is useful to flush out bugs.
//
// The Debug.m flag enables diagnostic output.  a single -m is useful for verifying
```
总结一下上面这段话中的核心部分:
- **80个节点的叶子函数，oneliners，panic，懒惰的类型检查** 会被内联
- 使用`-N -l`来告诉编译器禁止内联
- 使用`-m`启用诊断输出

也就是说，**只要我们的函数/方法足够小，就可能会被内联。** 因此，很多人会使用许多小的函数组合来代替大段代码提升性能。比如我们经常使用的互斥锁(标准库中`sync`包里的`Mutex`)就利用了这一点, 我们平时使用的`Lock`方法一共就只有这几行:
```go
func (m *Mutex) Lock() {
    // Fast path: grab unlocked mutex.
    if atomic.CompareAndSwapInt32(&m.state, 0, mutexLocked) {
        if race.Enabled {
            race.Acquire(unsafe.Pointer(m))
        }
        return
    }
    // Slow path (outlined so that the fast path can be inlined)
    m.lockSlow()
}
```
注意倒数第三行的注释:`outlined so that the fast path can be inlined`,利用这一特性,`Lock`里的FastPath就能被内联到我们的程序里而不需要额外的函数调用，从而提升代码的性能。

> 函数内联部分的入口是函数[inline.InlinePackage](https://github.com/golang/go/blob/7ad92e95b56019083824492fbec5bb07926d8ebd/src/cmd/compile/internal/gc/main.go#LL282 "内联入口"), 想要深入了解的小伙伴可以去看一看。

## 内联能为我的程序带来多少性能上的提升?
前面介绍了这么多内联，连标准库都刻意使用内联来提升Go程序的性能，那么内联究竟能为我们带来多少性能上的提升呢？

我们来扩充一下文章开篇提到的例子:
```go
package user

import (
      "errors"
)

func ValidateName(name string) bool {
      if len(name) < 1 {
            return false
      } else if len(name) > 12 {
            return false
      }
      return true
}

//go:noinline
func ValidateNameNoInline(name string) bool {
      if len(name) < 1 {
            return false
      } else if len(name) > 12 {
            return false
      }
      return true
}

func (s *Server) CreateUser(name string, password string) error {
      if !ValidateName(name) {
            return errors.New("invalid name")
      }
      return nil
}

// CreateUserNoInline 使用的是禁止内联版本的 ValidateName
func (s *Server) CreateUserNoInline(name string, password string) error {
      if !ValidateNameNoInline(name) {
            return errors.New("invalid name")
      }
      return nil
}

type Server struct{}
```
我们复制了以`ValidateName`函数，在上面标注上`//go:noinline`来禁止编译器对其进行内联优化，并将其并将其更名为`ValidateNameNoInline`。同时我们也复制了`CreateUser`方法，新的方法内部使用`ValidateNameNoInline`来验证`name`参数，除此之外所有的地方都和原方法相同。

我们来写两个Benchmark测试一下:
```go
package user

import "testing"

// BenchmarkCreateUser 测试内联过的函数的性能
func BenchmarkCreateUser(b *testing.B) {
      srv := Server{}
      for i := 0; i < b.N; i++ {
            if err := srv.CreateUser("bootun", "123456"); err != nil {
                  b.Logf("err: %v", err)
            }
      }
}

// BenchmarkValidateNameNoInline 测试函数禁止内联后的性能
func BenchmarkValidateNameNoInline(b *testing.B) {
      srv := Server{}
      for i := 0; i < b.N; i++ {
            if err := srv.CreateUserNoInline("bootun", "123456"); err != nil {
                  b.Logf("err: %v", err)
            }
      }
}
```
测试结果如下:
```sh
# 内联版本的基准测试结果(BenchmarkCreateUser)
goos: windows
goarch: amd64
pkg: github.com/bootun/example/user
cpu: AMD Ryzen 7 6800H with Radeon Graphics
BenchmarkCreateUser
BenchmarkCreateUser-16          1000000000               0.2279 ns/op
PASS


# 禁止内联版本的基准测试结果(BenchmarkValidateNameNoInline)
goos: windows
goarch: amd64
pkg: github.com/bootun/example/user
cpu: AMD Ryzen 7 6800H with Radeon Graphics
BenchmarkValidateNameNoInline
BenchmarkValidateNameNoInline-16        733243102                1.635 ns/op
PASS
```
可以看到,禁止内联后每次操作耗费1.6纳秒，而内联后只需要0.22纳秒(因机器而异)。从比例上看，内联优化带来的收益还是很可观的。

## 我需要做什么来启用内联优化吗?
当然不需要，在Go编译器中，内联优化是默认启用的，如果你的函数符合文中提到的内联优化的策略(比如函数很小)，并且没有显式的禁用内联，就可能会被编译器执行内联优化。

在某些场景下，我们可能不希望函数进行内联(比如使用`dlv`进行DEBUG时，或者查看程序生成的汇编代码时)，可以使用`go build -gcflags='-N -l' xxx.go`来禁用内联优化。

> 编译器默认优化出来的代码可能比较难以阅读和理解，不方便我们进行调试和学习。

> `-gcflags`是传递给go编译器`gc`的命令行标志, `go build` 背后做了很多事，也不止用到了`gc`一个程序。使用`go build -x main.go`可以查看编译过程中的详细步骤。