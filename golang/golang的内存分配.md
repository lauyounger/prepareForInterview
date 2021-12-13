### golang的内存分配

go的内存分配采用了类似tcmalloc的结构，使用一小块一小块的连续内存页，进行分配某个范围大小的内存需求，比如某个连续8KB的内存专门用于分配17-24KB，以此减少内存碎片。线程拥有一定的cache，可用于无锁分配。同时GC后对于回收页，也不会立马归还操作系统，而是会延迟归还，用于满足未来的内存需求。



#### 内存空间结构

首先我们看一下golang的内存空间图

![golang内存空间](/Users/liuyangyang/刘阳阳/2021-interview/images/golang内存空间.jpeg)

go在1.10之前堆地址空间是线性连续扩展的，比如在1.10(linux amd64)中，最大可以扩展到512GB，因为go在gc的时候会根据拿到的指针地址来判断是否位于go的heap，以及找到对应的span结构，其判断地址需要gc heap的地址空间连续，但是连续有个问题，cgo中的代码尤其是32位系统上，可能会占用未来会用于 go heap的内存，这样在扩展go heap的时，mmap出现不连续的地址空间，会throw panic。

在1.11版本中，改用了稀疏索引的方式来管理整体的内存，允许内存空间扩展式不连续，空间可以达到256TB。如图，在全局的mheap的结构中有个arenas二阶的数组，在linux amd64上，一阶只有一个slot，二阶有4M个slot，每个slot指向一个heapArena结构，每个heapArena结构管理64M的内存，所以在目前64位机器上，管理的内存空间可以到256T。

#### 内存管理组件

下面我们从内存管理里面的几个组件结构入手，学习理解下内存分配的过程，首先看一下张图

![golang内存管理组件结构图](/Users/liuyangyang/刘阳阳/2021-interview/images/go 内存管理组件结构图.png)

##### mspan

```go
type mspan struct {
	next *mspan     // next span in list, or nil if none
	prev *mspan     // previous span in list, or nil if none
	list *mSpanList // For debugging. TODO: Remove.

	startAddr uintptr // address of first byte of span aka s.base()
	npages    uintptr // number of pages in span

	manualFreeList gclinkptr // list of free objects in mSpanManual spans

	// freeindex is the slot index between 0 and nelems at which to begin scanning
	// for the next free object in this span.
	// Each allocation scans allocBits starting at freeindex until it encounters a 0
	// indicating a free object. freeindex is then adjusted so that subsequent scans begin
	// just past the newly discovered free object.
	//
	// If freeindex == nelem, this span has no free objects.
	//
	// allocBits is a bitmap of objects in this span.
	// If n >= freeindex and allocBits[n/8] & (1<<(n%8)) is 0
	// then object n is free;
	// otherwise, object n is allocated. Bits starting at nelem are
	// undefined and should never be referenced.
	//
	// Object n starts at address n*elemsize + (start << pageShift).
	freeindex uintptr
	// TODO: Look up nelems from sizeclass and remove this field if it
	// helps performance.
	nelems uintptr // number of object in the span.

	// Cache of the allocBits at freeindex. allocCache is shifted
	// such that the lowest bit corresponds to the bit freeindex.
	// allocCache holds the complement of allocBits, thus allowing
	// ctz (count trailing zero) to use it directly.
	// allocCache may contain bits beyond s.nelems; the caller must ignore
	// these.
	allocCache uint64

	// allocBits and gcmarkBits hold pointers to a span's mark and
	// allocation bits. The pointers are 8 byte aligned.
	// There are three arenas where this data is held.
	// free: Dirty arenas that are no longer accessed
	//       and can be reused.
	// next: Holds information to be used in the next GC cycle.
	// current: Information being used during this GC cycle.
	// previous: Information being used during the last GC cycle.
	// A new GC cycle starts with the call to finishsweep_m.
	// finishsweep_m moves the previous arena to the free arena,
	// the current arena to the previous arena, and
	// the next arena to the current arena.
	// The next arena is populated as the spans request
	// memory to hold gcmarkBits for the next GC cycle as well
	// as allocBits for newly allocated spans.
	//
	// The pointer arithmetic is done "by hand" instead of using
	// arrays to avoid bounds checks along critical performance
	// paths.
	// The sweep will free the old allocBits and set allocBits to the
	// gcmarkBits. The gcmarkBits are replaced with a fresh zeroed
	// out memory.
	allocBits  *gcBits
	gcmarkBits *gcBits

	// sweep generation:
	// if sweepgen == h->sweepgen - 2, the span needs sweeping
	// if sweepgen == h->sweepgen - 1, the span is currently being swept
	// if sweepgen == h->sweepgen, the span is swept and ready to use
	// if sweepgen == h->sweepgen + 1, the span was cached before sweep began and is still cached, and needs sweeping
	// if sweepgen == h->sweepgen + 3, the span was swept and then cached and is still cached
	// h->sweepgen is incremented by 2 after every GC

	sweepgen    uint32
	divMul      uint16        // for divide by elemsize - divMagic.mul
	baseMask    uint16        // if non-0, elemsize is a power of 2, & this will get object allocation base
	allocCount  uint16        // number of allocated objects
	spanclass   spanClass     // size class and noscan (uint8)
	state       mSpanStateBox // mSpanInUse etc; accessed atomically (get/set methods)
	needzero    uint8         // needs to be zeroed before allocation
	divShift    uint8         // for divide by elemsize - divMagic.shift
	divShift2   uint8         // for divide by elemsize - divMagic.shift2
	elemsize    uintptr       // computed from sizeclass or from npages
	limit       uintptr       // end of data in span
	speciallock mutex         // guards specials list
	specials    *special      // linked list of special records sorted by offset.
}
```

