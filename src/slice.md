# `slice` source code 

## make slice
```golang

```
## append
源码位置：`/usr/local/go/src/runtime/slice.go`
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