# sync.Pool

## 什么时候用？

在pprof里看
- 进程中的inuse_objects数过多，**gc mark消耗大量CPU**
- 进程中的inuse_objects数过多，**进程RSS占用过高**

请求生命周期开始时，pool.Get。请求结束时pool.Put。在fasthttp中有大量应用