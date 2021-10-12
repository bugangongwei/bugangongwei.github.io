---
title: 内存分配
date: 2021-08-23 14:10:00
tags: [GO]
categories: GO
comments: true
---

# 概念
1. 内存分配方法: 
(1) 线性分配器: 内存中只需要维护一个指针, 指针指向上次内存分配到的地址, 用户程序向内存分配器申请内存的时候, 分配器只需要检查剩余的空闲内存, 把合适大小的空闲内存分配给用户程序, 然后指针指向最新的空闲内存地址;
线性分配器的优点, 实现复杂度低, 执行速度快; 缺点, 内存释放后, 被释放的内存块将无法被重新利用;
(2) 空闲链表分配器: 维护一个空闲链表, 分配内存时, 遍历链表, 找到可分配的内存块并返回;
遍历策略: 首次适应, 循环首次适应, 最优适应, 隔离适应;
其中, 隔离适应策略会将内存分割成不同大小的内存块, 每种大小的内存块被组织成一个链表, 当应用程序需要一块 8M 的内存时, 找到 8M 内存对应的链表, 通过遍历, 适应到合适的内存块;

2. 分级分配
TCMalloc(Thread-Caching Malloc) 的核心理念, 就是利用多级缓存将对象大小进行分类, 并按照类别实施不同的分配策略;
|| 对象类别 || 大小 ||
| 微对象 | (0, 16B) |
| 小对象 | [16B, 32KB] |　
| 大对象 | (32KB, ) |
缓存分级:
(1) 线程缓存: 每个线程拥有的独立缓存, 微对象和小对象;
(2) 中心缓存: 线程缓存满时, 可以作为补充, 小对象;
(3) 页堆: 大对象;

3. 二维稀疏内存
````go

type heapArena struct {
  bitmap       [heapArenaBitmapBytes]byte
  spans        [pagesPerArena]*mspan
  pageInUse    [pagesPerArena / 8]uint8
  pageMarks    [pagesPerArena / 8]uint8
  pageSpecials [pagesPerArena / 8]uint8
  checkmarks   *checkmarksMap
  zeroedBase   uintptr
}

````
(1) bitmap: 用于标志 arena 区域哪些地址保存了对象;
(2) spans: 是一个存储了 mspan 结构体指针的数组;
一个 spans 数组 spans[1, 2, 2, 3], arena 区域的页 arena[1, 1, 1, 1];
spans[0] = 1, 指向一个 m0 = mspan{}, 对应一个 page arena[0];
spans[1] = 2, 指向一个 m1 = mspan{}, 对应一个 page arena[1];
spans[2] = 2, 指向一个 m1 = mspan{}, 对应一个 page arena[2];
m1 分配了 2 个 page 的内存;
spans[3] = 3, 指向一个 m2 = mspan{}, 对应一个 page arena[3];
每个 *span 对应 arena 中的一个 page, 不同的 span 下标可以存储指向同一个 mspan 的指针, 表示这个 mspan 分配了 n*page 的内存;
(3) zeroedBase 字段指向了该结构体, 也就是一个 heapArena 管理的一段内存的基地址; 我理解就是 heapArena 的 arena 的基地址;

Linux 的 x86-64 的机器上, golang 的二维稀疏数组为 `[1][4,194,304]*heapArena`, 每个指针占用 8B 内存空间, `size = 1*4,194,304*8B = 32MB`;
而每个堆区 ``heapArena`` 会管理 ``64MB`` 的内存, 所以, 整个堆区最多可以管理 ``32MB * 64MB = 256TB`` 的内存;


4. 地址空间
golang 对操作系统的内存管理进行了一次抽象, 该抽象层将运行时内存管理的地址空间分成以下四个状态
|| 状态 || 解释 || 转换 ||
| None | 内存初始状态 | Reserve, Prepared, Ready - runtime.sysFree -> None |
| Reserved | 运行时持有该地址空间, 但是访问该内存空间会报错 | None - sysReserve -> Reserve; Prepared, Ready - sysFault -> Reserve |
| Prepared | 运行时持有该地址空间, 但是还没有对应的物理内存, 但是可以快速转换到 Ready 状态 | Ready - sysUnused -> Prepared; Reserve - sysMap -> Prepared |
| Ready | 内存可以被安全访问 | None - sysAlloc -> Ready; Prepared - sysUsed -> Ready |

5. 数据结构

5.1 mspan

````go
type mspan struct {
  next *mspan
  prev *mspan
  ...

  startAddr uintptr // 起始地址
  npages    uintptr // 页数, 每个页的大小是 8KB
  freeindex uintptr // 扫描页中空闲对象的初始索引

  allocBits  *gcBits // 标记标记内存被 Object 占用的情况; 0 表示空闲, 1 表示被占用;
  gcmarkBits *gcBits // 标记标记内存的回收情况
  allocCache uint64 // allocBits 的补码，可以用于快速查找内存中未被使用的内存；0 表示被占用, 1 表示被空闲;
  ...

  state mSpanStateBox // 
  ...

  spanclass spanClass // 对象大小类别, golang 里面有 (68-1) 种
}
````
mSpanStateBox 四种状态:
(1) mSpanFree: mSpan 处于空闲堆中;
(2) mSpanInUse: mSpan 被分配时;
(3) mSpanManual: mSpan 被分配时;
(4) mSpanDead

spanClass
spanClass 是一个 uint8 类型的整数, 前7位存储跨度类的 ID, 最后1位存储 noscan 值, noscan 表示结构体是否包含指针, 如果包含, 则需要 scan, 不包含则垃圾回收可以直接清除, 不需要 scan;
````go
type spanClass uint8

