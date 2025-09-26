# `sync.Mutex` source code

## Mutex 定义
源码位置：`/usr/local/go/src/internal/sync/mutex.go`
```go
type Mutex struct {
	state int32
	sema  uint32
}

const (
	mutexLocked = 1 << iota // mutex is locked
	mutexWoken
	mutexStarving
	mutexWaiterShift = iota

	// Mutex fairness.
	//
	// Mutex can be in 2 modes of operations: normal and starvation.
	// In normal mode waiters are queued in FIFO order, but a woken up waiter
	// does not own the mutex and competes with new arriving goroutines over
	// the ownership. New arriving goroutines have an advantage -- they are
	// already running on CPU and there can be lots of them, so a woken up
	// waiter has good chances of losing. In such case it is queued at front
	// of the wait queue. If a waiter fails to acquire the mutex for more than 1ms,
	// it switches mutex to the starvation mode.
	//
	// In starvation mode ownership of the mutex is directly handed off from
	// the unlocking goroutine to the waiter at the front of the queue.
	// New arriving goroutines don't try to acquire the mutex even if it appears
	// to be unlocked, and don't try to spin. Instead they queue themselves at
	// the tail of the wait queue.
	//
	// If a waiter receives ownership of the mutex and sees that either
	// (1) it is the last waiter in the queue, or (2) it waited for less than 1 ms,
	// it switches mutex back to normal operation mode.
	//
	// Normal mode has considerably better performance as a goroutine can acquire
	// a mutex several times in a row even if there are blocked waiters.
	// Starvation mode is important to prevent pathological cases of tail latency.

    // 互斥锁（Mutex）有两种运行模式：正常模式和饥饿模式。
    // 在正常模式下，等待者会按 FIFO（先进先出）顺序排队，
    // 但被唤醒的等待者并不会直接获得互斥锁，而是要和新到达的
    // Goroutine 一起竞争锁的所有权。新来的 Goroutine 有优势，
    // 因为它们已经在 CPU 上运行，并且可能数量很多，
    // 所以被唤醒的等待者很容易竞争失败。在这种情况下，
    // 它会被重新放到等待队列的前端。如果一个等待者超过 1ms
    // 都没能成功获取互斥锁，那么互斥锁会切换到饥饿模式。
    //
    // 在饥饿模式下，互斥锁的所有权会直接从解锁的 Goroutine
    // 转交给等待队列头部的 Goroutine。新到达的 Goroutine
    // 即使看到互斥锁是未加锁状态，也不会去尝试获取，
    // 也不会自旋，而是直接排到等待队列的尾部。
    //
    // 如果某个等待者获得了互斥锁，并且它发现：
    // (1) 它是队列中最后一个等待者，或者
    // (2) 它的等待时间少于 1ms，
    // 那么互斥锁会切换回正常模式。
    //
    // 正常模式的性能要好得多，因为 Goroutine 可以连续多次
    // 获得互斥锁，即使此时还有其他等待者。
    // 而饥饿模式的意义在于避免极端情况下的尾部延迟问题。
	starvationThresholdNs = 1e6
)
```

### Go Mutex 状态位示意图：
`state` 字段 32 位示意（低位在右，高位在左）：
```
Bit index:   31 ... 6  5  4   3  2  1  0
            [      Waiters    ]  S  W  L
```
**说明：**
| 位 | 名称       | 说明 |
|----|------------|------|
| 0  | L          | `mutexLocked`：锁是否被占用 |
| 1  | W          | `mutexWoken`：是否有 goroutine 被唤醒 |
| 2  | S          | `mutexStarving`：是否进入饥饿模式 |
| 3~31 | Waiters  | 等待锁的 goroutine 数量（从 bit3 开始到高位） |

**操作示例：**
- 检查锁是否被占用: `old & mutexLocked != 0`
- 检查是否唤醒过:     `old & mutexWoken != 0`
- 检查是否饥饿:       `old & mutexStarving != 0`
- 获取等待者数量:     `old >> mutexWaiterShift`

## Lock 逻辑
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

