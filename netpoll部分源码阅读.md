
# netpoll源码阅读

## connection

connection的结构定义如下：

```go
// connection is the implement of Connection
type connection struct {
   netFD                       //网络文件描述符的封装，专注于网络IO
   onEvent                     //网络层面的事件触发器，由用户定义，在NetPoll的server初始化的时候，通过option的方式设置
   locker                      //分片锁
   operator        *FDOperator //同样是socket文件描述符的封装，专注与内存层面的IO，里面的OnRead触发器，只有监听描述符（执行accept的文件描述符）才会有，NetPoll通过onRead是否为nil来判断是不是监听文件描述符。
   readTimeout     time.Duration
   readTimer       *time.Timer
   readTrigger     chan struct{}
   waitReadSize    int64
   writeTimeout    time.Duration
   writeTimer      *time.Timer
   writeTrigger    chan error
   inputBuffer     *LinkBuffer //client--->server数据读缓冲区
   outputBuffer    *LinkBuffer //server--->client数据写缓冲区
   inputBarrier    *barrier    //数据读缓冲区，存在的目的是为了适配C语言的数组，方便调用系统调用
   outputBarrier   *barrier    //写缓冲区，存在的目的是为了适配C语言的数组，方便调用系统调用
   supportZeroCopy bool        //是否支持零拷贝
   maxSize         int         // The maximum size of data between two Release().
   bookSize        int         // The size of data that can be read at once.
}
```

connection 是对client与server之间连接的封装，是两者通讯的桥梁。connection一端是poller，代表client往connetion的inputBuffer中写数据集，或者往outputBuffer中读取数据。另一端是user，代表用户程序从inputBuffer中读取数据，或者往outputBuffer中写数据。inputBuffer和outputBuffer是有linkBufferNode组成的链式缓冲区。

```go
func (c *connection) init(conn Conn, opts *options) (err error) {
	// init buffer, barrier, finalizer
    /******************************/
    /*....上面省略connection部分代码*/
    /*****************************/

    //conn是由通过执行accept产生，Conn类型虽然是个接口，但是initNetFD里面有断言，限制了传入的conn必须是*netFD。
	c.initNetFD(conn) // conn must be *netFD{}
    //initFDOperator 初始化operator，里面的文件描述符和conn中的fd是同一个
	c.initFDOperator()
	c.initFinalizer()

	syscall.SetNonblock(c.fd, true)
	// enable TCP_NODELAY by default
	switch c.network {
	case "tcp", "tcp4", "tcp6":
		setTCPNoDelay(c.fd, true)
	}
	// check zero-copy
	if setZeroCopy(c.fd) == nil && setBlockZeroCopySend(c.fd, defaultZeroCopyTimeoutSec, 0) == nil {
		c.supportZeroCopy = true
	}

	// connection initialized and prepare options
	return c.onPrepare(opts)
}

func (c *connection) initNetFD(conn Conn) {
	if nfd, ok := conn.(*netFD); ok {
		c.netFD = *nfd
		return
	}
	c.netFD = netFD{
		fd:         conn.Fd(),
		localAddr:  conn.LocalAddr(),
		remoteAddr: conn.RemoteAddr(),
	}
}

func (c *connection) initFinalizer() {
	c.AddCloseCallback(func(connection Connection) error {
		c.stop(flushing)
		// stop the finalizing state to prevent conn.fill function to be performed
		c.stop(finalizing)
		freeop(c.operator)
		c.netFD.Close()
		c.closeBuffer()
		return nil
	})
}
```

## connection初始化

connection在初始化时，会利用系统调用对文件描述符做一系列的设置

```go
func SetNonblock(fd int, nonblocking bool) (err error) {
	flag, err := fcntl(fd, F_GETFL, 0)
	if err != nil {
		return err
	}
	if nonblocking {
		flag |= O_NONBLOCK
	} else {
		flag &^= O_NONBLOCK
	}
	_, err = fcntl(fd, F_SETFL, flag)
	return err
}

func fcntl(fd int, cmd int, arg int) (val int, err error) {
	r0, _, e1 := Syscall(SYS_FCNTL, uintptr(fd), uintptr(cmd), uintptr(arg))
	val = int(r0)
	if e1 != 0 {
		err = errnoErr(e1)
	}
	return
}
```

