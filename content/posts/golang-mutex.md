+++
title = "Golang Mutex"
date = "2024-05-20T23:32:13+08:00"
tags = ["Go", "mutex", "并发编程", "futex"]
description = "Golang sync.Mutex 源码分析， 普通模式 饥饿模式"

+++

# 问题引入

上一篇博客提到IO阻塞，打印log时，时间可能是乱序的

由此来分析Go中sync.Mutex在不同场景的表现

测试代码 go1.19，在每分钟0s Write时阻塞3s

```
package main

import (
	"log"
	"math/rand"
	"os"
	"sync/atomic"
	"time"
)

type DelayWriter struct {
	*os.File
}

var timeslot [60]int64

func (w *DelayWriter) Write(b []byte) (int, error) {
	now := time.Now()
	if now.Second() == 0 && atomic.AddInt64(&timeslot[now.Minute()], 1) == 1 {
		time.Sleep(time.Millisecond * 3333) // 每分钟 0s，模拟写日志阻塞
	}
	return w.File.Write(b)
}

func NewDelayWriter(filename string) *DelayWriter {
	f, err := os.OpenFile(filename, os.O_CREATE|os.O_RDWR|os.O_APPEND|os.O_TRUNC, 0666)
	if err != nil {
		panic(err)
	}
	return &DelayWriter{f}
}

var logger = log.New(NewDelayWriter("test.log"), "", log.LstdFlags|log.Lmicroseconds|log.Lshortfile)

func main() {
	for i := 0; i < 100; i++ {
		go func(index int) {
			var seq int
			for {
				seq++

				r := rand.Int31n(1000) + 100
				time.Sleep(time.Duration(r) * time.Millisecond)

				go serve(index, seq) // 处理请求
			}
		}(i)
	}

	time.Sleep(time.Minute * 2)
	logger.Println("end")
}

func serve(index int, seq int) {
	logger.Println("recv request", index, seq)
}
```

运行结果，可以看到在0s附近，时间有递增、递减、反复变化

![log](/img/golang-mutex-log.jpg)

# sync.Mutex实现分析

当前sync.Mutex的实现比较复杂，引入了普通（normal）模式，饥饿（starvation）模式

sync.Mutex代码经过了4个版本变化，直接看最新代码较难理解，接下来按照4个版本变化依次介绍

## 信号量原语

```
// Semacquire waits until *s > 0 and then atomically decrements it.
// It is intended as a simple sleep primitive for use by the synchronization
// library and should not be used directly.
func runtime_Semacquire(s *uint32)

// Semrelease atomically increments *s and notifies a waiting goroutine
// if one is blocked in Semacquire.
// It is intended as a simple wakeup primitive for use by the synchronization
// library and should not be used directly.
func runtime_Semrelease(s *uint32)
```



## V1：Simple implementation.

```
type Mutex struct {
    key  int32 // Indication of whether the lock is held
    sema int32 // Semaphore dedicated to block/wake up goroutine
}

func (m *Mutex) Lock() {
    if atomic.AddInt32(&m.key, 1) == 1 {
        return
    }
    semacquire(&m.sema)
}

func (m *Mutex) Unlock() {
    if atomic.AddInt32(&m.key, -1) == 0 {
        return
    }
    semrelease(&m.sema)
}
```

### Lock

- key原子增加
  - 新值为1获取到锁；否则：
  - 信号量semacquire

### Unlock

- key原子增加-1
  - 新值为0，说明没有其他在等待的，直接返回；否则：
  - 信号量semrelease，唤醒等待的goroutine



### 举例

1. goroutine1（简称g1） Lock
2. g2请求Lock，semacquire
3. g1 Unlock，semrelease，唤醒等待者g2
4. g2进入临界区操作，操作完 再Unlock



如果g2 原子incr key，还未调用 semacquire(&m.sema) 

g1先semrelease(&m.sema)，还能不能唤醒g2？



1. g1 Lock 成功
2. g2 Lock，进程暂停
3. g1 Unlock，semrelease(&m.sema)，此时g1还未休眠，无法唤醒，信号量自增
4. g2 进程继续，semacquire(&m.sema) 