func (m *Mutex) lockSlow() {
	var waitStartTime int64
	starving := false
	awoke := false
	iter := 0
	old := m.state
	for {
		// Don't spin in starvation mode, ownership is handed off to waiters
		// so we won't be able to acquire the mutex anyway.
		// 在饥饿模式下不会自旋，因为锁的所有权会直接交给等待者，
		// 所以当前 goroutine 无法通过自旋获取锁。

        // 检查当前互斥锁是否处于“已加锁”状态，但没有进入饥饿模式
		if old&(mutexLocked|mutexStarving) == mutexLocked && runtime_canSpin(iter) {
			// Active spinning makes sense.
			// Try to set mutexWoken flag to inform Unlock
			// to not wake other blocked goroutines.
			// 在正常模式下，自旋是有意义的。
			// 尝试设置 mutexWoken 标志，通知 Unlock 不要唤醒其他阻塞的 goroutine。

            // old>>mutexWaiterShift != 0 取出 waiters 的数量
            // 当前 goroutine 不是刚被唤醒的，并且等待队列中有人。
            // 尝试通过 CAS 设置 mutexWoken，标记已经有一个 goroutine 正在唤醒。 
            // 这样 Unlock() 就不会唤醒额外的 goroutine，避免竞争。 
            // 同时，如果锁即将释放，当前 goroutine 可能会自旋尝试获取锁。
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
		// 不要尝试获取饥饿模式下的互斥锁，新来的 goroutine 必须进入等待队列。

        // 如果 Mutex 不是饥饿模式，当前 goroutine 可以直接把锁标记为 locked，进入临界区。
		if old&mutexStarving == 0 {
			new |= mutexLocked
		}

        // Mutex 当前已被锁定或处于饥饿模式。将 waiter 数量+1
		if old&(mutexLocked|mutexStarving) != 0 { 
			new += 1 << mutexWaiterShift
		}
		// The current goroutine switches mutex to starvation mode.
		// But if the mutex is currently unlocked, don't do the switch.
		// Unlock expects that starving mutex has waiters, which will not
		// be true in this case.
		// 当前 goroutine 将互斥锁切换到饥饿模式。
		// 但如果锁此时未被持有，不要切换。
		// 因为 Unlock 期望饥饿模式下锁一定有等待者，而这里可能并没有。
		if starving && old&mutexLocked != 0 {
			new |= mutexStarving
		}
		if awoke {
			// The goroutine has been woken from sleep,
			// so we need to reset the flag in either case.
			// 当前 goroutine 是被唤醒的，
			// 因此无论如何都需要清除 mutexWoken 标志。
			if new&mutexWoken == 0 {
				throw("sync: inconsistent mutex state")
			}
			new &^= mutexWoken
		}
		if atomic.CompareAndSwapInt32(&m.state, old, new) {
			if old&(mutexLocked|mutexStarving) == 0 {
				break // locked the mutex with CAS
				// 通过 CAS 成功获取锁，退出循环。
			}
			// If we were already waiting before, queue at the front of the queue.
			// 如果之前已经在等待，则放到等待队列的前面。
			queueLifo := waitStartTime != 0
			if waitStartTime == 0 {
				waitStartTime = runtime_nanotime()
			}
			runtime_SemacquireMutex(&m.sema, queueLifo, 2)
			starving = starving || runtime_nanotime()-waitStartTime > starvationThresholdNs
			old = m.state
			if old&mutexStarving != 0 {
				// If this goroutine was woken and mutex is in starvation mode,
				// ownership was handed off to us but mutex is in somewhat
				// inconsistent state: mutexLocked is not set and we are still
				// accounted as waiter. Fix that.
				// 如果当前 goroutine 被唤醒，并且互斥锁处于饥饿模式，
				// 那么锁的所有权已转交给我们，但状态可能不一致：
				// mutexLocked 没有设置，但我们仍被计为等待者。需要修复这个问题。
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
					// 退出饥饿模式。
					// 必须在这里处理并考虑等待时间。
					// 饥饿模式效率极低，两个 goroutine 可能会无限交替锁定，
					// 一旦它们切换到饥饿模式。
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

	if race.Enabled {
		race.Acquire(unsafe.Pointer(m))
	}
}
```

## Reference
- [Locks实现:背后不为人知的故事](https://www.hitzhangjie.pro/blog/2021-04-17-locks%E5%AE%9E%E7%8E%B0%E9%82%A3%E4%BA%9B%E4%B8%8D%E4%B8%BA%E4%BA%BA%E7%9F%A5%E7%9A%84%E6%95%85%E4%BA%8B/#syncmutex)