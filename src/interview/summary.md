## scheduler

Reference:
- [Go scheduler](https://nghiant3223.github.io/2025/04/15/go-scheduler.html)

## 内存分配

## GC
[GC图示](../assets/GC.pdf)
### GC触发的时机
- 阈值触发（内存分配时触发） mallogc
- 定时触发 sysmon
- 手动触发`runtime.GC`
- `debug.FreeOSMemory`

Reference:
- [go如何触发垃圾回收的](https://www.hitzhangjie.pro/blog/2022-11-20-go%E5%A6%82%E4%BD%95%E8%A7%A6%E5%8F%91%E5%9E%83%E5%9C%BE%E5%9B%9E%E6%94%B6/)

## Mutex
```go
type Mutex struct {
	state int32
	sema  uint32
}

mutexLocked = 1 << iota // mutex is locked
mutexWoken
mutexStarving
mutexWaiterShift = iota
```
`sema` 是用来管理 goroutine 阻塞与唤醒的机制，保证当锁不可用时，goroutine 可以安全地挂起，并在锁释放时被唤醒重新尝试获取锁。它是 Mutex 的慢路径核心，负责解决并发下的等待问题。**阻塞的队列也是通过信号量由runtime维护的**

`state`是一个int32的字段，低三位分别表示：是否加锁、是否有唤醒的goroutine、是否饥饿模式，其他位表示等待锁的goroutine数量。

### 加锁逻辑
**lock** 操作分位 *fast path* 和 *slow path*，*fast path* 是通过`cas`获取锁，失败后进入 *slow path*。

**slow path：**
互斥锁（Mutex）有两种运行模式：*正常模式*和*饥饿模式*。
在*正常模式*下，等待者会按 FIFO（先进先出）顺序排队，
但被唤醒的等待者并不会直接获得互斥锁，而是要和新到达的
Goroutine 一起竞争锁的所有权。新来的 Goroutine 有优势，
因为它们已经在 CPU 上运行，并且可能数量很多，
所以被唤醒的等待者很容易竞争失败。在这种情况下，
它会被重新放到等待队列的前端。如果一个等待者超过 1ms
都没能成功获取互斥锁，那么互斥锁会切换到饥饿模式。

在*饥饿模式*下，互斥锁的所有权会直接从解锁的 Goroutine
转交给等待队列头部的 Goroutine。新到达的 Goroutine
即使看到互斥锁是未加锁状态，也不会去尝试获取，
也不会自旋，而是直接排到等待队列的尾部。

如果某个等待者获得了互斥锁，并且它发现：它是**队列中最后一个等待者**，或者**它的等待时间少于 1ms**那么互斥锁会切换回*正常模式*。

正常模式的性能要好得多，因为 Goroutine 可以连续多次
获得互斥锁，即使此时还有其他等待者。
而饥饿模式的意义在于避免极端情况下的尾部延迟问题。
### 指令重排
为了提高cpu指令吞吐

#### 内存屏障：用来解决执行重排带来的部分不利影响。用来阻止屏障前后的指令共同参与重排序，保证屏障后的指令不会出现在屏障前执行，保证屏障前的指令不会在屏障后执行。相当于屏障之前和之后确立了happens-before关系，保证了屏障之前的操作对屏障之后的操作都是可见的。

#### happens-before：
Happens-before (HB) 是并发编程里的一个核心概念，用来描述 操作之间的可见性和顺序关系。

**定义**：
- 如果操作 A happens-before 操作 B，那么 B 一定 能看到 A 的结果，即 A 的所有副作用对 B 可见。
- 如果 A 和 B 之间没有 happens-before 关系，它们是 data race 的潜在源，因为没有同步保证。

**特点**：
- 传递性：如果 A → B 且 B → C，那么 A → C。
- 同步操作建立 HB：原子操作、锁、通道通信等可以建立 HB。
- 无 HB：可能出现重排序或不可见状态。


**golang中的HB体现**：Go 官方在 Memory Model
中明确了 HB 的规则，主要通过 同步原语 建立 HB 关系
- Mutex / RWMutex
- Channel 发送接收
- WaitGroup / Cond
- 原子操作 (atomic)
- sadl

#### cas
CAS 是一种 原子操作，用于多线程/多 goroutine 下安全地修改共享变量。底层是CPU 提供的原子指令。

操作步骤：
1. 读取内存地址 addr 的当前值 v
2. 比较 v 和期望值 old：
3. 如果相等 → 交换 v 为 new，返回 true
4. 如果不等 → 不修改，返回 false

通过cpu的`lock`指令保证原子性。通过
**用途：**
- 实现无锁算法
- 实现互斥锁、计数器等并发数据结构

**CAS 的局限性**：

- ABA 问题：如果值从 A → B → A，CAS 会误认为没变化，导致错误。解决方案：加版本号（如 atomic.Value 或指针+计数器）

- 自旋开销：
    - CAS 失败频繁自旋 → CPU 占用高

- 只能操作单个内存位置，复杂结构需要多 CAS 或锁

**ABA问题**：实际上中间已经发生了变化（A → B → A），但是另一个线程未发现。
1. 一个线程读取共享变量 V 的值，假设是 A。
2. 在该线程还未执行 CAS 前，其他线程把 V 从 A 改成 B，又改回 A。
3. 第一个线程执行 CAS 时，看到的仍然是 A，认为没有变化，于是 CAS 成功。

## Slice
在老的版本中，

新版本中
1. `newLen`>`2*oldCap`，返回`newLen`
2. `oldCap`>threshold=256，返回 `2 * oldCap`
3. 在当前`cap`基础上循环累加`(cap + 3*256)>>2`，直到满足`newLen`
```go
func growslice(oldPtr unsafe.Pointer, newLen, oldCap, num int, et *_type) slice {
    ...
    newcap := nextslicecap(newLen, oldCap)
    ...
}

func nextslicecap(newLen, oldCap int) int {
	newcap := oldCap
	doublecap := newcap + newcap
	if newLen > doublecap {
		return newLen
	}

	const threshold = 256
	if oldCap < threshold {
		return doublecap
	}
	for {
		// Transition from growing 2x for small slices
		// to growing 1.25x for large slices. This formula
		// gives a smooth-ish transition between the two.
		newcap += (newcap + 3*threshold) >> 2

		// We need to check `newcap >= newLen` and whether `newcap` overflowed.
		// newLen is guaranteed to be larger than zero, hence
		// when newcap overflows then `uint(newcap) > uint(newLen)`.
		// This allows to check for both with the same comparison.
		if uint(newcap) >= uint(newLen) {
			break
		}
	}

	// Set newcap to the requested cap when
	// the newcap calculation overflowed.
	if newcap <= 0 {
		return newLen
	}
	return newcap
}
```
结论：
- `newLen`大于两倍的`oldCap`，扩容到`newLen`
- `oldCap`小于256，扩容到`2*oldCap`
- 否则在当前`cap`基础上循环累加`(cap + 3*256)>>2`，直到满足`newLen`
## Map
### go1.24之前
```go
// A header for a Go map.
type hmap struct {
	count     int // # size of map.  Must be first (used by len() builtin)
	flags     uint8
	B         uint8  // log_2 of # of buckets (can hold up to loadFactor * 2^B items)
	noverflow uint16 // approximate number of overflow buckets; see incrnoverflow for details
	hash0     uint32 // hash seed

	buckets    unsafe.Pointer // array of 2^B Buckets. may be nil if count==0.
	oldbuckets unsafe.Pointer // previous bucket array of half the size, non-nil only when growing
	nevacuate  uintptr        // progress counter for evacuation (buckets less than this have been evacuated)
	clearSeq   uint64

	extra *mapextra // optional fields
}

// A bucket for a Go map.
type bmap struct {
	// tophash generally contains the top byte of the hash value
	// for each key in this bucket. If tophash[0] < minTopHash,
	// tophash[0] is a bucket evacuation state instead.
	tophash [abi.OldMapBucketCount]uint8
	// Followed by bucketCnt keys and then bucketCnt elems.
	// NOTE: packing all the keys together and then all the elems together makes the
	// code a bit more complicated than alternating key/elem/key/elem/... but it allows
	// us to eliminate padding which would be needed for, e.g., map[int64]int8.
	// Followed by an overflow pointer.
}

// 因为哈希表中可能存储不同类型的键值对，go bmap的原代码不可能知道用户使用什么类型，所以只能编译期重建。
// bmap编译期被还原成如下：
type bmap struct {
    topbits  [8]uint8
    keys     [8]keytype
    values   [8]valuetype
    pad      uintptr
    overflow uintptr
}
```
#### 赋值
```go
func mapassign(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
    if h == nil {
        panic(plainError("assignment to entry in nil map"))
    }
    ...
    if h.flags&hashWriting != 0 { // 并发写冲突检查，说明 map 不是并发安全的
		fatal("concurrent map writes")
	}
    hash := t.Hasher(noescape(unsafe.Pointer(&key)), uintptr(h.hash0))

    // Set hashWriting after calling t.hasher, since t.hasher may panic,
	// in which case we have not actually done a write. 
	h.flags ^= hashWriting // 写操作标记写入状态，说明 map 是有状态的

    if h.buckets == nil {
		h.buckets = newobject(t.Bucket) // newarray(t.bucket, 1)
	}

again:
	bucket := hash & bucketMask(h.B) // 取哈希值低 B 位，计算 key 应该放到哪个桶中
	if h.growing() { 
		growWork(t, h, bucket) // 渐进式搬迁旧桶。搬迁当前桶对应旧桶，并额外再搬一个桶。它们链接的所有溢出桶一起迁移。
	}
	b := (*bmap)(add(h.buckets, bucket*uintptr(t.BucketSize)))
	top := tophash(hash) // 取哈希值的高 8 位

	var insertb *bmap
	var inserti uintptr
	var insertk unsafe.Pointer
bucketloop:
	for {
		for i := uintptr(0); i < abi.MapBucketCount; i++ { // 遍历8个槽位
			// 槽位的 tophash 和 键值的 tophash 不一致,可能是个空槽位.
			if b.tophash[i] != top { 
				if isEmpty(b.tophash[i]) && inserti == nil {
                    // 记录空槽的位置。注意：这里说明在 tophash 和某个已存在的key 的 tophash 冲突时，会优先放到同一个桶的其他槽里。即开放地址法解决哈希冲突。
					inserti = &b.tophash[i]
					insertk = add(unsafe.Pointer(b), dataOffset+i*uintptr(t.KeySize))
					elem = add(unsafe.Pointer(b), dataOffset+abi.MapBucketCount*uintptr(t.KeySize)+i*uintptr(t.ValueSize))
				}
				if b.tophash[i] == emptyRest { 
                    // 从当前槽到桶尾都是空的，可以提前 break。跳出bucketloop中所有循环，去执行
                    // if !h.growing() && (overLoadFactor(h.count+1, h.B) || tooManyOverflowBuckets(h.noverflow, h.B))
					break bucketloop
				}
				continue 因为该槽位的 tophash 不等于 key 的 top，且没有命中提前结束的条件，所以去看下一个槽位。
			}
            // 已经找到了对应的槽
			k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.KeySize))
			if t.IndirectKey() {
				k = *((*unsafe.Pointer)(k))
			}
            // 该位置的 key 和 我们的key不相等，继续找
			if !t.Key.Equal(key, k) {
				continue
			}
			// already have a mapping for key. Update it.
            // 找到了
			if t.NeedKeyUpdate() {
				typedmemmove(t.Key, k, key)
			}
			elem = add(unsafe.Pointer(b), dataOffset+abi.MapBucketCount*uintptr(t.KeySize)+i*uintptr(t.ValueSize))
			goto done
		}
        // 普通桶里没有找到，看下溢出桶
		ovf := b.overflow(t)
		if ovf == nil {
			break
		}
		b = ovf
	}

    // Did not find mapping for key. Allocate new cell & add entry.

	// If we hit the max load factor or we have too many overflow buckets,
	// and we're not already in the middle of growing, start growing.
    // 桶扩容的条件：命中了最大负载因子或溢出桶太多
    // - 负载因子：6.5
    // - 溢出桶太多
    //    - 如果 B 特别大，溢出桶阈值也会非常大，这里把 B 限制到最大 15
    //    - 如果溢出桶数 >= 主桶数，就算“太多”
	if !h.growing() && (overLoadFactor(h.count+1, h.B) || tooManyOverflowBuckets(h.noverflow, h.B)) {
		hashGrow(t, h)
		goto again // Growing the table invalidates everything, so try again
	}

	if inserti == nil {
		// The current bucket and all the overflow buckets connected to it are full, allocate a new one.
        // 当前桶和溢出桶都满了，分配新的桶。
		newb := h.newoverflow(t, b)
		inserti = &newb.tophash[0]
		insertk = add(unsafe.Pointer(newb), dataOffset)
		elem = add(insertk, abi.MapBucketCount*uintptr(t.KeySize))
	}

	// store new key/elem at insert position
	if t.IndirectKey() {
		kmem := newobject(t.Key)
		*(*unsafe.Pointer)(insertk) = kmem
		insertk = kmem
	}
	if t.IndirectElem() {
		vmem := newobject(t.Elem)
		*(*unsafe.Pointer)(elem) = vmem
	}
	typedmemmove(t.Key, insertk, key)
	*inserti = top
	h.count++

done:
	if h.flags&hashWriting == 0 {
		fatal("concurrent map writes")
	}
	h.flags &^= hashWriting
	if t.IndirectElem() {
		elem = *((*unsafe.Pointer)(elem))
	}
	return elem
}
```
总结：
1. 赋值会检查是否并发写，并发就报错。
2. 对 key 进行 hash，用后 *B* 位找到对应的桶，并判断当前如果处在扩容期间就进行当前桶的数据迁移，并额外再搬一个桶。它们链接的所有溢出桶一起迁移。
3. 用 hash 值的*高8位*找到对应的槽并判断key是否一致，最后找到空槽或者 key 相等的。PS：对应uint64、uint32等类型不会用 hash 值的高8位，而是直接用key。
4. 判断是否需要扩容，需要就扩容。桶扩容的条件：命中了最大负载因子或溢出桶太多
- 负载因子：6.5
- 溢出桶太多
   - 如果 B 特别大，溢出桶阈值也会非常大，这里把 B 限制到最大 15。
    - 加倍扩容。一个桶的元素会放到两个新桶里。
   - 如果溢出桶数 >= 主桶数，就算“太多”
    - 等量扩容。把存储稀疏的kv，更集中的存放
5. 如果找到应该存的桶和对应的溢出桶都满了，就创建新的桶
6. 插入元素。
7. h.flags &^= hashWriting，位清除。

#### 数据迁移
一次迁移一个桶，并把对应的溢出桶也迁移了
```go
func growWork(t *maptype, h *hmap, bucket uintptr) {
	// make sure we evacuate the oldbucket corresponding
	// to the bucket we're about to use
	evacuate(t, h, bucket&h.oldbucketmask())

	// evacuate one more oldbucket to make progress on growing
	if h.growing() {
		evacuate(t, h, h.nevacuate)
	}
}

func evacuate(t *maptype, h *hmap, oldbucket uintptr) {
	b := (*bmap)(add(h.oldbuckets, oldbucket*uintptr(t.BucketSize)))
	newbit := h.noldbuckets()
	if !evacuated(b) {
		// TODO: reuse overflow buckets instead of using new ones, if there
		// is no iterator using the old buckets.  (If !oldIterator.)

		// xy contains the x and y (low and high) evacuation destinations.
		var xy [2]evacDst
		x := &xy[0]
		x.b = (*bmap)(add(h.buckets, oldbucket*uintptr(t.BucketSize)))
		x.k = add(unsafe.Pointer(x.b), dataOffset)
		x.e = add(x.k, abi.MapBucketCount*uintptr(t.KeySize))

		if !h.sameSizeGrow() {
			// Only calculate y pointers if we're growing bigger.
			// Otherwise GC can see bad pointers.
			y := &xy[1]
			y.b = (*bmap)(add(h.buckets, (oldbucket+newbit)*uintptr(t.BucketSize)))
			y.k = add(unsafe.Pointer(y.b), dataOffset)
			y.e = add(y.k, abi.MapBucketCount*uintptr(t.KeySize))
		}

        // 说明把溢出桶也一起迁移了
		for ; b != nil; b = b.overflow(t) {
			k := add(unsafe.Pointer(b), dataOffset)
			e := add(k, abi.MapBucketCount*uintptr(t.KeySize))
			for i := 0; i < abi.MapBucketCount; i, k, e = i+1, add(k, uintptr(t.KeySize)), add(e, uintptr(t.ValueSize)) {
				top := b.tophash[i]
				if isEmpty(top) {
					b.tophash[i] = evacuatedEmpty
					continue
				}
				if top < minTopHash {
					throw("bad map state")
				}
				k2 := k
				if t.IndirectKey() {
					k2 = *((*unsafe.Pointer)(k2))
				}
				var useY uint8
				if !h.sameSizeGrow() {
					// Compute hash to make our evacuation decision (whether we need
					// to send this key/elem to bucket x or bucket y).
					hash := t.Hasher(k2, uintptr(h.hash0))
					if h.flags&iterator != 0 && !t.ReflexiveKey() && !t.Key.Equal(k2, k2) {
						// If key != key (NaNs), then the hash could be (and probably
						// will be) entirely different from the old hash. Moreover,
						// it isn't reproducible. Reproducibility is required in the
						// presence of iterators, as our evacuation decision must
						// match whatever decision the iterator made.
						// Fortunately, we have the freedom to send these keys either
						// way. Also, tophash is meaningless for these kinds of keys.
						// We let the low bit of tophash drive the evacuation decision.
						// We recompute a new random tophash for the next level so
						// these keys will get evenly distributed across all buckets
						// after multiple grows.
						useY = top & 1
						top = tophash(hash)
					} else {
						if hash&newbit != 0 {
							useY = 1
						}
					}
				}

				if evacuatedX+1 != evacuatedY || evacuatedX^1 != evacuatedY {
					throw("bad evacuatedN")
				}

				b.tophash[i] = evacuatedX + useY // evacuatedX + 1 == evacuatedY
				dst := &xy[useY]                 // evacuation destination

				if dst.i == abi.MapBucketCount {
					dst.b = h.newoverflow(t, dst.b)
					dst.i = 0
					dst.k = add(unsafe.Pointer(dst.b), dataOffset)
					dst.e = add(dst.k, abi.MapBucketCount*uintptr(t.KeySize))
				}
				dst.b.tophash[dst.i&(abi.MapBucketCount-1)] = top // mask dst.i as an optimization, to avoid a bounds check
				if t.IndirectKey() {
					*(*unsafe.Pointer)(dst.k) = k2 // copy pointer
				} else {
					typedmemmove(t.Key, dst.k, k) // copy elem
				}
				if t.IndirectElem() {
					*(*unsafe.Pointer)(dst.e) = *(*unsafe.Pointer)(e)
				} else {
					typedmemmove(t.Elem, dst.e, e)
				}
				dst.i++
				// These updates might push these pointers past the end of the
				// key or elem arrays.  That's ok, as we have the overflow pointer
				// at the end of the bucket to protect against pointing past the
				// end of the bucket.
				dst.k = add(dst.k, uintptr(t.KeySize))
				dst.e = add(dst.e, uintptr(t.ValueSize))
			}
		}
		// Unlink the overflow buckets & clear key/elem to help GC.
		if h.flags&oldIterator == 0 && t.Bucket.Pointers() {
			b := add(h.oldbuckets, oldbucket*uintptr(t.BucketSize))
			// Preserve b.tophash because the evacuation
			// state is maintained there.
			ptr := add(b, dataOffset)
			n := uintptr(t.BucketSize) - dataOffset
			memclrHasPointers(ptr, n)
		}
	}

	if oldbucket == h.nevacuate {
		advanceEvacuationMark(h, t, newbit)
	}
}
```
**map中解决hash冲突（tophash冲突，不是key冲突）的方法**：优先用开放地址法，如果应该放的桶里没有空余的槽，才会链地址法。

**新旧数据迁移在什么时候**：渐进式迁移，在访问、赋值、删除时。
### go1.24之后

**Referrence:**
<!-- 1.24之前 -->
- [浅谈Golang 1.21.0 map源码](https://zhuanlan.zhihu.com/p/653518993)
<!-- 1.24之后 -->
- [Faster Go maps with Swiss Tables](https://go.dev/blog/swisstable)
- [How Go 1.24's Swiss Tables saved us hundreds of gigabytes](https://www.datadoghq.com/blog/engineering/go-swiss-tables/)
- [Go 1.24用户报告：Datadog如何借助 Swiss Tables版map节省数百 GB 内存？](https://tonybai.com/2025/07/22/go-swiss-table-map-user-report/)
- [Map internals in Go 1.24](https://themsaid.com/map-internals-go-1-24?utm_source=chatgpt.com)
## Channel

## Context

## Sync
### Sync.Map
### Sync.Once
### 