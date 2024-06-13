+++
title = "Go sync包分析"
date = "2024-06-13T23:00:00+08:00"
tags = ["并发编程", "读写锁"]
description = "并发编程 读写锁 互斥锁"

+++

# sync.RWMutex

读者写者是经典的进程同步问题

按照实现的优先级可以分为读者优先、读写公平、写者优先

### 读者优先

如下伪码实现

```
type RWMutex struct {
	rmutex sync.Mutex
	wmutex sync.Mutex
	readerCnt int
}

func RLock() {
	rmutex.Lock
	r := ++readerCnt

	if r == 1  {wmutex.Lock} // 应该在rmutex内调用

	rmutex.Unlock
}

func RUnlock() {
	rmutex.Lock
	r := --readerCnt

	if r == 0 {wmutex.Unlock} // 应该在rmutex内调用

	rmutex.Unlock
}

func Lock() {
	wmutex.Lock()
}

func Unlock() {
	wmutex.Unlock()
}
```



注意，RLock中 if r == 1   RUnlock中if r == 0 的逻辑要在持有rmutex时进行

- 如果RLock中 if r==1在rmutex.Unlock之后...
- 如果RUnlock中if r==0在rmutex.Unlock之后...



Rlock 先锁rmutex，读者数量自增， 如果是第一个读者，锁住wmutex

- 如果在RLock前有写者持有写锁，第一个读者阻塞在wmutex.Lock，后续的RLock请求会阻塞在rmutex.Lock
- 如果第一个RLock时没有写者，第一个RLock加wmutex成功， 后续的写会等待，读者可以进入
- 持续有读请求，写者可能会饥饿



### 写者优先

```
type RWMutex struct {
	rcntmu sync.Mutex
	wcntmu sync.Mutex
	wmu sync.Mutex // 写者互斥
	mu sync.mutex  // 读者加锁时需要锁，第一个写进入时锁，写者最后一个退出时解锁
	readerCnt int
	writerCnt int
}

func RLock() {
	mu.Lock  // 第一步获取mu
	rcntmu.Lock
	r := ++readerCnt

	if r == 1  {wmutex.Lock} // 应该在rcntmu内调用

	rcntmu.Unlock

	mu.Unlock
}

func RUnlock() {
	rcntmu.Lock
	r := --readerCnt

	if r == 0 {wmutex.Unlock} // 应该在rcntmu内调用

	rcntmu.Unlock
}

func Lock() {
	wcntmu.Lock
	w := ++writerCnt
	if w == 1 { mu.Lock } // 第一个写者，Lock mu，阻塞后来的读
	wcntmu.Unlock

	wmu.Lock // 写互斥
}

func Unlock() {
	wcntmu.Lock
	w := --writerCnt
	if w == 0 { mu.UnLock }  // 最后一个写退出，解锁
	wcntmu.Unlock

	wmu.Unlock
}
```

读RLock时需要获取mu，

写Lock时，第一个写者竞争mu，持有后等待最后一个写者Unlock时Unlock mu

写者优先级高于读者



## 读写公平

省略



## Go实现

利用原子变量


```
type RWMutex struct {
	w           Mutex        // held if there are pending writers
	writerSem   uint32       // semaphore for writers to wait for completing readers
	readerSem   uint32       // semaphore for readers to wait for completing writers
	readerCount atomic.Int32 // number of pending readers
	readerWait  atomic.Int32 // number of departing readers
}

const rwmutexMaxReaders = 1 << 30

func (rw *RWMutex) RLock() {
	if rw.readerCount.Add(1) < 0 {
		// A writer is pending, wait for it.
		runtime_SemacquireRWMutexR(&rw.readerSem, false, 0)
	}
}

func (rw *RWMutex) RUnlock() {
	if r := rw.readerCount.Add(-1); r < 0 {
		// Outlined slow-path to allow the fast-path to be inlined
		rw.rUnlockSlow(r)
	}
}

func (rw *RWMutex) rUnlockSlow(r int32) {
	// A writer is pending.
	if rw.readerWait.Add(-1) == 0 {
		// The last reader unblocks the writer.
		runtime_Semrelease(&rw.writerSem, false, 1)
	}
}

func (rw *RWMutex) Lock() {
	// First, resolve competition with other writers.
	rw.w.Lock()
	// Announce to readers there is a pending writer.
	r := rw.readerCount.Add(-rwmutexMaxReaders) + rwmutexMaxReaders
	// Wait for active readers.
	if r != 0 && rw.readerWait.Add(r) != 0 {
		runtime_SemacquireRWMutex(&rw.writerSem, false, 0)
	}
}

func (rw *RWMutex) Unlock() {
	// Announce to readers there is no active writer.
	r := rw.readerCount.Add(rwmutexMaxReaders)

	// Unblock blocked readers, if any.
	for i := 0; i < int(r); i++ {
		runtime_Semrelease(&rw.readerSem, false, 0)
	}
	// Allow other writers to proceed.
	rw.w.Unlock()
}
```



RLock增加readerCount，<0表示有写者申请写锁或持有写锁，等待readerSem

RUnlock减少readerCount，<0表示有写者申请写锁或持有写锁，进入慢路径rUnlockSlow

rUnlockSlow内减少readerWait，新值=0时，唤醒writerSem

写者Lock时串行的，写者之间用rw.w互斥

获取rw.w的写者将readerCount减小rwmutexMaxReaders(10.7亿)，并取到减小前的readerCount

如果原来的readerCount>0，表示有读者已获取读锁，将readerWait增加读者数量，Add后不为0睡眠等待writerSem

写者Lock与读者RUnlock有关联

- 写者加锁请求将readerCount减少10.7亿后，新来的读者会等待readerSem
- 现有的读者会进入rUnlockSlow，减少readerWait，=0唤醒writerSem
  - 一种情况是在写者readerWait之前，全部现有读者RUnlock完毕，写者Add后为0，不需要等待writerSem；同时读者rUnlockSlow也不需要唤醒writerSem



写者Unlock时（一定没有正在进行的读者），将readerCount增加10.7亿，得到等待的读者数量

依次唤醒阻塞在readerSem上的读者



Go的实现不是"写者优先"，一个写者持有锁，新的写者与其余读者同时等待时，新的写者并不是高优处理



# sync.Mutex

异步Log文章中有简单分析



# sync.Once

double check



# sync.Map



# atomic.Value