```c
 #include <fcntl.h>
 int fcntl(int fd, int cmd, ... /* arg */ );
```

> The bit that enables nonblocking mode for the file. If this bit is set, read requests on the file can return immediately with a failure status if there is no input immediately available, instead of blocking. Likewise, write requests can also return immediately with a failure status if the output can’t be written immediately.

由于go中代码运行在协程中，而对默认情况下对文件描述的读写会阻塞线程，由于netpoll为了性能，读写普遍采用RawsysCall，对于阻塞系统调用，RawsysCall在线程阻塞之前不会调不会主动将协程调度出去，为了线程阻塞导致线程下的所有协程阻塞，所以这里将fd设置成为非阻塞模式，这种模式下当数据未就绪时，errno会返回EINTR

```go
  //....省略
	switch c.network {
	case "tcp", "tcp4", "tcp6":
		setTCPNoDelay(c.fd, true)
	}
  //....
```

```c
#include <sys/types.h>     /* See NOTES */
#include <sys/socket.h>
int getsockopt(int sockfd, int level, int optname, void *optval, socklen_t *optlen);
int setsockopt(int sockfd, int level, int optname, const void *optval, socklen_t optlen);
```

> If set, disable the Nagle algorithm. This means that segments are always sent as soon as possible, even if there is only a small amount of data. When not set, data is buffered until there is a sufficient amount to send out, thereby avoiding the frequent sending of small packets, which results in poor utilization of the network. This option is overridden by TCP_CORK; however, setting this option forces an explicit flush of pending output, even if TCP_CORK is currently set.

nagle算法为了更好地解决利用网络带宽会把小数据包缓存起来发送，直到超过最大报文段长度MSS，这个算法在高并发小数据包场景下会加大client与server之间的延迟，所以把文件描述符设置成o_nodelay，可以关闭tcp传输层的nagle算法。

```go
    //....省略
	// check zero-copy
	if setZeroCopy(c.fd) == nil && setBlockZeroCopySend(c.fd, defaultZeroCopyTimeoutSec, 0) == nil {
		c.supportZeroCopy = true
	}
    //...省略

func setZeroCopy(fd int) error {
	return syscall.SetsockoptInt(fd, syscall.SOL_SOCKET, SO_ZEROCOPY, 1)
}

func setBlockZeroCopySend(fd int, sec, usec int64) error {
	return syscall.SetsockoptTimeval(fd, syscall.SOL_SOCKET, SO_ZEROBLOCKTIMEO, &syscall.Timeval{
		Sec:  sec,
		Usec: usec,
	})
}
```

