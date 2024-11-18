Channel剖析
===
> TODO: 该章节内大量细节等待裁剪

## channel的结构体表示
channel的运行时结构由`runtime.hchan`来表示:

```go
type hchan struct {
	qcount   uint
	dataqsiz uint
	buf      unsafe.Pointer
	elemsize uint16
	closed   uint32
	elemtype *_type // element type
	sendx    uint   // send index
	recvx    uint   // receive index
	recvq    waitq  // list of recv waiters
	sendq    waitq  // list of send waiters

	lock mutex
}

type waitq struct {
	first *sudog
	last  *sudog
}

type sudog struct {
	// ...
	g *g
	next *sudog
	prev *sudog
    // ...
}
```
我们来看一下各个字段的意义：
- `qcount`字段是当前channel队列里的数据量，也就是channel里目前存了多少个数据。  
- `dataqsiz`指channel的总容量，在创建channel时传入。  
- `buf`是一个指向数组的指针，它的总大小为- `dataqsize*elemsize`，用来存储发给channel的数据。  
- `elemsize`是channel的元素大小。  
- `closed`用来表示当前channel是否关闭。  
- `elemtype`顾名思义就是元素的类型了。  
- `sendx`和`recvx`分别表示发送者和接受者的数据位置(你可以想象成两个指针/下标)。  
- `recvq`和`sendq`分别表示接受者和发送者的等待队列，它是一个(双向)链表数据结构,`waitq`类型里保存了链表的头和尾。而`sudog`里则保存了对应goroutine的`runtime.g`结构体以及链表前后节点的指针。

## 创建channel
`runtime/chan.go`里定义了创建channel的函数:  
```go
func makechan(t *chantype, size int) *hchan {
	elem := t.Elem
	...

	// 计算创建channel所需内存大小
	mem, overflow := math.MulUintptr(elem.Size_, uintptr(size))
	if overflow || mem > maxAlloc-hchanSize || size < 0 {
		panic(plainError("makechan: size out of range"))
	}

	var c *hchan
	switch {
	case mem == 0:
		// Queue or element size is zero.
		c = (*hchan)(mallocgc(hchanSize, nil, true))
		// Race detector uses this location for synchronization.
		c.buf = c.raceaddr()
	case !elem.Pointers():
		// Elements do not contain pointers.
		// Allocate hchan and buf in one call.
		c = (*hchan)(mallocgc(hchanSize+mem, nil, true))
		c.buf = add(unsafe.Pointer(c), hchanSize)
	default:
		// Elements contain pointers.
		c = new(hchan)
		c.buf = mallocgc(mem, elem, true)
	}

	c.elemsize = uint16(elem.Size_)
	c.elemtype = elem
	c.dataqsiz = uint(size)
	lockInit(&c.lock, lockRankHchan)

	return c
}
```
channel的初始化非常简单，主要逻辑就是根据元素类型和元素数量来计算出channel的总内存大小，然后根据内存大小来分配内存并初始化channel的一些字段。

## 向channel中发送数据
`xx <-`语法会被编译器翻译为`runtime.chansend1`, 它底层对应着`runtime.chansend`函数，主要逻辑如下:

