Go剖析系列
===
## 简介
从汇编/编译器等多个角度了解Go语言内部原理。  

本书写作时定的目标很宏大，希望能覆盖面试时可能问到的各种内部实现问题(Go语言本身)，目前看来[切片剖析](./articles/2-slice.md)一章节算是"勉强接近"了这一目标，如果读者有什么写作建议或者内容建议，可以联系我或在issue中告知我。

<!--## 在线阅读
[GitBook](https://bootun.gitbook.io/go-analysis/)-->

## 目录
 - [数组剖析](./articles/array.md)
 - [切片剖析](./articles/slice.md)
 - [map剖析(写作中)](./articles/map.md)
 - [channel剖析(写作中)](./articles/channel.md)
 - [附录1:如何寻找源码位置](./articles/appendix/1-source.md)
  
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
本系列文章首发于微信公众号:**梦真日记**，由于微信公众号无法对文章进行更新，所以如果我在学习的过程中对过去的章节有什么新的收获，会通过本仓库进行更新