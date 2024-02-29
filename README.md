Go剖析系列
===
## 简介
从编译器/runtime/程序员等多个角度了解Go语言内部原理。

## 目录
### 编译器
 - [内联优化](./articles/compiler/inline.md)
### 数据结构
 - [切片剖析](./articles/container/slice.md)
 - [数组剖析](./articles/container/array.md)
 - [map剖析(写作中)](./articles/container/map.md)
 
### 并发
- [channel剖析(写作中)](./articles/concurrent/channel.md)
<!-- - [select](./articles/select.md) -->
<!-- - [WaitGroup]() -->

### 静态分析

### 附录
 - [附录1: 如何寻找源码位置](./articles/appendix/1-source.md)
 - [静态分析: 不要复制我的结构体!](./articles/go-vet/copylock.md)
## 推荐阅读
**排名不分先后**

 - [Go语言原本](https://golang.design/under-the-hood/)
 - [Go语言问题集](https://golang.design/go-questions/)
 - [Go语言高性能编程](https://geektutu.com/post/high-performance-go.html)
 - [Golang编译器代码浅析](https://gocompiler.shizhz.me/)
 - [Go语言设计与实现](https://draveness.me/golang/)
 - [Go语言高级编程](https://github.com/chai2010/advanced-go-programming-book)
 - [Go语言规范](https://go.dev/ref/spec#Slice_types)
 - [Go语法树入门](https://github.com/chai2010/go-ast-book)

## 关于
本系列文章首发于微信公众号:**比特要塞**，由于微信公众号无法对文章进行更新，所以如果我在学习的过程中对过去的章节有什么新的收获，会通过本仓库进行更新