```go
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
	if c == nil {
		// 如果向一个nil channel发送数据
		if !block {
			// 如果是非阻塞的发送，直接返回false
			// 注意，只要是通过 xxx <- 这种方式进来的，都是block的
			// 通过select进来的才可能是false
			return false
		}
		// 向一个nil channel 写数据会永远的挂起(阻塞)
		gopark(nil, nil, waitReasonChanSendNilChan, traceBlockForever, 2)
		throw("unreachable")
	}

	// Fast path: check for failed non-blocking operation without acquiring the lock.
	//
	// After observing that the channel is not closed, we observe that the channel is
	// not ready for sending. Each of these observations is a single word-sized read
	// (first c.closed and second full()).
	// Because a closed channel cannot transition from 'ready for sending' to
	// 'not ready for sending', even if the channel is closed between the two observations,
	// they imply a moment between the two when the channel was both not yet closed
	// and not ready for sending. We behave as if we observed the channel at that moment,
	// and report that the send cannot proceed.
	//
	// It is okay if the reads are reordered here: if we observe that the channel is not
	// ready for sending and then observe that it is not closed, that implies that the
	// channel wasn't closed during the first observation. However, nothing here
	// guarantees forward progress. We rely on the side effects of lock release in
	// chanrecv() and closechan() to update this thread's view of c.closed and full().
	if !block && c.closed == 0 && full(c) {
		return false
	}

	var t0 int64
	if blockprofilerate > 0 {
		t0 = cputicks()
	}

	lock(&c.lock)

	if c.closed != 0 {
		unlock(&c.lock)
		panic(plainError("send on closed channel"))
	}

	if sg := c.recvq.dequeue(); sg != nil {
		// Found a waiting receiver. We pass the value we want to send
		// directly to the receiver, bypassing the channel buffer (if any).
		// send底层会调用sendDirect，直接把数据通过memmove直接写入到buf中，然后调用goready唤醒接收者。
		send(c, sg, ep, func() { unlock(&c.lock) }, 3)
		return true
	}

	if c.qcount < c.dataqsiz {
		// Space is available in the channel buffer. Enqueue the element to send.
		qp := chanbuf(c, c.sendx)
		if raceenabled {
			racenotify(c, c.sendx, nil)
		}
		typedmemmove(c.elemtype, qp, ep)
		c.sendx++
		if c.sendx == c.dataqsiz {
			c.sendx = 0
		}
		c.qcount++
		unlock(&c.lock)
		return true
	}

	if !block {
		unlock(&c.lock)
		return false
	}

	// Block on the channel. Some receiver will complete our operation for us.
	gp := getg()
	mysg := acquireSudog()
	mysg.releasetime = 0
	if t0 != 0 {
		mysg.releasetime = -1
	}
	// No stack splits between assigning elem and enqueuing mysg
	// on gp.waiting where copystack can find it.
	mysg.elem = ep
	mysg.waitlink = nil
	mysg.g = gp
	mysg.isSelect = false
	mysg.c = c
	gp.waiting = mysg
	gp.param = nil
	c.sendq.enqueue(mysg)
	// Signal to anyone trying to shrink our stack that we're about
	// to park on a channel. The window between when this G's status
	// changes and when we set gp.activeStackChans is not safe for
	// stack shrinking.
	gp.parkingOnChan.Store(true)
	gopark(chanparkcommit, unsafe.Pointer(&c.lock), waitReasonChanSend, traceBlockChanSend, 2)
	// Ensure the value being sent is kept alive until the
	// receiver copies it out. The sudog has a pointer to the
	// stack object, but sudogs aren't considered as roots of the
	// stack tracer.
	KeepAlive(ep)

	// someone woke us up.
	if mysg != gp.waiting {
		throw("G waiting list is corrupted")
	}
	gp.waiting = nil
	gp.activeStackChans = false
	closed := !mysg.success
	gp.param = nil
	if mysg.releasetime > 0 {
		blockevent(mysg.releasetime-t0, 2)
	}
	mysg.c = nil
	releaseSudog(mysg)
	if closed {
		if c.closed == 0 {
			throw("chansend: spurious wakeup")
		}
		panic(plainError("send on closed channel"))
	}
	return true
}
```

## 从channel中读取数据
`<- xx`这种语法会被翻译为`runtime.chanrecv1`，它是`runtime.chanrecv`的包装函数,`chanrecv`的源码如下:
```go
// chanrecv receives on channel c and writes the received data to ep.
// ep may be nil, in which case received data is ignored.
// If block == false and no elements are available, returns (false, false).
// Otherwise, if c is closed, zeros *ep and returns (true, false).
// Otherwise, fills in *ep with an element and returns (true, true).
// A non-nil ep must point to the heap or the caller's stack.
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
    // channel == nil(尝试读一个空的channel)
    if c == nil {
        // none blocking(非阻塞情况，比如在select里有default分支，就直接返回)
		if !block {
			return
		}
        // 挂起当前goroutine(阻塞)
		gopark(nil, nil, waitReasonChanReceiveNilChan, traceEvGoStop, 2)
		throw("unreachable")
	}

    // 如果读channel非阻塞(select内有default)并且channel此时为空(没数据可以读)
    if !block && empty(c) {
        // 检查channel是否关闭
        if atomic.Load(&c.closed) == 0 {
			return false, false
		}
		// 再次检查channel是否有数据到达
		if empty(c) {
			// ...
			return true, false
		}
    }
    // 加锁
    lock(&c.lock)
    // 如果channel没关闭但里面没数据可以读了
    if c.closed != 0 && c.qcount == 0 {
		if raceenabled {
			raceacquire(c.raceaddr())
		}
        // 释放锁并将ep置为零值
		unlock(&c.lock)
		if ep != nil {
			typedmemclr(c.elemtype, ep)
		}
		return true, false
	}

    // 获取发送队列的第一个sudog结构体
    if sg := c.sendq.dequeue(); sg != nil {
		// 如果缓冲区大小为0，则直接从发送方里接收值，否则
        // 从channel队列的头部接收值，并将发送者的值添加到
        // 队列的末尾。这部分逻辑在recv里。
		recv(c, sg, ep, func() { unlock(&c.lock) }, 3)
		return true, true
	}

    // 如果缓冲区里有值
    if c.qcount > 0 {
		// 直接从队列缓冲区里取值
		qp := chanbuf(c, c.recvx)
		// ...
        // 更新channel的字段
		c.recvx++
		if c.recvx == c.dataqsiz {
			c.recvx = 0
		}
		c.qcount--
        // 释放锁
		unlock(&c.lock)
		return true, true
	}

    // 走到这里说明：
    // 发送队列为空
    // 缓冲区没东西
    // channel不为nil
    
    // 非阻塞接收
	if !block {
		unlock(&c.lock)
		return false, false
	}

    // 后面就是阻塞的逻辑，把当前G(goroutine)打包为一个sudog结构体，挂到channel的接收队列中,然后调用gopark挂起当前goroutine
    // ...
    return true, success
}
```

