+++
title = "Go异步日志实现"
date = "2024-05-18T12:47:51+08:00"
tags = ["Go", "并发编程", "问题定位"]
description = "线上IO打满，写日志阻塞，上游超时结束。 实现异步日志，并分析mutex与chan使用问题"

+++

# 一、问题背景

线上某接口偶现pvlost，问题实例物理机所在磁盘都有大量**磁盘IO**，ioutil持续100%

# 二、分析

- 为什么pvlost？
  - header中透传deadline，判断time.Now() > deadline后，直接丢弃请求
  - 框架在收到请求时立即打印一条请求日志
  - 日志库实现类似go标准库的log，使用sync.Mutex保护buf，打印日志被**串行化**
  - 磁盘IO打满时，log.Info("msg")在Write时可能会**阻塞N秒**，后续的请求也被阻塞，打印日志后进入 time.Now() > deadline 的判断时，已经超过了deadline，丢弃请求。对外表现为pvlost

- Write写文件会阻塞？
  - 文件Write只需要写到内核的cache中，由操作系统负责flush，IO压力大时，cache不足，打印日志会阻塞
  - 线上遇到的case，一般**阻塞3秒以下**



写日志阻塞有以下解决方式

1. 硬件层面
   1. 更换SSD：SSD有着更高的读写性能、更高的IOPS
   2. 独立部署：不与其他占用IO大的实例混部
2. 服务层面
   1. 打印log不立即Write，将日志写到ctx上下文中（例如ctx提供Info、Error日志方法），回包后再Write，此时阻塞不影响请求
   2. **异步日志**：日志写到进程缓冲区，异步Write



其实，Write行为可以看作是异步的，内核有page cache，并且会定期flush

但是内核cache不足时，也是会阻塞

因此，异步日志本质是在进程内增加cache，不依赖内核的cache



----



PS：起初并未怀疑是磁盘IO的问题，物理机有多磁盘，监控上需要选中对应磁盘才能看到相应监控

定位问题时最直接的表现是日志中时间乱序，故障时间前后几秒，日志中的时间是乱的，时间变化不合规律

正常的日志是下面这样，时间递增

```
2024-01-01 12:34:05.000 msg...
2024-01-01 12:34:05.100 msg...
2024-01-01 12:34:05.200 msg...
```

故障时，时间从5s到6s到7s，又会回到5s

```
2024-01-01 12:34:05.000 msg...
2024-01-01 12:34:06.100 msg...
2024-01-01 12:34:07.200 msg...
2024-01-01 12:34:05.000 msg...
2024-01-01 12:34:06.100 msg...
2024-01-01 12:34:05.000 msg...
```

有怀疑是时钟波动的问题，但是时钟波动概率还是很小，多次出现故障，并且时钟短时间频繁波动，排除





# 三、实现异步日志

几种解决方式中，异步日志可行性最高。



当前日志库实现整体逻辑类似标准库log，额外增加了**日志滚动**功能，增加Info、Error等日志等级

```
// 标准库log
// A Logger represents an active logging object that generates lines of
// output to an io.Writer. Each logging operation makes a single call to
// the Writer's Write method. A Logger can be used simultaneously from
// multiple goroutines; it guarantees to serialize access to the Writer.
type Logger struct {
	mu        sync.Mutex // ensures atomic writes; protects the following fields
	prefix    string     // prefix on each line to identify the logger (but see Lmsgprefix)
	flag      int        // properties
	out       io.Writer  // destination for output
	buf       []byte     // for accumulating text to write
}
```

二者实现上对比

- 标准库log
  - out为io.Writer

- 我们的日志库
  - out为*os.File，使用到Write写日志、Close方法关闭日志。打开新文件，滚动到新文件



既然只用到Write、Close，那么可以将out定义为io.WriteCloser

提供fileWrapper方法， 将*os.File转为io.WriteCloser

未开启异步log时，out = file，开启异步log时，将file包装为异步WriteCloser



### 3.1 简单实现：利用chan []byte

