---
title: libmalloc 知识学习（一）结构
updated: 2022-01-01 13:14
---

## 大概说说内存
内存一台计算机中所有程序共享的重要的系统资源。程序将自己的代码和数据装载进内存中才能运行起来。为了使用连续的地址空间，以及进程/内核之间的内存隔离与共享等等诸多优点，现在几乎所有的操作系统都会使用**虚拟内存**。当程序代码访问到逻辑地址时，**MMU**(memory management unit)通过页表将逻辑地址转换为实际的硬件地址。如果应用程序访问一段当前不在物理 RAM  中的内存页上的地址，就会发生 **page fault**。此时虚拟内存系统会调用一个特殊的页面错误处理程序来立即响应错误。页面错误处理程序停止当前正在执行的代码，寻找一个空闲的物理内存页面，从磁盘加载包含所需数据的页面，更新页表，然后将控制权返回给程序代码。这个过程被称为**paging**。请注意这个过程是有些耗时的。通过优化掉这些耗时，可以一定程度提高App的启动速度。 这就是 [抖音二进制重排提高启动速度](https://www.infoq.cn/article/b8m2hrjwbryr7tasurhq) 的原理。

如果物理内存用完了，处理程序必须释放掉一些现有的页面为新页面腾出空间。释放页面的方式在不同系统上不一样:Mac OS 上会通过 page out -- page in 的方式在内存和硬盘中传递数据，在iOS 中，page out到磁盘的部分被内存数据压缩方法取代。不过仍然会有只读数据被存放在磁盘中。

Mac OS 和 iOS 都提供了多种分配方式。但最终所有内存分配最终都使用 libmalloc 库来创建内存。甚至 Cocoa 对象最终也是使用 **libmalloc** 库分配的。使用这个单一库的主要好处是可以使性能工具可以报告应用程序中的所有内存分配。

本文将主要介绍**lbmalloc的结构**和**简单的内存申请流程**。

## libmalloc 的结构
首先，从小到大记住几个关键名词:**block，region，magazine，rack，zone**

##### block
 block 也可以称为 quantum。是libmalloc 所分配内存的最小单位。不同分配情况下block的大小不同，但是固定的。每个block的大小是0x10byte(或者0x200byte)。当请求的size小于0x10 时，分配一个block，介于0x10 和 0x20 之间时，分配两个block。libmalloc 会将free过后的block添加到free_lists 中。注意，一个block的大小刚好存放2个指针（0x8byte）。当block 被释放后，可以存放pre/next 两个指针
##### zone
libmalloc 中主要的操作对象。内存分配的主要入口，类型上可以分为scalable_zone 和 nano_zone 两类

##### rack
zone 中包含着许多种类的rack。以scalable_zone为例，使用不同的rack 和申请内存的size有关。在不同的rack里，block的大小是不一样的。
> **tiny_rack**: size < 1009byte  
> **samll_rack**:1009byte ~ 32KB  
> **mediun_rack**: 32KB~8MB

##### magazine
magazine是真正操作堆内存的对象。rack 中有magazine的数组magazines，其中magazine 的个数是由CPU(logical CPU)的个数决定的。  

其中，logicalCPU = 物理CPU数*每个物理CPU的核心数  

##### heap region

heap region 是分配给用户的的内存的真正区域  


## 从大到小，从zone 到 block 
##### zone
在libmalloc 中 zone 分2种类型: **scalable_zone** 和 **nano_zone**。分配size小于256byte，使用nano_zone，否则使用scalable_zone。其中 nano_zone 还区分为v1，v2两个版本，v2是v1 的升级版本。当启动时设置了v2 mode 为force 或者 enable 的是时候，会使用v2，否则其他情况下会使用v1 mode 的 nano_zone  

除了scalable_zone 和 nano_zone 之外，还有 **default_zone** 和 **helper_zone** 两个辅助的zone，其中default_zone 是一个虚拟的zone ，负责寻找对应的zone来分配内存， helper_zone是一个scalable_zone，用来分配其他的zone。  

**malloc_zones** 是libmalloc的一个全局变量指针，指向一个zone指针数组，Zone对象通过系统调用(mach_ vm_ map)申请内存，申请后的Zone对象指针存放在zone指针数组中。nano_zone 和 scalable_zone 都会在启动后懒加载初始化1个。  

当执行malloc 时，会调用 malloc_zone_malloc(default_zone, size) 。其中default_zone 是一个静态的zone,是一个 virtual_default_zone 结构。其内部存放了default_zone 所需要的函数指针。如default_zone_size,default_zone_malloc 等。而default_zone 本质上不会参与函数分配，而是寻找malloc_zones 中的第0个zone 来分配内存。(default_zone 存在的意义应该是对外提供接口，方便开发者写自己的default_zone, 监听内存申请释放流程)。下面是 default_zone_malloc 函数的实现

```
static void *
default_zone_malloc(malloc_zone_t *zone, size_t size)
{
        zone = runtime_default_zone();

        return zone->malloc(zone, size);
}
```
  

libmalloc 的外层大致结构如下图所示，静态指针malloc_zones 管理着nano_zone 和 scalable_zone。外部通过default_zone 调用真实的zone 来分配内存。此外也可以看到scalable_zone  中所管理的3个rack。内存分配的核心操作在scalable_zone 和 nano_zone中

![Minion](https://s2.loli.net/2022/01/02/bghnet5PTiUYF34.png)

内存在nano_zone 和 scalable_zone中的分配流程还不太一样，其中scalable_zone比较复杂一点。后续的例子都以scalable_zone为例。  

下面这段代码是zone根据size走不同的rack进行分配的流程
```
MALLOC_NOINLINE void *
szone_malloc_should_clear(
    szone_t *szone, 
    size_t size, 
    boolean_t cleared_requested)
{
        void *ptr; // 返回给上层的指针
        msize_t msize; //size转换成需要的block数目

        if (size <= SMALL_THRESHOLD) {
                // tiny size: <=1008 bytes (64-bit), <=496 bytes (32-bit)
                // think tiny
                msize = TINY_MSIZE_FOR_BYTES(size + TINY_QUANTUM - 1);
                if (!msize) {
                        msize = 1;
                }
                ptr = tiny_malloc_should_clear(&szone->tiny_rack, msize, cleared_requested);
        } else if (size <= szone->large_threshold) {
                // small size: <=15k (iOS), <=64k (large iOS), <=128k (macOS)
                // think small
                msize = SMALL_MSIZE_FOR_BYTES(size + SMALL_QUANTUM - 1);
                if (!msize) {
                        msize = 1;
                }
                ptr = small_malloc_should_clear(&szone->small_rack, msize, cleared_requested);
        } else {
                // large: all other allocations
                size_t num_kernel_pages = round_page_quanta(size) >> vm_page_quanta_shift;
                if (num_kernel_pages == 0) { /* Overflowed */
                        ptr = 0;
                } else {
                        ptr = large_malloc(szone, num_kernel_pages, 0, cleared_requested);
                }
        }

        //内存涂鸦
        if ((szone->debug_flags & MALLOC_DO_SCRIBBLE) && ptr && !cleared_requested && size) {
                memset(ptr, SCRIBBLE_BYTE, szone_size(szone, ptr));
        }

        return ptr;
}

```

rack 和 magazine的关系如下图。每个rack中持有一个长度为逻辑CPU数的magazine数组（libmalloc-166.200.60 中没有medium_rack）
![](https://s2.loli.net/2022/01/02/OiXJInhQyuKredZ.png)

rack 中的变量magazines 是 magazine的数组。数组大小由logic CPU数来决定。实际magazines 的count = logical_cpu_num + 1。其中多出来的一个是depot_magazine。当1个magazine的内存不够分配时，会去depot_magazine 中请求内存。数组中每个magazine都会有一把自己的锁，这样做可以降低锁的范围，在线程层面做到并发，提高内存申请速度。
来看下rack结构的声明
```
//来看下rack结构的声明
typedef struct rack_s {
        /* Regions for tiny objects */
        _malloc_lock_s region_lock MALLOC_CACHE_ALIGN;

        rack_type_t type; //类型
        size_t num_regions; //有多少个region
        size_t num_regions_dealloc;
        region_hash_generation_t *region_generation;
        region_hash_generation_t rg[2];
        region_t initial_regions[INITIAL_NUM_REGIONS];

        int num_magazines; //magazine的个数
        unsigned num_magazines_mask; //计算应该用哪个magazine的辅助变量
        int num_magazines_mask_shift;
        uint32_t debug_flags;

        // array of per-processor magazines
        magazine_t *magazines; //magazine 数组

        uintptr_t cookie;
        uintptr_t last_madvise;
} rack_t;

```


magazine 是真正操作内存的对象。来看一下其中几个重要的属性
1. **mag_last_free** 上次释放的地址/ mag_last_free_msize:上次释放的size/mag_last_free_rgn:上次释放的region 。这三个属性的作用是形成一个cache。当本次申请的size和上次释放的size一致的时候，直接返回上次释放的地址。目的是为了提高速度
2. **mag_bitmap** magazine 中的这个bitmap是管理空闲链表的。如果一个空闲链表中没有结点，mag_bitmap 中对应的bit会被设置为0。请注意不要和heap region中的bitmap 搞混。
3. **mag_free_lists** 空闲链表数组。数组中的元素，是空闲链表。所以这是一个数组套链表的结构。而第1个空闲链表的每个节点内存大小为1个block（16byte），第2个空闲链表的每个节点大小为2个block（32byte），第3个空闲链表的每个节点大小为3个block（48byte），由于Tiny最多一次分配1008byte，所以最后一个空闲链表的每个节点大小为63个block。 但是，最后1个，也就是第64个空闲链表的每个节点大小是大于64个block，这是由于该链表的节点是free的时候合并产生的。
4. **recirculation_entries/firstNode/lastNode** 这三个指针管理了一个双向链表。双链表中的节点是heap region。

```
//magazine的声明
typedef struct magazine_s { 
        // Take magazine_lock first, Depot lock when needed for recirc, then szone->{tiny,small}_regions_lock when needed for alloc
        _malloc_lock_s magazine_lock MALLOC_CACHE_ALIGN;
        // 当前是否在向内核申请heap region
        volatile boolean_t alloc_underway;

        //上一次释放的地址
        void *mag_last_free;
        //上一次释放的大小
        msize_t mag_last_free_msize;
#if MALLOC_TARGET_64BIT
        uint32_t _pad;
#endif
        // 上一次释放的Region
        region_t mag_last_free_rgn;

        //空闲链表
        free_lists_t mag_free_lists[magazine_free_lists_SLOTS];
        //空闲位图
        uint32_t mag_bitmap[magazine_free_lists_BITMAP_WORDS];

        // the first and last free region in the last block are treated as big blocks in use that are not accounted for
        size_t mag_bytes_free_at_end;
        size_t mag_bytes_free_at_start;
        // 最后一次向内核申请的heap region地址
        region_t mag_last_region; 
        
        
        size_t mag_num_bytes_in_objects; // 已经分配出去的内存的大小
        size_t num_bytes_in_magazine; // magazine 中总共有多少大小
        unsigned mag_num_objects; //region 数量

        //循环链表和首位节点
        unsigned recirculation_entries;
        region_trailer_t *firstNode;
        region_trailer_t *lastNode;

#if MALLOC_TARGET_64BIT
        uintptr_t pad[320 - 14 - magazine_free_lists_SLOTS -
                        (magazine_free_lists_BITMAP_WORDS + 1) / 2];
#else
        uintptr_t pad[320 - 16 - magazine_free_lists_SLOTS -
                        magazine_free_lists_BITMAP_WORDS];
#endif

} magazine_t;

```

tiny_regin 的结构声明如下面的代码

```
// tiny_regin 的结构
typedef struct tiny_region {
        // region 所持有的全部block
        tiny_block_t blocks[NUM_TINY_BLOCKS];
        // 追踪器，管理内部的blocks
        region_trailer_t trailer;
        // 标记header是否被使用的数据结构，是header 和 used 组成的
        tiny_header_used_pair_t pairs[CEIL_NUM_TINY_BLOCKS_WORDS];

        uint8_t pad[TINY_REGION_SIZE - (NUM_TINY_BLOCKS * sizeof(tiny_block_t)) - TINY_METADATA_SIZE];
} * tiny_region_t;
```

heap region 是调用内核 mach_vm_map 分配的，是用户可以实际操作的内存。以1个Tiny region来举例子看。Region 可以分为 data 和 meta data两个区。 蓝色的区域表示被分配出去的block或者是刚刚释放的block，绿色的data区可以作为block分配出去，也可以作为空闲链表的一个节点，存放两个指针。 meta data 中主要存放了将region串在一起的pre/next 指针，以及bitmap。其中bitmap中又包含了2个子bitmap。
![](https://s2.loli.net/2022/01/02/d7mozIOW2KtpPgy.png)


heap region和magazine的关系
- heap regions 会用自身的两个pre/next指针将自己和其他的heap region串起来成为一个双向链表
- magazine的first_node和last_node指向heap region metaData双向链表的第一个和最后一个节点
- magazine的mag_last_region指向最后一次magazine向内核申请的heap region
![](https://s2.loli.net/2022/01/02/5CnjHZ9qDPFeRT1.png)

这里我们稍微停留一下，看看region中的bitmap和magazine中的空闲链表是怎么工作的
首先，bitmap是两个子bit组成的，分别可以成为header bitmap 和 used bitmap，它们中间用0xFFFFFFFF来隔开。每个block在子bitmap中占一个bit，也就是说在bitmap中一个block占2个bit。header bitmap 用来标记一块内存区域的起点，如果是1，表示是一个起点（head），used bitmap 用来标记一个区域是否是在使用中，1表示使用中，0表示空闲
bitmap使用下面3条规则:
- 如果一块区域已经被使用，那么这块区域的第一个block在header bitmap 和 used bitmap中对应的值都是1.  
- 如果一块区域是空闲的，那么这块区域的第一个 block 在 header bitmap 和 used bitmap中对应的值分别是1和0.  
- 其他情况下，quantum在 header bitmap中的值是0.

header bitmap 用来标记一个块的起始位置，如果为1，就是一个块的起始。那么如何判断这个block是已经分配还是空闲的呢？在used bit map 中为1的就是1个已分配块的起始。如果header bitmap 为1 ，used bitmap 为0，那就是一个空闲block的起始。其实很简单，header标记一个块的起始，used 记录使用状态
![](https://s2.loli.net/2022/01/02/B2pEZkL73HYxs9d.png)

那么我们以上图为例，阐述下空闲区1是怎么被定位的:
1. 从左往右扫描，发现q0在两个bitmap中的值都是1，那么表明q0是一块已使用区域的开始。
2. 继续扫描，发现q2到qi-1在header bitmap中的值都是0，说明这是已使用区域的一部分。
3. 扫描到qi，发现qi在header bitmap中是1，in use bitmap中是0，说明这是空闲块的起始
4. 继续扫往下扫描就好了...

##### free_lists
free_lists是magazine 的空闲链表数组。释放过的内存会作为节点存储在空闲链表中。数组元素空闲链表按照内存块大小递增。其中，第1个空闲链表，每个节点的大小是1个block（16byte）；第2个空闲链表每个节点的大小是2个block（32byte）；第3个空闲链表的大小为（48byte）；依次类推下去。

对于tiny_rack最多分配1008个字节，因此该rack下的每个magazine的空闲链表数组个数最大为1008/16 + 1 = 63，第63个空闲链表的每个节点内存为63blocks(=1008B)。但是空闲链表的实际个数为63+1 = 64 个。最后一个第64个空闲链表的每个节点内存大于等于64blocks(>=1024B）该链表的节点是free的时候合并产生的。
空闲链表节点的结构非常简单，只有pre,next 两个指针。所以对于16byte的空闲链表节点，其中只能存放pre，next两个指针。
对于大于16byte小于等于1008byte的空闲链表的节点。前面16byte存放pre/next指针，指针后的两个字节存放块大小。剩余空间就是分配出去供使用的内存了。
![](https://s2.loli.net/2022/01/02/B2pEZkL73HYxs9d.png)


## 总结一下
- <big>在iOS/MacOS 上，内存管理使用了唯一的库libmalloc。所有的内存申请和释放都会通过libmalloc 进行。这样做的好处是方便内存管理，也方便一些内存分析工具的开发建设</big>
- <big>开发者可以通过接受libmalloc的回调函数来了解内存的使用情况。Instrument中很多调试工具也是通过这个原理开发的</big>
- <big>libmalloc 的结构:zone，rack，magazine，heap region</big>
- <big>libmalloc 管理内存的主要方式是空闲链表和bitmap，其中bitmap是由header和used 两个子map组成</big>


## 知识来源参考
[tannerjin 的 blog](https://tannerjin.github.io/2019/12/16/iOS%E5%86%85%E5%AD%98-%E5%A0%86-libmalloc-magazine-zone/)  
[luchenzong 的blog](https://luchengzhong.github.io/gitblog/2018/06/12/iOS-%E5%86%85%E6%A0%B8-%E6%BA%90%E7%A0%81%E8%A7%A3%E8%AF%BB-3-%E8%AF%A6%E8%A7%A3iOS%E6%98%AF%E6%80%8E%E4%B9%88malloc%E7%9A%84-%E4%B8%8A.html)