mspan是go内存管理的基本单元，结构里面的next，prev分别代表指向下一个和前一个mspan，双向链表存储。

每个mspan都管理npages个大小为8KB的页，这里的页和操作系统的内存页不同。它们是操作系统内存页的整数倍，该结构体会用下面的字段管理内存页的分配与回收

```go
type mspan struct {
	startAddr uintptr // 起始地址
	npages    uintptr // 页数
	freeindex uintptr

	allocBits  *gcBits
	gcmarkBits *gcBits
	allocCache uint64
	...
}
```

- `startAddr` 和 `npages` — 确定该结构体管理的多个页所在的内存，每个页的大小都是 8KB；
- `freeindex` — 扫描页中空闲对象的初始索引；
- `allocBits` 和 `gcmarkBits` — 分别用于标记内存的占用和回收情况；
- `allocCache` — `allocBits` 的补码，可以用于快速查找内存中未被使用的内存；

runtime.mspan会以两种不同的视角看待管理的内存，当结构体管理的内存不足时，运行时会以页为单位向堆申请内存：

![内存管理单元与页](/Users/liuyangyang/刘阳阳/2021-interview/images/内存管理单元与页.png)



当用户程序或者线程向 [`runtime.mspan`](https://draveness.me/golang/tree/runtime.mspan) 申请内存时，它会使用 `allocCache` 字段以对象为单位在管理的内存中快速查找待分配的空间：

![内存管理单元与对象](/Users/liuyangyang/刘阳阳/2021-interview/images/内存管理单元与对象.png)

如果我们能在内存中找到空闲的内存单元会直接返回，当内存中不包含空闲的内存时，上一级的组件 [`runtime.mcache`](https://draveness.me/golang/tree/runtime.mcache) 会为调用 [`runtime.mcache.refill`](https://draveness.me/golang/tree/runtime.mcache.refill) 更新内存管理单元以满足为更多对象分配内存的需求。

跨度类

[`runtime.spanClass`](https://draveness.me/golang/tree/runtime.spanClass) 是 [`runtime.mspan`](https://draveness.me/golang/tree/runtime.mspan) 的跨度类，它决定了内存管理单元中存储的对象大小和个数：Go 语言的内存管理模块中一共包含 67 种跨度类，每一个跨度类都会存储特定大小的对象并且包含特定数量的页数以及对象，所有的数据都会被预选计算好并存储在 [`runtime.class_to_size`](https://draveness.me/golang/tree/runtime.class_to_size) 和 [`runtime.class_to_allocnpages`](https://draveness.me/golang/tree/runtime.class_to_allocnpages) 等变量中：除了上述 67 个跨度类之外，运行时中还包含 ID 为 0 的特殊跨度类，它能够管理大于 32KB 的特殊对象

mspan全景图

![mspan全景图](/Users/liuyangyang/刘阳阳/2021-interview/images/mspan全景图.jpeg)

##### mcache

```go
type mcache struct {
	// The following members are accessed on every malloc,
	// so they are grouped here for better caching.
	nextSample uintptr // trigger heap sample after allocating this many bytes
	scanAlloc  uintptr // bytes of scannable heap allocated

	// Allocator cache for tiny objects w/o pointers.
	// See "Tiny allocator" comment in malloc.go.

	// tiny points to the beginning of the current tiny block, or
	// nil if there is no current tiny block.
	//
	// tiny is a heap pointer. Since mcache is in non-GC'd memory,
	// we handle it by clearing it in releaseAll during mark
	// termination.
	//
	// tinyAllocs is the number of tiny allocations performed
	// by the P that owns this mcache.
	tiny       uintptr
	tinyoffset uintptr
	

	// The rest is not accessed on every malloc.

	alloc [numSpanClasses]*mspan // spans to allocate from, indexed by spanClass

	stackcache [_NumStackOrders]stackfreelist

	// flushGen indicates the sweepgen during which this mcache
	// was last flushed. If flushGen != mheap_.sweepgen, the spans
	// in this mcache are stale and need to the flushed so they
	// can be swept. This is done in acquirep.
	flushGen uint32
}
```



mcache是线程内存，结构和其他组件的层级关系如上两图。

mcache是和P绑定的，线程内无锁访问，所以速度很快。

线程缓存在刚刚初始化时是不包含runtime.mspan结构的，只有当用户程序申请内存时才会从上一级组件获取新的runtime.mspan满足分配需求。运行时在初始化线程缓存时，调用系统栈中的runtime.mheap中的线程缓存分配器初始化新的runtime.mcache结构。完成后，runtime.mcache中的所有runtime.mspan都是空的占位符emptymspan。应用程序申请的时候，会为线程缓存获取一个指定跨度类的内存管理单元，替换到线程缓存中去。

结构里面的tiny等字段，用于分配微对象，专门管理16byte以下的对象。微分配器只会分配非指针类型的内存。

![tiny内存分配器](/Users/liuyangyang/刘阳阳/2021-interview/images/tiny内存分配器.jpeg)

##### mcentral

```go
type mcentral struct {
	spanclass spanClass

	// partial and full contain two mspan sets: one of swept in-use
	// spans, and one of unswept in-use spans. These two trade
	// roles on each GC cycle. The unswept set is drained either by
	// allocation or by the background sweeper in every GC cycle,
	// so only two roles are necessary.
	//
	// sweepgen is increased by 2 on each GC cycle, so the swept
	// spans are in partial[sweepgen/2%2] and the unswept spans are in
	// partial[1-sweepgen/2%2]. Sweeping pops spans from the
	// unswept set and pushes spans that are still in-use on the
	// swept set. Likewise, allocating an in-use span pushes it
	// on the swept set.
	//
	// Some parts of the sweeper can sweep arbitrary spans, and hence
	// can't remove them from the unswept set, but will add the span
	// to the appropriate swept list. As a result, the parts of the
	// sweeper and mcentral that do consume from the unswept list may
	// encounter swept spans, and these should be ignored.
	partial [2]spanSet // list of spans with a free object
	full    [2]spanSet // list of spans with no free objects
}
```

mcentral是内存分配器的中心缓存，与线程缓存不同，访问中心缓存的内存管理单元需要使用互斥锁。

每个中心缓存都会管理某个跨度类的内存管理单元，他会同时持有两个runtime.spanSet，分别存储包含空闲对象的span列表和不包含空闲对象的内存管理单元。

线程缓存会向中心缓存获取自己的内存管理单元，流程大致如下（内存管理单元就是mspan）

-从清理过的，包含空闲空间的spanset结构中查找可以使用的内存管理单元

-从未被清理过的，包含空闲空间的spanset结构中查找可以使用的内存管理单元

-从未被清理过的，不包含空闲空间的spanset中获取内存管理单元，并清理它的内存空间

-向heap申请新的内存管理单元

-更新内存管理单元的allocCache等字段帮助快速分配内存

无论通过哪种方法获取到了内存单元，该方法的最后都会更新内存单元的 `allocBits` 和 `allocCache` 等字段，让运行时在分配内存时能够快速找到空闲的对象。

##### mheap

[`runtime.mheap`](https://draveness.me/golang/tree/runtime.mheap) 是内存分配的核心结构体，Go 语言程序会将其作为全局变量存储，而堆上初始化的所有对象都由该结构体统一管理，该结构体中包含两组非常重要的字段，其中一个是全局的中心缓存列表 `central`，另一个是管理堆区内存区域的 `arenas` 以及相关字段。

页堆中包含一个长度为 136 的 [`runtime.mcentral`](https://draveness.me/golang/tree/runtime.mcentral) 数组，其中 68 个为跨度类需要 `scan` 的中心缓存，另外的 68 个是 `noscan` 的中心缓存：![golang内存全景图](/Users/liuyangyang/刘阳阳/2021-interview/images/golang内存全景图.jpeg)