> The **MSG_ZEROCOPY** flag enables copy avoidance for socket send calls.The feature is currently implemented for TCP sockets.
> Copying large buffers between user process and kernel can be expensive. Linux supports various interfaces that eschew copying, such as sendpage and splice. The MSG_ZEROCOPY flag extends the underlying copy avoidance mechanism to common socket send calls.
> Copy avoidance is not a free lunch. As implemented, with page pinning, it replaces per byte copy cost with page accounting and completion notification overhead. As a result, MSG_ZEROCOPY is generally only effective at writes over around 10 KB.
> Page pinning also changes system call semantics. It temporarily shares the buffer between process and network stack. Unlike with copying, the process cannot immediately overwrite the buffer after system call return without possibly modifying the data in flight. Kernel integrity is not affected, but a buggy program can possibly corrupt its own data stream.
> The kernel returns a notification when it is safe to modify data. Converting an existing application to MSG_ZEROCOPY is not always as trivial as just passing the flag, then.
> [👉🏻传送门](https://www.kernel.org/doc/html/v4.15/networking/msg_zerocopy.html)

netpoll在网络读写时，调用的系统调用是readv和writev，默认情况下这两系统调用并不是零拷贝的。通过 *syscall.SetsockoptInt(fd, syscall.SOL_SOCKET, SO_ZEROCOPY, 1)* 把文件描述符设置成零拷贝模式，但是这会引入额外的额问题: 由于零拷贝绕过了内核，在内存的用户区存存放用户数据的内存区域，在数据没有完全发送到网卡之前，不允许被释放。并且需要引入额外的信号通知机制，在数据发送完成的时候通知用户程序。
netpoll虽然有上面两行设置零拷贝的代码，但是并没有真正使用！！！！！所有引用supportZeroCopy变量的地方都是这样写的：

```go
    //...省略
	var n, err = sendmsg(c.fd, bs, c.outputBarrier.ivs, false && c.supportZeroCopy)
    //...省略
```

我猜可能是netpoll在一开始设计的时候并没有设计成支持零拷贝，上面supportZeroCopy是为了将来支持零拷贝做准备。

看了netpoll设计文档和byteTeah的博客，很多地方提到了零拷贝，但是netpoll在网络读写时，调用的系统调用是readv和writev，默认情况下这两系统调用并不是零拷贝的，一开始我觉得奇怪，明明不是零拷贝为什么宣称是零拷贝呢？后来我发现我理解错了，我原本以为netpoll是支持的零拷贝是类似于sendfile或者mmap那样的零拷贝，然而看了代码发现，拷贝存在于 **内核 -> netpoll -> 用户程序**，netpoll作为桥梁连接内核和用户程序，并没有支持内核到netpoll的零拷贝，它支持的是netpoll与用户程序之间的零拷贝。

后来我又看到了这片文章：

> 字节跳动框架组和字节跳动内核组合作，由内核组提供了同步的接口：当调用 sendmsg 的时候，内核会监听并拦截内核原先给业务的回调，并且在回调完成后才会让 sendmsg 返回。这使得我们无需更改原有模型，可以很方便地接入 ZeroCopy send。同时，字节跳动内核组还实现了基于 unix domain socket 的 ZeroCopy，可以使得业务进程与 Mesh sidecar 之间的通信也达到零拷贝。
> -----------《字节跳动技术团队微信公众号》[👉🏻传送门](https://mp.weixin.qq.com/s/wSaJYg-HqnYY4SdLA2Zzaw)

结论：魔改内核黑科技

## connection接收数据

上文提到了netpoll的数据传递层次是 **内核 <---> connection <--->  用户程序** 这一节写专注于 **内核 -->> connection.intputBuffer**层面的数据接收接口。

```go

// inputs implements FDOperator.
func (c *connection) inputs(vs [][]byte) (rs [][]byte) {
	vs[0] = c.inputBuffer.book(c.bookSize, c.maxSize)
	return vs[:1]
}

func (c *connection) initFDOperator() {
	op := allocop()
	op.FD = c.fd
	op.OnRead, op.OnWrite, op.OnHup = nil, nil, c.onHup
	op.Inputs, op.InputAck = c.inputs, c.inputAck
	op.Outputs, op.OutputAck = c.outputs, c.outputAck   //这里c.inputs函数传递给了op.Inputs函数指针，最终c.inputs会以operator.Inputs()的形式执行。

	// if connection has been registered, must reuse poll here.
	if c.pd != nil && c.pd.operator != nil {
		op.poll = c.pd.operator.poll
	}
	c.operator = op
}

//执行epoll_wait后的对返回的结果进行处理，方法位于poll_default_linux.go中
//关于poller的内容在其他章节，这里只关注operator.Inputs
func (p *defaultPoll) handler(events []epollevent) (closed bool) {
	for i := range events {
		var operator = *(**FDOperator)(unsafe.Pointer(&events[i].data))
		if !operator.do() {
			continue
		}
        //...省略

		evt := events[i].events
		// check poll in
		if evt&syscall.EPOLLIN != 0 {
			if operator.OnRead != nil {
				// for non-connection  //👈🏻英文注释是官方团队在源码中的注释，为啥我感觉他们写反了，这里应该是for-connection，黑人问号？？？
				operator.OnRead(p)     //netpoll中只有执行accpt的文件描述符设置了这个OnRead方法，在这个方法中会建立一个新连接。
			} else {
				// for connection      //👈🏻感觉写反了，应该是for-nonconnection，理由同上👆🏻
				var bs = operator.Inputs(p.barriers[i].bs) //operator.Inputs()的执行时机。
				if len(bs) > 0 {
					var n, err = readv(operator.FD, bs, p.barriers[i].ivs)
					operator.InputAck(n)
					if err != nil && err != syscall.EAGAIN && err != syscall.EINTR {
						log.Printf("readv(fd=%d) failed: %s", operator.FD, err.Error())
						p.appendHup(operator)
						continue
					}
				}
			}
		}

        //...省略
}

// inputAck implements FDOperator.
func (c *connection) inputAck(n int) (err error) {
	if n <= 0 {
		c.inputBuffer.bookAck(0)
		return nil
	}

	// Auto size bookSize.
	if n == c.bookSize && c.bookSize < mallocMax {
		c.bookSize <<= 1
	}

	length, _ := c.inputBuffer.bookAck(n)
	if c.maxSize < length {
		c.maxSize = length
	}
	if c.maxSize > mallocMax {
		c.maxSize = mallocMax
	}

	var needTrigger = true
	if length == n { // first start onRequest
		needTrigger = c.onRequest()
	}
	if needTrigger && length >= int(atomic.LoadInt64(&c.waitReadSize)) {
		c.triggerRead()
	}
	return nil
}
```

inputs会在connection内部的inputBuffer中分配空间，inputBuffer是由linkBufferNode组成的链表，用于管理读写buffer，具体在LinkBuffer章节中描述。

根据上面的代码，我猜netpoll对连接做了抽象，同一个连接从内核视角看连接是FDOperator类型，从用户程序看连接是connection类型。在connection初始化的时候，把connection的inputs方法传递给connection内部的FDOperator的Inputs指针，使inputs可以以FDOperator的身份在poller那一层运行。这样做可以实现更好的解耦和分层。

从handler方法可以看出，对于非连接的读事件，当一个文件描述符上有EPOLLIN事件时，首先调用inputs方法在操作系统的用户区分配空间，input底层低啊用LinkBuffer的book方法，这个方法返回的是一个[]byte切片，把这个片赋值给vs[0]后，给这个一维切片套了一层二维切片的皮，假装他是二维的。然后调用readv函数，这个函数是对readv系统调用的封装。

```go
// readv wraps the readv system call.
// return 0, nil means EOF.
func readv(fd int, bs [][]byte, ivs []syscall.Iovec) (n int, err error) {
	iovLen := iovecs(bs, ivs)
	if iovLen == 0 {
		return 0, nil
	}
	// syscall
	r, _, e := syscall.RawSyscall(syscall.SYS_READV, uintptr(fd), uintptr(unsafe.Pointer(&ivs[0])), uintptr(iovLen))   //采用RaySyscall 避免runtime介入，提高效率。
	resetIovecs(bs, ivs[:iovLen])
	if e != 0 {
		return int(r), syscall.Errno(e)
	}
	return int(r), nil
}

// iovecs limit length to 2GB(2^31)
func iovecs(bs [][]byte, ivs []syscall.Iovec) (iovLen int) {
	totalLen := 0
	//把bs 二维切片转化成为Iovec结构，在做系统调用时需要形成类似于C语言的数据结构
	//C语言没有切片，数组不包含数据长度信息，Iovec中的Base指针指向数组的地址，Len表示数组中有效数据长度
	for i := 0; i < len(bs); i++ {
		chunk := bs[i]
		l := len(chunk)
		if l == 0 {
			continue
		}
		ivs[iovLen].Base = &chunk[0]
		ivs[iovLen].SetLen(l)
		totalLen += l
		iovLen++
	}
	// iovecs limit length to 2GB(2^31)
	if totalLen <= math.MaxInt32 {
		return iovLen
	}
	// reset here
	totalLen = math.MaxInt32
	for i := 0; i < iovLen; i++ {
		l := int(ivs[i].Len)
		if l < totalLen {
			totalLen -= l
			continue
		}
		ivs[i].SetLen(totalLen)
		iovLen = i + 1
		resetIovecs(nil, ivs[iovLen:])
		return iovLen
	}
	return iovLen
}
```

readv系统调用：

```c
#include <sys/uio.h>
ssize_t readv(int fd, const struct iovec *iov, int iovcnt);
```

> The readv() system call reads iovcnt buffers from the file associated with the file descriptor fd into the buffers described by iov ("scatter input").
> The pointer iov points to an array of iovec structures, defined in <sys/uio.h> as:

```c
    struct iovec {
        void  *iov_base;    /* Starting address */
        size_t iov_len;     /* Number of bytes to transfer */
    };
```

> The readv() system call works just like read(2) except that multiple buffers are filled.
> [👉🏻传送门](https://man7.org/linux/man-pages/man2/writev.2.html)

一般read系统调用可以把数据读从内核取到用户区连续的数据，对于想把数据读取到离散内存块就需要对每一个游离的内存块做一次系统调用，readv可以把数据从内核读取到离散的内存块中，不要求连续，可以减少系统调用的次数。但是从上面的代码可以看出目标空间是一个披着二维切片皮的一维切片，所以这里的readv效果和read一样，并不能真正发挥readv作用。

## connection数据读取

这一节讲的是**connection.inputBuffer-->>用户程序**的数据读取过程。上面的inputs方法和下面的Next方法一个王connection的inputBuffer中存放数据，一个往其中读取数据，返回给应用程序，共同组成了生产者消费者模型。

```go
// Next implements Connection.
func (c *connection) Next(n int) (p []byte, err error) {
	if err = c.waitRead(n); err != nil {
		return p, err
	}
	return c.inputBuffer.Next(n)
}

// waitRead will wait full n bytes.
func (c *connection) waitRead(n int) (err error) {
	if n <= c.inputBuffer.Len() {
		return nil
	}
	atomic.StoreInt64(&c.waitReadSize, int64(n))
	defer atomic.StoreInt64(&c.waitReadSize, 0)
	if c.readTimeout > 0 {
		return c.waitReadWithTimeout(n)
	}
	// wait full n
	for c.inputBuffer.Len() < n {
		if c.IsActive() {
			<-c.readTrigger  //协程阻塞，让出cpu
			continue
		}
		// confirm that fd is still valid.
		if atomic.LoadUint32(&c.netFD.closed) == 0 {
			return c.fill(n)
		}
		return Exception(ErrConnClosed, "wait read")
	}
	return nil
}

// waitReadWithTimeout will wait full n bytes or until timeout.
func (c *connection) waitReadWithTimeout(n int) (err error) {
	// set read timeout
	if c.readTimer == nil {
		c.readTimer = time.NewTimer(c.readTimeout)
	} else {
		c.readTimer.Reset(c.readTimeout)
	}

	for c.inputBuffer.Len() < n {
		if !c.IsActive() {
			// cannot return directly, stop timer before !
			// confirm that fd is still valid.
			if atomic.LoadUint32(&c.netFD.closed) == 0 {
				err = c.fill(n)
			} else {
				err = Exception(ErrConnClosed, "wait read")
			}
			break
		}

		select {
		case <-c.readTimer.C:
			// double check if there is enough data to be read
			if c.inputBuffer.Len() >= n {
				return nil
			}
			return Exception(ErrReadTimeout, c.remoteAddr.String())
		case <-c.readTrigger:
			continue
		}
	}

	// clean timer.C
	if !c.readTimer.Stop() {
		<-c.readTimer.C
	}
	return err
}
```

用户程序调用connection的Next方法，参数n表示预期读取的字节数，inputBuffer里面的数据长度不够时，会阻塞在readChannal，go会调度协程让出cpu，readChannel，当poller调用inputAck往inputBuffer写入预期的数据时，调用triggerRead，唤醒上层应用程序，此时应用程序再次从inputBuffer中读取数据。

## connection发送数据

这一节讲的是**connection.outputBuffer-->>内核**的数据发送接口，代码如下：

```go
func (p *defaultPoll) handler(events []epollevent) (closed bool) {
	for i := range events {
		var operator = *(**FDOperator)(unsafe.Pointer(&events[i].data))
		if !operator.do() {
			continue
		}

		//中间代码省略。。。

		// check poll out
		if evt&syscall.EPOLLOUT != 0 {
			if operator.OnWrite != nil {
				// for non-connection
				operator.OnWrite(p)
			} else {
				// for connection
				var bs, supportZeroCopy = operator.Outputs(p.barriers[i].bs)  // 调用的是下面connection的outputs方法
				if len(bs) > 0 {
					// TODO: Let the upper layer pass in whether to use ZeroCopy.
					var n, err = sendmsg(operator.FD, bs, p.barriers[i].ivs, false && supportZeroCopy)
					operator.OutputAck(n) //调用的是下面connection的outputsAck方法
					if err != nil && err != syscall.EAGAIN {
						log.Printf("sendmsg(fd=%d) failed: %s", operator.FD, err.Error())
						p.appendHup(operator)
						continue
					}
				}
			}
		}
		operator.done()
	}
	// hup conns together to avoid blocking the poll.
	p.detaches()
	return false
}

// outputs implements FDOperator.
func (c *connection) outputs(vs [][]byte) (rs [][]byte, supportZeroCopy bool) {
	if c.outputBuffer.IsEmpty() {
		c.rw2r()  //在poller上修改对应的文件描述符，监听可读可写（接收和发送）事件，改成监听读事件，因为监听可写事件只有在outputBuffer上有待发送的数据的情况下才有意义，同时这样可以减轻内核的压力。
		return rs, c.supportZeroCopy
	}
	rs = c.outputBuffer.GetBytes(vs) //把outputBuffer底下的部分节点的buf，用一个二维切片引用起来。
	return rs, c.supportZeroCopy
}

// outputAck implements FDOperator.
func (c *connection) outputAck(n int) (err error) {
	if n > 0 {
		c.outputBuffer.Skip(n) //递进读指针（往内核写数据，对内核来讲是从outputBuffer中读数据，所以这里是递进读指针）
		c.outputBuffer.Release() //释放outputBuffer的head指针到read指针之间的linkBufferNode
	}
	if c.outputBuffer.IsEmpty() {
		c.rw2r()
	}
	return nil
}
```

## connection数据写入

这一节讲的是用户程序-->>connection.outputBuffer的数据写入过程，源码如下:

```go
// WriteString implements Connection.
func (c *connection) WriteString(s string) (n int, err error) {
	return c.outputBuffer.WriteString(s)
}

// WriteString implements Writer.
func (b *LinkBuffer) WriteString(s string) (n int, err error) {
	if len(s) == 0 {
		return
	}
	buf := unsafeStringToSlice(s)
	return b.WriteBinary(buf)
}

// WriteBinary implements Writer.
func (b *LinkBuffer) WriteBinary(p []byte) (n int, err error) {
	n = len(p)
	if n == 0 {
		return
	}
	b.mallocSize += n

	// TODO: Verify that all nocopy is possible under mcache.
	if n > BinaryInplaceThreshold {  //写入大于4K的数据，直接复用上层应用程序传入的数据的内存，但是这样就需要上层层自觉的保证调用完connection的WriteString方法后，不再修改数据，否则存在内存安全问题，因为数据不会被立刻写入到内核。并且依赖go的GC管理内存。
		// expand buffer directly with nocopy
		b.write.next = newLinkBufferNode(0)  //长度为0的节点是只读节点
		b.write = b.write.next				 //向后移动写指针
		b.write.buf, b.write.malloc = p[:0], n  //这里len(b.write.buf)和b.write.malloc出现了分歧，我猜可能是：b.write.buf = p[:0] 是为了防止后续回收内存时，内部的mcache回收这一段内存，毕竟这一块内存是从引用层借来的，不是通过mcache分配出去的。mcache遇到len(buf) == 0的节点就跳过了，不会回收内存。仅仅是猜测，不一定对。
		return n, nil
	}
	// here will copy
	b.growth(n)   //分配一段大小为n字节的连续内存，不过write节点剩余空间不够就新建一个节点。
	malloc := b.write.malloc
	b.write.malloc += n
	return copy(b.write.buf[malloc:b.write.malloc], p), nil
}
```

## LinkBuffer数据读写

```go
// LinkBuffer implements ReadWriter.
type LinkBuffer struct {
	length     int64 //所有节点的可读数据总长度和
	mallocSize int   //可以看做是LinkBuffer对外暴露的可利用空间和，也是所有节点的已分配空间和，但是不代表底层内存空间容量和，因为有些节点的内存利用率并非100%

	head  *linkBufferNode // release head   //指向第一个节点
	read  *linkBufferNode // read head      //读指针
	flush *linkBufferNode // malloc head    //可以看做是读指针和写指针中间的屏障，read指针到flush指针之间的数是可以安全读取的的，flush到write是正在写入的，每次运行LinkBuffer的Flush()方法会，会向后移动flush指针。
	write *linkBufferNode // malloc tail    //写指针，指向最后一个LinkBuffer节点

	caches [][]byte // buf allocated by Next when cross-package, which should be freed when release
	//提供给上层应用程序的内存空间的集合
}

// Next implements Reader.
// Next会移动read指针
func (b *LinkBuffer) Next(n int) (p []byte, err error) {
	if n <= 0 {
		return
	}
	// check whether enough or not.
	if b.Len() < n {
		return p, fmt.Errorf("link buffer next[%d] not enough", n)
	}
	b.recalLen(-n) // re-cal length 可读数据总数减去n

	// single node
	if b.isSingleNode(n) { 
		return b.read.Next(n), nil
	}
	// multiple nodes
	var pIdx int
	if block1k < n && n <= mallocMax {   //1kB到8MB之间，采用mcache内存池分配内存
		p = malloc(n, n)
		b.caches = append(b.caches, p)
	} else {
		p = make([]byte, n)     //超过8MB利用go分配内存
	}
	var l int
	for ack := n; ack > 0; ack = ack - l {
		l = b.read.Len()
		if l >= ack {  //最后一个可读节点
			pIdx += copy(p[pIdx:], b.read.Next(ack))
			break
		} else if l > 0 {
			pIdx += copy(p[pIdx:], b.read.Next(l))
		}
		b.read = b.read.next
	}
	_ = pIdx
	return p, nil
}

// Write implements io.Writer.
func (w *ioWriter) Write(p []byte) (n int, err error) {
	dst, err := w.w.Malloc(len(p))
	if err != nil {
		return 0, err
	}
	n = copy(dst, p)
	err = w.w.Flush()
	if err != nil {
		return 0, err
	}
	return n, nil
}
```
## LinkBuffer内存管理释放
```go
// Release the node that has been read.
// b.flush == nil indicates that this LinkBuffer is created by LinkBuffer.Slice
func (b *LinkBuffer) Release() (err error) {
	//向前推进read指针，并且跳过所有已读节点。
	for b.read != b.flush && b.read.Len() == 0 {
		b.read = b.read.next
	}
	for b.head != b.read { //从head节点开始回收内存
		node := b.head
		b.head = b.head.next
		node.Release()  
	}
	for i := range b.caches { //清空提供给用户程序的空间。
		free(b.caches[i])  //同样采用mcache回收
		b.caches[i] = nil
	}
	b.caches = b.caches[:0]
	return nil
}

// Release consists of two parts:
// 1. reduce the reference count of itself and origin.
// 2. recycle the buf when the reference count is 0.
func (node *linkBufferNode) Release() (err error) {
	// 对于origin节点引用计数减一，到0就回收。对于引用节点需要先处理origin节点，然后回收
	if node.origin != nil {
		node.origin.Release()
	}
	// release self
	if atomic.AddInt32(&node.refer, -1) == 0 {
		// readonly nodes cannot recycle node.buf, other node.buf are recycled to mcache.
		if !node.readonly {
			// linkedPool 回收内存的内存只会把LinkBufferNode放回，实际存放数据的是linkBufferNode内部的buf, 这是一个字节切片，由mcache回收
			free(node.buf)
		}
		node.buf, node.origin, node.next = nil, nil, nil
		linkedPool.Put(node)
	}
	return nil
}
````

## LinkBuffer内存分配

LinkBuffer内存分配方法有两种，一种是book搭配bookAck, 另一种是Malloc搭配MallocAck。其中book搭配bookAck属于私有方法，专供poller在接收客户端数据，往connection的inputBuffer中写的时候调用。Malloc搭配MallocAck专供用用户程序往connection的outputBuffer中写数据的时候调用。

```go

// book will grow and malloc buffer to hold data.
//
// bookSize: The size of data that can be read at once.
// maxSize: The maximum size of data between two Release(). In some cases, this can
//
//	guarantee all data allocated in one node to reduce copy.
//
// book 的作用是在写指指针指向的节点上分配空间，改方法只会在单个节点分配空间，如果最后一个节点空间为0，就新建一个节点，不会跨节点分配空间，确保分配出去的是连续的空间
// maxSize由外部传入，初始值为8KB，后续会自动翻倍扩容，最大8MB
func (b *LinkBuffer) book(bookSize, maxSize int) (p []byte) {
	l := cap(b.write.buf) - b.write.malloc
	// grow linkBuffer
	if l == 0 {
		l = maxSize
		b.write.next = newLinkBufferNode(maxSize)
		b.write = b.write.next
		//write指针前进，此时flush指针可能落后于write指针，因此在下面的bookAck中需要把flush指针向后移动。
	}
	if l > bookSize {
		l = bookSize
	}
	//有可能不走上面的if，这意味着分配的空间可能比预定的空间小
	//Malloc传递出去的是一个切片，但是对这个切片的修改，不会导致write指针指向的节点的buf切片的len的变化，所以有了下面的bookAck方法
	return b.write.Malloc(l)
}

// bookAck will ack the first n malloc bytes and discard the rest.
//
// length: The size of data in inputBuffer. It is used to calculate the maxSize
// bookAck和上面的book方法成对调用
// bookAck用来移动flush指针
func (b *LinkBuffer) bookAck(n int) (length int, err error) {
	b.write.malloc = n + len(b.write.buf)
	b.write.buf = b.write.buf[:b.write.malloc]
	b.flush = b.write

	// re-cal length
	length = b.recalLen(n)
	return length, nil
}

```

```go
// Malloc pre-allocates memory, which is not readable, and becomes readable data after submission(e.g. Flush).
func (b *LinkBuffer) Malloc(n int) (buf []byte, err error) {
	if n <= 0 {
		return
	}
	b.mallocSize += n
	b.growth(n)
	return b.write.Malloc(n), nil
}
// MallocAck will keep the first n malloc bytes and discard the rest.
func (b *LinkBuffer) MallocAck(n int) (err error) {
	if n < 0 {
		return fmt.Errorf("link buffer malloc ack[%d] invalid", n)
	}
	b.mallocSize = n
	b.write = b.flush

	var l int
	for ack := n; ack > 0; ack = ack - l {
		l = b.write.malloc - len(b.write.buf)
		if l >= ack {
			b.write.malloc = ack + len(b.write.buf)
			break
		}
		b.write = b.write.next
	}
	// discard the rest
	for node := b.write.next; node != nil; node = node.next {
		node.off, node.malloc, node.refer, node.buf = 0, 0, 1, node.buf[:0]
	}
	return nil
}

// Malloc 从调用链路看，总是在在linkBuffer的最后一个节点（write节点）分配空间，即便前面的节点可能还有空间
// 调用Malloc之前需要确保想分配的空间不大于最后一个节点的剩余空间，否则会溢出，所以一般和*LinkBuffer的growth方法搭配使用
func (node *linkBufferNode) Malloc(n int) (buf []byte) {
	malloc := node.malloc
	node.malloc += n
	return node.buf[malloc:node.malloc]
}
```

底层的内存由mcache管理。
```go

// malloc limits the cap of the buffer from mcache.
// 超过8MB就让go分配内存，否则利用mcache内存池分配内存
func malloc(size, capacity int) []byte {
	if capacity > mallocMax {
		return make([]byte, size, capacity)
	}
	return mcache.Malloc(size, capacity)
}

// free limits the cap of the buffer from mcache.
// 超过8MB就让go进行垃圾回收
// 不超过8MB的就利用mcache内存池回收，回收的条件是切片的cap是2的整数次方，否则仍旧利用go进行GC
func free(buf []byte) {
	if cap(buf) > mallocMax {
		return
	}
	mcache.Free(buf)
}

// Malloc supports one or two integer argument.
// The size specifies the length of the returned slice, which means len(ret) == size.
// A second integer argument may be provided to specify the minimum capacity, which means cap(ret) >= cap.

// mcache底层是一个[46]sync.Pool的数组，分别对应cap为2^0次方的切片的sync.Pool，cap为2^1次方的切片的sync.Pool......到2^45次方切片的sync.Pool
// 虽然macahe支持分配2^45次方（32TB）的切片, 但是超过调用层面做了限制，超过maxSiz的不会动用mcache，而是采用多个节点的组合的方案
// 分配内存时会把capacity向上圆整成2的整数次方
func Malloc(size int, capacity ...int) []byte {
	if len(capacity) > 1 {
		panic("too many arguments to Malloc")
	}
	var c = size
	if len(capacity) > 0 && capacity[0] > size {
		c = capacity[0]
	}
	var ret = caches[calcIndex(c)].Get().([]byte)
	ret = ret[:size]
	return ret
}

// Free should be called when the buf is no longer used.
func Free(buf []byte) {
	size := cap(buf)
	if !isPowerOfTwo(size) {
		return
	}
	buf = buf[:0]
	caches[bsr(size)].Put(buf)
}

```