g2执行semacquire(&m.sema)  后

> Semacquire waits until *s > 0 and then atomically decrements it.

还是能够正常执行



## V2：New Goroutine participates in lock competition.

信号量先进先出，V1版本如果有goroutine在等待了，新来的**正在**运行的goroutine也必须等待

唤醒老的在睡眠的goroutine，显然开销大于正在运行的goroutine直接获取锁



V2版本，允许已经有goroutine睡眠等待时，正在运行的goroutine先获取到锁

```
type Mutex struct {
   state int32
   sema  uint32
}

const (
   mutexLocked = 1 << iota // mutex is locked
   mutexWoken
   mutexWaiterShift = iota
)
```

### Lock

```
func (m *Mutex) Lock() {
   if atomic.CompareAndSwapInt32(&m.state, 0, mutexLocked) {
      return
   }

   awoke := false
   for {
      old := m.state
      var new int32
      if old&mutexLocked == 0 {
         new = old | mutexLocked
      } else {
      	 new = old + 1<<mutexWaiterShift
      }
      if awoke {
         new &^= mutexWoken
      }
      if atomic.CompareAndSwapInt32(&m.state, old, new) {
         if old&mutexLocked == 0 {
            break
         }
         runtime.Semacquire(&m.sema)
         awoke = true
      }
   }
}
```

- state cas从0到1成功，直接得到锁； 否则
  - for 循环，进行cas操作
    - 如果当前没有锁标记，加上锁标记
    - 有锁标记，增加一个等待计数
    - 另外，awoke为true时，清除mutexWoken比特位
  - cas进行更新，更新成功时
    - 旧值没有锁标记，表示本次cas拿到锁，直接退出；否则
    - Semacquire，进入睡眠等待，等待结束后，awoke置为true



### Unlock

```
func (m *Mutex) Unlock() {
   new := atomic.AddInt32(&m.state, -mutexLocked)

   old := new
   for {
      if old>>mutexWaiterShift == 0 || old&(mutexLocked|mutexWoken) != 0 {
         return
      }

      new = (old - 1<<mutexWaiterShift) | mutexWoken
      if atomic.CompareAndSwapInt32(&m.state, old, new) {
         runtime.Semrelease(&m.sema)
         return
      }
      old = m.state
   }
}
```

- 原子减-1
- for循环
  - 如果没有等待者， 或者 有lock/woken标记， 退出；否则
  - cas更新，新值为减去1个等待计数|woken标记
    - cas成功，Semrelease唤醒一个等待者
    - cas失败， 继续判断



### 举例

1. g1 Lock

2. g2, g3 请求Lock，进入休眠等待

3. g1 Unlock 和 g4 Lock， g5 Lock几乎同时发生

   1. 第一种情况 g1 Unlock 先完成了原子-1

      1. g4, g5看到 old&mutexLocked == 0，都增加locked比特，进行cas更新
      2. g4 g5其中一个更新成功， 获取到锁
      3. 另一个更新失败，再次进入for循环，
         1. 看到  old&mutexLocked != 0， 增加等待计数，cas更新成功后休眠等待
      4. 此时g1 进入for循环， 看到已经**有了mutex标记**，结束Unlock，**不需要再进行唤醒**

      

   2. 第二种情况 g1 Unlock 先完成了原子-1，并且cas新增woken标记成功

      1. g4其中一个获取锁成功，g5获取锁失败，再次进入for循环，**还未进行第二次cas更新**
      2. g4 很快完成操作，调用Unlock
      3. 此时，g4看到有**woken标记**，结束Unlock，不需要再进行唤醒
         1. 如果g4第二次cas更新成功，去掉woken标记，此时g4还会进行唤醒



### 问题

- 为什么要加woken标记
- CAS会不会有ABA问题？



#### 问题1：为什么要加woken标记

防止惊群，只需要唤醒一个

g1 Lock成功

g2, g3, g4 请求Lock， 进入睡眠等待

g1 Unlock，准备semrelease（还未调用）