```

type AsyncWriteCloser struct {
	*os.File
	ch    chan []byte
	close chan error
}

func NewAsyncWriteCloser(f *os.File) *AsyncWriteCloser {
	res := &AsyncWriteCloser{
		File:  f,
		ch:    make(chan []byte, 1024),
		close: make(chan error),
	}
	go res.consume()
	return res
}

func (wc *AsyncWriteCloser) consume() {
	for b := range wc.ch {
		wc.File.Write(b)
	}
	wc.close <- wc.File.Close()
}

func (wc *AsyncWriteCloser) Write(b []byte) (int, error) {
	copy := append([]byte(nil), b...)
	wc.ch <- copy
	return len(b), nil
}

func (wc *AsyncWriteCloser) Close(b []byte) error {
	close(wc.ch)
	return <-wc.close
}
```

注意：Write参数b要进行**深拷贝**。函数返回后，调用函数可以修改b的内容



这个实现：

- Write次数不变，Write在chan未满时立即返回，阻塞只会在consume中
- 多了一次内存拷贝， Write方法中的拷贝
- []byte对象多

### 3.2 优化实现：积攒数据，每秒写一次

先写到cur []byte, 写满cap后再写入chan，减少Write次数，避免产生大量[]byte对象

#### Write逻辑

```
type AsyncWriteCloser2 struct {
	*os.File
	frozen chan []byte

	mu  sync.Mutex
	cur []byte

	closed  bool

	wg sync.WaitGroup
}

func (wc *AsyncWriteCloser2) Write(b []byte) (int, error) {
	wc.mu.Lock()
	defer wc.mu.Unlock()

	if wc.closed {
		return 0, errors.New("write into closed file")
	}

	l := len(b)
	if l+len(wc.cur) <= cap(wc.cur) {
		wc.cur = append(wc.cur, b...)
		return l, nil
	}

	if len(wc.cur) > 0 {
		// 阻塞send
		wc.frozen <- wc.cur
		wc.cur = nil // TODO 池化
	}

	wc.cur = append(wc.cur, b...)
	return l, nil
}
```

此外， 当日志量过少时，写满cap需要一定时间，日志更新慢，需要增加**每秒写一次**的逻辑

#### 消费逻辑

```
func (wc *AsyncWriteCloser2) consume() {
	defer wc.wg.Done()

	ticker := time.NewTicker(time.Second)
	defer ticker.Stop()

	var exit bool
	for !exit {
		select {
		case b, ok := <-wc.frozen:
			if !ok { // closed
				exit = true
				break
			}
			wc.File.Write(b)

		case <-ticker.C:
			if len(wc.frozen) > 0 {
				continue
			}

			wc.mu.Lock()
			// 此时 frozen可能满了
			if len(wc.cur) > 0 {
				select {
				case wc.frozen <- wc.cur: // 要写到frozen 顺序消费
					wc.cur = nil
				default:
					// 已经满了 or closed
				}
			}
			wc.mu.Unlock()
		}
	}
}
```



#### Close等待数据写完

Close时，需要将已有数据写完，因此，在Close方法等待consume完成后调用文件Close

```
func (wc *AsyncWriteCloser2) waitLocked() {
	// cur还有数据， 写到frozen
	if len(wc.cur) > 0 {
		wc.frozen <- wc.cur
		wc.cur = nil
	}

	close(wc.frozen)
	wc.wg.Wait()
}

func (wc *AsyncWriteCloser2) Close(b []byte) error {
	wc.mu.Lock()
	defer wc.mu.Unlock()

	if wc.closed {
		return errors.New("closed")
	}
	wc.closed = true

	wc.waitLocked()

	return wc.File.Close()
}

```

乍一看没有问题，但是，这样可能会**死锁**

waitLocked时持有mutex，当前chan已满，此时写入cur阻塞，等待consume消费fronze

如果consume的select触发ticker分支，ticker会加锁，保护cur，不会消费frozen，也就是waitLocked中无法写入chan

二者互相等待，死锁



问题发生在哪？cur写入chan阻塞？写cur移到File.Close前也会死锁

问题在于sync.Mutex不可重入， Close等待consume退出，consume内会尝试获取Close持有的锁



解决方式

1. consume内使用TryLock，go1.18及以上可使用
2. waitLocked写入chan前，先Unlock
   1. 这个场景可以解决问题，但是不建议
3. consume中select有两个分支，拆成**2个goroutine**，一个负责消费frozen，一个负责ticker加锁将cur写入frozen，Close还是等待consume退出，consume不加锁，不会有问题