const (
  numSpanClasses = _NumSizeClasses << 1
  tinySpanClass  = spanClass(tinySizeClass<<1 | 1)
)

func makeSpanClass(sizeclass uint8, noscan bool) spanClass {
  return spanClass(sizeclass<<1) | spanClass(bool2int(noscan))
}

func (sc spanClass) sizeclass() int8 {
  return int8(sc >> 1)
}

func (sc spanClass) noscan() bool {
  return sc&1 != 0
}

````

跨度类
````go
_NumSizeClasses = 68

// 对象大小
// 0 表示大对象, size > 32KB
var class_to_size = [_NumSizeClasses]uint16{0, 8, 16, 24, 32, 48, 64, 80, 96, 112, 128, 144, 160, 176, 192, 208, 224, 240, 256, 288, 320, 352, 384, 416, 448, 480, 512, 576, 640, 704, 768, 896, 1024, 1152, 1280, 1408, 1536, 1792, 2048, 2304, 2688, 3072, 3200, 3456, 4096, 4864, 5376, 6144, 6528, 6784, 6912, 8192, 9472, 9728, 10240, 10880, 12288, 13568, 14336, 16384, 18432, 19072, 20480, 21760, 24576, 27264, 28672, 32768}

// 可分配页数
// 0 表示大对象, size > 32KB
var class_to_allocnpages = [_NumSizeClasses]uint8{0, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 2, 1, 2, 1, 2, 1, 3, 2, 3, 1, 3, 2, 3, 4, 5, 6, 1, 7, 6, 5, 4, 3, 5, 7, 2, 9, 7, 5, 8, 3, 10, 7, 4}
````


5.2 mcache
````go
type mcache struct {
  // 微对象分配器, 专门用来管理 16B 以下的微对象的分配
  tiny       uintptr
  tinyoffset uintptr
  tinyAllocs uintptr

  // numSpanClasses = 68 * 2
  alloc [numSpanClasses]*mspan // spans to allocate from, indexed by spanClass
  ...
}

````
线程缓存在刚刚被初始化时是不包含 runtime.mspan 的，只有当用户程序申请内存时才会从上一级组件获取新的 runtime.mspan 满足内存分配的需求;
mcache 初始化: p 调用 runtime.allocmcache -> runtime.mheap 使用缓存分配器分配一个空的 mcache;
mcache 某个跨度类的链表满了: runtime.mcache.refill, 用有空闲内存的单元去替换已经满掉的单元;
`` numSpanClasses = _NumSizeClasses << 1 `` 数组里一半的mspan中分配的对象不包含指针，另一半则包含指针;

5.3 mcentral
````go
type mcentral struct {
  // 跨度类
  spanclass spanClass
  // 还有空闲空间的列表
  // [0] 清理过的、包含空闲空间的
  // [1] 从未被清理过的、有空闲空间的
  partial  [2]spanSet
  // 已经没有空闲空间的列表
  // [0] 清理过的、不包含空闲空间的
  // [1] 未被清理的、不包含空闲空间的
  full     [2]spanSet
}

````
(1) mcache 的某个 spanClass 已经没有了空闲空间, 则去 mcentral 列表中查找到相应的 spanClass 的 mcentral 结构题体;
(2) 找到这个结构体之后, 按照 partial[0], partial[1] 的顺序查找可以使用的 mspan;
(3) 顺便清理一下 full[1]
(4) 再向 mheap 申请新的内存空间
(5) 更新 allocBits 和 allocCache 字段


5.4 heapArena
````go
type heapArena struct {
  bitmap [heapArenaBitmapBytes]byte
  spans [pagesPerArena]*mspan

  pageInUse [pagesPerArena / 8]uint8
  pageMarks [pagesPerArena / 8]uint8
  pageSpecials [pagesPerArena / 8]uint8
  checkmarks *checkmarksMap

  zeroedBase uintptr
}


````

5.4 mheap
````go
type mheap struct {
  lock      mutex
  pages     pageAlloc // page allocation data structure
  sweepgen  uint32    // sweep generation, see comment in mspan; written during STW
  sweepdone uint32    // all spans are swept
  sweepers  uint32    // number of active sweepone calls
  ...

  allspans []*mspan // all spans out there

  pagesInUse         uint64  // pages of spans in stats mSpanInUse; updated atomically
  pagesSwept         uint64  // pages swept this cycle; updated atomically
  pagesSweptBasis    uint64  // pagesSwept to use as the origin of the sweep ratio; updated atomically
  sweepHeapLiveBasis uint64  // value of heap_live to use as the origin of sweep ratio; written with lock, read without
  sweepPagesPerByte  float64 // proportional sweep ratio; written with lock, read without

  arenas [1 << arenaL1Bits]*[1 << arenaL2Bits]*heapArena

  heapArenaAlloc linearAlloc
  arenaHints *arenaHint
  arena linearAlloc

  allArenas []arenaIdx

  sweepArenas []arenaIdx

  markArenas []arenaIdx

  curArena struct {
    base, end uintptr
  }

  central [numSpanClasses]struct {
    mcentral mcentral
    pad      [cpu.CacheLinePadSize - unsafe.Sizeof(mcentral{})%cpu.CacheLinePadSize]byte
  }

  spanalloc             fixalloc // allocator for span*
  cachealloc            fixalloc // allocator for mcache*
  specialfinalizeralloc fixalloc // allocator for specialfinalizer*
  specialprofilealloc   fixalloc // allocator for specialprofile*
  speciallock           mutex    // lock for special record allocators.
  arenaHintAlloc        fixalloc // allocator for arenaHints

  unused *specialfinalizer // never set, just here to force the specialfinalizer type into DWARF
}

````
mheap 中有 `68*2` 个 mcentral 结构;

