此时 新来g5 可以Lock成功

g5 Unlock，准备semrelease（还未调用）

然后 g1, g5 同时semrelease，会唤醒多个在睡眠等待的goroutine

---

加了woken标记，就只会唤醒一个

woken的goroutine和新来的（也可能没有新来的）竞争， 失败则再次睡眠

被woken的会去掉woken标记进行cas， cas更新成功，都会去掉woken标记，未抢到锁时还会进入睡眠

---



#### 问题2：会不会有ABA问题

不会

- Unlock时， cas什么时候失败？

1. Unlock 原子-1后，没有lock标记，新来的g先于Unlock的cas加上了lock，cas失败
   1. 此时，在进入判断，会认为有lock标记，退出
2. 和上面类似，Unlock 原子-1后，没有lock标记，新来的g先于Unlock的cas加上了lock，cas失败。但是在Unlock再次判断之前，新来的g调用了Unlock。
   1. 此时有两个g在Unlock
   2. 两个都判断没有lock标记，假设还有一个等待者， 这是二者都会尝试加上woken标记，并减去1个等待者
   3. 只有一个添加woken成功，然后唤醒等待者，另一个cas失败，再次判断，由于woken存在或者等待者为0退出

- cas判断，会不会有N次semacquire，但是semrelease小于N? 
  - 不会，有多少次 semacquire， 就有多少次 semrelease



## V3：Give new goroutines some more chances.

增加自旋

有mutexLocked时，进行有限次数自旋，并尝试cas增加mutexWoken标记

整体逻辑和V2类似

### Lock

```
func (m *Mutex) Lock() {
    // Fast path: grab unlocked mutex.
    if atomic.CompareAndSwapInt32(&m.state, 0, mutexLocked) {
        return
    }

    awoke := false
    iter := 0
    for {
        old := m.state
        new := old | mutexLocked
        if old&mutexLocked != 0 {
            if runtime_canSpin(iter) {
                if !awoke && old&mutexWoken == 0 && old>>mutexWaiterShift != 0 &&
                    atomic.CompareAndSwapInt32(&m.state, old, old|mutexWoken) {
                    awoke = true
                }
                runtime_doSpin()
                iter++
                continue
            }
            new = old + 1<<mutexWaiterShift
        }
        if awoke {
            new &^= mutexWoken
        }
        if atomic.CompareAndSwapInt32(&m.state, old, new) {
            if old&mutexLocked == 0 {
                break
            }
            runtime_Semacquire(&m.sema)
            awoke = true
            iter = 0
        }
    }
}
```



### Unlock

逻辑同V2



## V4：Solve the old goroutine starvation problem.

V3版本新来的goroutine更容易竞争到锁，老的竞争者可能一直得不到锁

V4版本解决饥饿问题

```
const (
    mutexLocked = 1 << iota // mutex is locked
    mutexWoken
    mutexStarving // separate out a starvation token from the state field
    mutexWaiterShift = iota
    starvationThresholdNs = 1e6    
)
```

### Lock