# 关闭channel
```go
func closechan(c *hchan) {
	if c == nil {
		panic(plainError("close of nil channel"))
	}

	lock(&c.lock)
	if c.closed != 0 {
		unlock(&c.lock)
		panic(plainError("close of closed channel"))
	}

	if raceenabled {
		callerpc := sys.GetCallerPC()
		racewritepc(c.raceaddr(), callerpc, abi.FuncPCABIInternal(closechan))
		racerelease(c.raceaddr())
	}

	c.closed = 1

	var glist gList

	// release all readers
	for {
		sg := c.recvq.dequeue()
		if sg == nil {
			break
		}
		if sg.elem != nil {
			typedmemclr(c.elemtype, sg.elem)
			sg.elem = nil
		}
		if sg.releasetime != 0 {
			sg.releasetime = cputicks()
		}
		gp := sg.g
		gp.param = unsafe.Pointer(sg)
		sg.success = false
		if raceenabled {
			raceacquireg(gp, c.raceaddr())
		}
		glist.push(gp)
	}

	// release all writers (they will panic)
	for {
		sg := c.sendq.dequeue()
		if sg == nil {
			break
		}
		sg.elem = nil
		if sg.releasetime != 0 {
			sg.releasetime = cputicks()
		}
		gp := sg.g
		gp.param = unsafe.Pointer(sg)
		sg.success = false
		if raceenabled {
			raceacquireg(gp, c.raceaddr())
		}
		glist.push(gp)
	}
	unlock(&c.lock)

	// Ready all Gs now that we've dropped the channel lock.
	for !glist.empty() {
		gp := glist.pop()
		gp.schedlink = 0
		goready(gp, 3)
	}
}
```

注释基本已经概括了这个函数的作用:`chanrecv`函数从c中接收数据并写入ep中，ep可能为`nil`,在这种情况下，接收到的数据将被忽略(ignored)。如果`block == false`并且没有元素可用,则返回`false, false`。否则，如果这个channel已经被close了,则将*ep置零，返回`true, false`。否则用一个元素填充ep并返回`true, true`。一个非`nil`的ep一定指向**堆**或者调用者的栈。  

block一般都是true的，也就是阻塞的，只有用作`select`的条件且有`default`分支时，才会传入`false`。

## 读写channel时的结果
| Operation | Channel state | Result |
| ---  | --- | --- |
| Read | nil | Block |
|      | Open and Not Empty | Value |
|      | Open and Empty | Block |
|      | Closed | &lt;default value&gt;, false |
|      | Write Only | Compilation Error |
| --- | --- | --- |
| Write | nil | Block |
|      | Open and Full | Block |
|      | Open and Not Full | Write Value |
|      | Closed | panic |
|      | Received Only | Compilation Error |
| --- | --- | --- |
| Close | nil |  |
|      | Open and Not Empty | Closes Channel; read succeed until channel is drained, then read produce default value |
|      | Open and Empty | Close Channel; read produce default value |
|      | Closed | panic |
|      | Received Only | Compilation Error |

以上内容选自《Concurrency in Go》，描述了在read, write, close各种不同状态的channel时的结果，可以对照一下源码，查看是否正确。