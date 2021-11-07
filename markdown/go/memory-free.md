go 使用 madvise 系统调用来释放物理内存给操作系统，该方法主要有两种归还类型可选：

- `MADV_DONTNEED`：立即归还物理内存给操作系统，如果下次访问到该范围的内存，则会触发 page fault 异常，需要重新分配物理页，使用该类型可以减少程序的RSS占用
- `MADV_FREE`：告诉操作系统这块内存已经不需要使用了，可以回收了，如果内存紧张，操作系统就会将其回收。这实际是一个lazily的释放过程。如果再次访问这块内存的时候，操作系统还没有将其回收，是不会触发 page fault 的。使用该类型，可能程序的RSS不会减少。

当前实现，首先尝试使用 `MADV_FREE`，如果失败了，再尝试使用 `MADV_DONTNEED`。`MADV_FREE`需要linux内核4.5及以上：

```go
var adviseUnused = uint32(_MADV_FREE)
func sysUnused(v unsafe.Pointer, n uintptr) {
    ......
  
    var advise uint32
    // 如果设置了`GODEBUG=madvdontneed=1`，强制使用MADV_DONTNEED
    if debug.madvdontneed != 0 {
        advise = _MADV_DONTNEED
    } else {
        advise = atomic.Load(&adviseUnused)
    }
    // 首先尝试使用`MADV_FREE`
    if errno := madvise(v, n, int32(advise)); advise == _MADV_FREE && errno != 0 {
        // MADV_FREE was added in Linux 4.5. Fall back to MADV_DONTNEED if it is
        // not supported.
        atomic.Store(&adviseUnused, _MADV_DONTNEED)
        madvise(v, n, _MADV_DONTNEED)
    }
}
```



`MADV_FREE`的性能会好一点，但是会让程序的`RSS`占用不会减少，可以通过`GODEBUG=madvdontneed=1`强制使用`MADV_DONTNEED`。

go中堆内存主要使用`mmap`来申请的。首先，使用`mmap`来`reserve`一段内存：

```go
func sysReserve(v unsafe.Pointer, n uintptr) unsafe.Pointer {
	p, err := mmap(v, n, _PROT_NONE, _MAP_ANON|_MAP_PRIVATE, -1, 0)
	if err != 0 {
		return nil
	}
	return p
}
```



可以看到，使用匿名映射一块内存，并且指定了 PROT_NONE ，即这块内存是不可以被访问的，因此就不会分配物理内存了。当需要真正分配内存的时候，在这块保留的内存中分配：

```go
func sysMap(v unsafe.Pointer, n uintptr, sysStat *uint64) {
	mSysStatInc(sysStat, n) // 内存统计
	// 修改前面保留的内存的指定区域为可读写
	p, err := mmap(v, n, _PROT_READ|_PROT_WRITE, _MAP_ANON|_MAP_FIXED|_MAP_PRIVATE, -1, 0)
	if err == _ENOMEM {
		throw("runtime: out of memory")
	}
	if p != v || err != 0 {
		throw("runtime: cannot map pages in arena address space")
	}
}
```



当我们查看进程的内存占用时，主要关注的是`RSS`，而`reserve`的内存是不能被访问的，不会分配物理内存页，并不会影响`RSS`的值。