```
func (m *Mutex) Lock() {
	// Fast path: grab unlocked mutex.
	if atomic.CompareAndSwapInt32(&m.state, 0, mutexLocked) {
		return
	}
	// Slow path (outlined so that the fast path can be inlined)
	m.lockSlow()
}

func (m *Mutex) lockSlow() {
	var waitStartTime int64
	starving := false
	awoke := false
	iter := 0
	old := m.state
	for {
		// Don't spin in starvation mode, ownership is handed off to waiters
		// so we won't be able to acquire the mutex anyway.
		if old&(mutexLocked|mutexStarving) == mutexLocked && runtime_canSpin(iter) {
			// Active spinning makes sense.
			// Try to set mutexWoken flag to inform Unlock
			// to not wake other blocked goroutines.
			if !awoke && old&mutexWoken == 0 && old>>mutexWaiterShift != 0 &&
				atomic.CompareAndSwapInt32(&m.state, old, old|mutexWoken) {
				awoke = true
			}
			runtime_doSpin()
			iter++
			old = m.state
			continue
		}
		new := old
		// Don't try to acquire starving mutex, new arriving goroutines must queue.
		if old&mutexStarving == 0 {
			new |= mutexLocked
		}
		if old&(mutexLocked|mutexStarving) != 0 {
			new += 1 << mutexWaiterShift
		}
		// The current goroutine switches mutex to starvation mode.
		// But if the mutex is currently unlocked, don't do the switch.
		// Unlock expects that starving mutex has waiters, which will not
		// be true in this case.
		if starving && old&mutexLocked != 0 {
			new |= mutexStarving
		}
		if awoke {
			// The goroutine has been woken from sleep,
			// so we need to reset the flag in either case.
			if new&mutexWoken == 0 {
				throw("sync: inconsistent mutex state")
			}
			new &^= mutexWoken
		}
		if atomic.CompareAndSwapInt32(&m.state, old, new) {
			if old&(mutexLocked|mutexStarving) == 0 {
				break // locked the mutex with CAS
			}
			// If we were already waiting before, queue at the front of the queue.
			queueLifo := waitStartTime != 0
			if waitStartTime == 0 {
				waitStartTime = runtime_nanotime()
			}
			runtime_SemacquireMutex(&m.sema, queueLifo, 1)
			starving = starving || runtime_nanotime()-waitStartTime > starvationThresholdNs
			old = m.state
			if old&mutexStarving != 0 {
				// If this goroutine was woken and mutex is in starvation mode,
				// ownership was handed off to us but mutex is in somewhat
				// inconsistent state: mutexLocked is not set and we are still
				// accounted as waiter. Fix that.
				if old&(mutexLocked|mutexWoken) != 0 || old>>mutexWaiterShift == 0 {
					throw("sync: inconsistent mutex state")
				}
				delta := int32(mutexLocked - 1<<mutexWaiterShift)
				if !starving || old>>mutexWaiterShift == 1 {
					// Exit starvation mode.
					// Critical to do it here and consider wait time.
					// Starvation mode is so inefficient, that two goroutines
					// can go lock-step infinitely once they switch mutex
					// to starvation mode.
					delta -= mutexStarving
				}
				atomic.AddInt32(&m.state, delta)
				break
			}
			awoke = true
			iter = 0
		} else {
			old = m.state
		}
	}

}
```



### Unlock

```
func (m *Mutex) Unlock() {
	// Fast path: drop lock bit.
	new := atomic.AddInt32(&m.state, -mutexLocked)
	if new != 0 {
		// Outlined slow path to allow inlining the fast path.
		// To hide unlockSlow during tracing we skip one extra frame when tracing GoUnblock.
		m.unlockSlow(new)
	}
}

func (m *Mutex) unlockSlow(new int32) {
	if new&mutexStarving == 0 {
		old := new
		for {
			// If there are no waiters or a goroutine has already
			// been woken or grabbed the lock, no need to wake anyone.
			// In starvation mode ownership is directly handed off from unlocking
			// goroutine to the next waiter. We are not part of this chain,
			// since we did not observe mutexStarving when we unlocked the mutex above.
			// So get off the way.
			if old>>mutexWaiterShift == 0 || old&(mutexLocked|mutexWoken|mutexStarving) != 0 {
				return
			}
			// Grab the right to wake someone.
			new = (old - 1<<mutexWaiterShift) | mutexWoken
			if atomic.CompareAndSwapInt32(&m.state, old, new) {
				runtime_Semrelease(&m.sema, false, 1)
				return
			}
			old = m.state
		}
	} else {
		// Starving mode: handoff mutex ownership to the next waiter.
		// Note: mutexLocked is not set, the waiter will set it after wakeup.
		// But mutex is still considered locked if mutexStarving is set,
		// so new coming goroutines won't acquire it.
		runtime_Semrelease(&m.sema, true, 1)
	}
}
```





## 总结

- V1版本：先进先出，新来的goroutine只能排队
- V2版本：新来的g和被唤醒的g竞争，可能直接拿到锁
  - g被唤醒不是直接拿到锁， 与新来的g竞争
  - 最多只有一个g被唤醒，被唤醒的g竞争失败，清空woken标记，再次睡眠