拆成2个goroutine，这种方法通用性很强



### 3.3 避免重启丢日志

异步io.WriteCloser只要满足在Close时写完数据即可。

这样，服务重启时，就不可避免丢数据

一种选择提供Flush方法，收到信号Flush，但Flush后还会有日志，不能完全避免日志不丢失

为避免日志丢失，可以提供一个Stop方法（并不只是实现io.WriteCloser了），停止异步写，转变为同步写文件



### 3.4 还能做些什么

增加统计信息，例如Write最大耗时，chan最大长度等



### 3.5 过程中的一些问题

起初尝试用context通知consume退出

- consume的select中可以增加ctx.Done，用来做退出通知
  - select的多个case同时触发时，会随机选择一个
  - 在退出后还要再消费完chan，代码偏复杂
- close(chan) 或者 写入nil 标识关闭，这样退出for循环后一定已经消费完chan



标准库log实现



- go1.18

```

// Output writes the output for a logging event. The string s contains
// the text to print after the prefix specified by the flags of the
// Logger. A newline is appended if the last character of s is not
// already a newline. Calldepth is used to recover the PC and is
// provided for generality, although at the moment on all pre-defined
// paths it will be 2.
func (l *Logger) Output(calldepth int, s string) error {
	now := time.Now() // get this early.
	var file string
	var line int
	l.mu.Lock()
	defer l.mu.Unlock()
	if l.flag&(Lshortfile|Llongfile) != 0 {
		// Release lock while getting caller info - it's expensive.
		l.mu.Unlock()
		var ok bool
		_, file, line, ok = runtime.Caller(calldepth)
		if !ok {
			file = "???"
			line = 0
		}
		l.mu.Lock()
	}
	l.buf = l.buf[:0]
	l.formatHeader(&l.buf, now, file, line)
	l.buf = append(l.buf, s...)
	if len(s) == 0 || s[len(s)-1] != '\n' {
		l.buf = append(l.buf, '\n')
	}
	_, err := l.out.Write(l.buf)
	return err
}
```

mutex临界区较大



注释写到 Release lock while getting caller info - it's expensive.

测试了下runtime.Caller的耗时

```
func main() {
	for i := 0; i < 10; i++ {
		start := time.Now()
		runtime.Caller(2)
		fmt.Println("cost", time.Since(start))
	}
}
```

Caller第一次调用耗时15us，后续在500ns-1us左右。 开销确实大。

同样功能C++可以用宏定义在编译期得到结果



- go1.21

mutex加锁范围变化，临界区变小

增加了[]byte池，不再是每个Logger对象使用一个buf []byte

```
func (l *Logger) output(pc uintptr, calldepth int, appendOutput func([]byte) []byte) error {
	now := time.Now() // get this early.

	// Load prefix and flag once so that their value is consistent within
	// this call regardless of any concurrent changes to their value.
	prefix := l.Prefix()
	flag := l.Flags()

	var file string
	var line int
	if flag&(Lshortfile|Llongfile) != 0 {
		if pc == 0 {
			var ok bool
			_, file, line, ok = runtime.Caller(calldepth)
			if !ok {
				file = "???"
				line = 0
			}
		}
	}

	buf := getBuffer()
	defer putBuffer(buf)
	formatHeader(buf, now, prefix, flag, file, line)
	*buf = appendOutput(*buf)
	if len(*buf) == 0 || (*buf)[len(*buf)-1] != '\n' {
		*buf = append(*buf, '\n')
	}

	l.outMu.Lock()
	defer l.outMu.Unlock()
	_, err := l.out.Write(*buf)
	return err
}

var bufferPool = sync.Pool{New: func() any { return new([]byte) }}

func getBuffer() *[]byte {
	p := bufferPool.Get().(*[]byte)
	*p = (*p)[:0]
	return p
}
```



# 四、总结

使用异步io.WriteCloser，避免打印日志时发生阻塞，避免了接口请求失败



- 遗留问题：sync.Mutex

发生问题时日志可以看出，mutex的等待队列不是先进先出，有着一些随机性

sync.Mutex有饥饿模式 ，饥饿模式下也不是先进先出吗？

怎样复现日志乱序的现象？



后续有时间再写篇博客分析