- V3版本：增加自旋逻辑，临界区较少时，自旋一会就能拿到锁，无需进入睡眠
  - 自旋时会尝试设置woken标记， 通知Unlock无需唤醒其他等待者
- V4版本：解决饥饿，新增mutexStarving标记位，引入普通/饥饿模式
  - 普通模式，同上述
  - 饥饿模式：
    - 新来的g，不竞争锁， 在队尾等待
    - 被唤醒的g，无需再次竞争，直接得到锁
    - Unlock，不设置woken标记， 直接唤醒



# 回到最开始的问题

- 为什么日志时间乱序？ mutex是先进先出的吗

线上go版本是1.16，是上面V4版本的实现，以测试代码为例

在每分钟0s Write时阻塞，此时后续的g，会先进行自旋，然后得不到锁，进入睡眠

到这里还都是按照先后顺序的，先进入，先打印，不应该出现时间乱序

第一个g，阻塞3s后，Write，然后Unlock

如果此时没有新来的g，交给sema第一个等待者，wake后没有竞争直接拿到锁，是不会进入饥饿模式的



但是如果有新来的g，Unlock时处于普通模式，新来的g与被唤醒的g竞争

1）如果新来的g（g1）竞争到，被唤醒的g0等待了3s，会cas进入饥饿模式。不过，这个cas不一定成功。g1 Unlock时刚好新来g2，仍然可以得到锁。g0会再次cas更新，直到CAS成功，一定能够加上mutexStarving标记

如果在CAS中直接获取到锁，会加上starving标记

如果在CAS中没有获取到锁，也会加上starving标记，并且加入到sema的队头，仍然符合先进先出的顺序

2）如果被唤醒的g竞争到，不会进入饥饿模式，新来的g加到信号量队尾

---



- 上述分析说明mutex整体上是先进先出的



- 普通模式，新来的g与被唤醒的g竞争，新来的g优势大
- 即使实际已经饥饿，等待时间过长，但是依次唤醒，依次都得到所有权，并不会转变到饥饿模式
  - 但是一但没有获取到锁，就会加上starving标记，后续进入饥饿模式
- 饥饿模式时，新来的g直接加入队尾

---



那么， 上述日志的时间乱序，另有原因

标准库log方法

```
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



## 解释现象 个人理解

时间是在Lock之前获取，为了获取代码行信息，会先Unlock，得到代码行数后，再Lock

日志中时间乱序是因为 Unlock后，runtime.Caller耗时有差异，再次Lock的顺序与获取time.Now顺序不一致

- 图片里第一个03s是因为新来的g先获取到了锁
- 后续进入饥饿模式，时间是按顺序的
- 饥饿模式退出后，有新来的g获取到了锁



---



- 去掉 Lshortfile|Llongfile， 再次运行，只会有零星的几个时间乱序，符合上述描述，还未实际进入饥饿模式时，新来的g也能够获取到锁
- 将标准库内 runtime.Caller 注释掉，Lock， 仍然是Unlock， Lock, Unlock的顺序， 有阻塞3s，日志时间是有序的



## 标准库log的优化

go1.21 [代码](https://github.com/golang/go/commit/c3b4c27fd31b51226274a0c038e9c10a65f11657) 修改了log实现

```
Performance:
	name           old time/op  new time/op  delta
	Concurrent-24  19.9µs ± 2%   8.3µs ± 1%  -58.37%  (p=0.000 n=10+10)
```



不再是每个Log对象用一个buf，使用了[]byte池

mutex只锁Write方法，runtime.Caller提前计算

用新版本测试，不会再出现时间乱序的问题





## futex

linux用futex实现pthread_mutex_t

[futex(2) — Linux manual page](https://man7.org/linux/man-pages/man2/futex.2.html)

与go的实现有何不同



后续分析




# 参考文章

[Deep Understanding of Golang Mutex](https://levelup.gitconnected.com/deep-understanding-of-golang-mutex-9964b02c56e9)