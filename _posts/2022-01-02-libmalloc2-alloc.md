---
title: libmalloc 知识学习（二）申请
updated: 2022-01-02 18:00
---

在上一篇[libmalloc 知识学习（一）结构](libmalloc1-strcut)中了解了 Apple 广泛使用的内存库libmalloc，学习了libmalloc的组成结构，这一篇文章将会重点研究libmalloc是如何分配内存的。

当用户调用malloc申请一块内存之后，libmalloc就开始为此忙碌起来。首先，它要选择合适的zone (nano_zone 和 scalcables_zone) 来承接需求，其次要根据所申请的size 调用zone内部的的函数申请内存。在申请的时候还需要判断是否命中cache，是否需要向内核申请新的内存页等复杂逻辑，最后获取到可用内存的地址。在交付给上层使用之前，还会根据你的需求帮你清理一下所申请的内存区域里的数据。

对于libmalloc来说，用户所申请内存的size是影响分配行为的关键。申请size有大有小，有串行有并行。如何合理的根据size给到用户内存同时保持高效。这是libmalloc的关键逻辑。
据《结构》 中的介绍，256Byte，1009Byte，32KB和8MB分别是4个分水岭。所以本篇通过申请4个不同大小的内存，来详细探究下申请流程，尝试在探究过程中学习libmalloc设计的优点。我们将分别申请 13Byte，261Byte，1010Byte，1.1MB的内存，通过图示加说明的方式来仔细探究下详细的申请流程

> ☕️食用方法☕️：本文是一篇对libmalloc 申请逻辑的指引手册。建议在 libmalloc 源码之前先通读本文，结合文中的图和流程来阅读源码，可以大幅提升学习效率。


## 申请13Byte
当size 在 [0,256] Byte 区间范围内的时候，会通过 nano_zone 进行分配，其流程图如下

![申请13B内存](https://s2.loli.net/2022/01/02/GmpYRMCTyWqvfw1.png)

1. 首先,malloc(13)  会进入malloc.c 中调用malloc(size_t size)，该函数只有1句，会调用**_malloc_zone_malloc(default_zone, size, MZ_POSIX)**
2. 而后调用default_zone 获取runtime_default_zone,然后调用zone的malloc，这里获取到的是**nanozone**
3. 调用nano_zone 的 **nano_malloc(nanozone_t *nanozone, size_t size)** ，里会判断是否在nano的分配范围内即是否小于 **NANO_MAX_SIZE (256Byte)** 。如果超过了，则会去请求helper_zone的帮助
4. 如果在 NANO_MAX_SIZE 范围内，则会调用进 _nano_malloc_check_clear 函数，这是nano_zone 分配的内存的核心函数
5. 在_nano_malloc_check_clear函数中，首先会进行 **segregated_size_to_fit** 这步操作主要目的是对size进行取整，方便后续分配。slot_bytes是大于size的最小的16的倍数。当申请size被47时。slot_bytes 就是48。如本次我们请求的size是13，经过此步骤后，size已经变为16。同时也会计算出本次申请所对应的slot_key, 根据slot_key 可以获取到对应线程magazine中的freelists 中对应尺寸的freelist。
6. 而后，通过正在运行的CPU number获取到对应的magazine。通过magazine 和 slot_key 从magazine中拿到 **nano_meta_admin_t**
```
//获取meta
nano_meta_admin_t pMeta = &(nanozone->meta_data[mag_index][slot_key]);
```
7. 通过 OSAtomicDequeue(&(pMeta->slot_LIFO), offsetof(struct chained_block_s, next)) 从freelist 中取出ptr。
8. 如果拿到了ptr，会判断是否需要清理内存然后直接返回给上层。如果没有拿到ptr，会继续走segregated_next_block 函数  

以上就是申请13Byte内存时libmalloc的主要流程。

> ☕️ 喝一杯咖啡休息一下，接下来在来看看261Btye的内存是怎么申请到手的


## 申请261Byte
申请261Byte超过了 nano的 256Byte的限制，申请流程又有什么不同呢？先上流程图，再来看代码。

![申请261B内存](https://s2.loli.net/2022/01/02/fzeX4mVsZvobuy1.png)

1. 出乎意料的是在获取**runtime_default_zone**的时候，这里的流程和nano_zone 一致。也就是说获取到的还是一个**nano_zone**。
2. 逻辑分岔点出现在调用 nano_malloc 的时候。当判断size超过了 **NANO_MAX_SIZE** 也就是256Byte之后，系统会请求helper zone的帮助。此时，scalcable_zone 的故事也就开始了。
3. 接下来调用helper_zone 的malloc 函数申请内存。这里有个小细节：在 create_scalable_szone 函数中创建szone的时候，将函数 **szone_malloc** 的地址赋值给了malloc，所以最后会调入 **szone_malloc** 函数中
```
szone->basic_zone.malloc = (void *)szone_malloc;
```
4. szone_malloc 中实现内存申请的重要函数是 **szone_malloc_should_clear**，接下来就看看这个函数是如何分配的内存的
5. 由于内存都是按照block的整数个数来分配的，根据申请的size计算所需block的个数即msize。由于申请261Byte内存，所以得出的mszie 为 2
6. 根据不同的size会走入不同的申请逻辑分支。我们申请的261Byte 内存走入了tiny分支，调用tiny_malloc_should_clear 函数，将rack，msize和cleared_requested 传入。
7. 在 tiny_malloc_should_clear 中首先会检查本次申请的msize和上次释放的msize是否一致，如果是会直接返回上次释放的ptr。这是个简单的cache逻辑
8. 其次在magazine 中的freelist 中查找合适的blocks并返回 ptr
9. 如果magazine 的 freelist 中没有空闲block 了，则会进入 depot_magazine 中寻找空闲的blocks并返回ptr
10. 如果上述操作都失败，则会调用 mvm_allocate_pages 获取新的region 并且分配合适的block 并返回ptr

以上就是通过tiny_rack 申请内存的全部流程了。为了方便理解，这里贴上一段去掉trace 和 log 之后，tiny_malloc_should_clear  函数代码。在szone 中，tiny，small，medium的分配主函数高度相似

```
void *
tiny_malloc_should_clear(rack_t *rack, msize_t msize, boolean_t cleared_requested)
{
        //根据当前的线程，获取对应的magazine
        void *ptr;
        mag_index_t mag_index = tiny_mag_get_thread_index() % rack->num_magazines;
        magazine_t *tiny_mag_ptr = &(rack->magazines[mag_index]);
        
        //对magazine 上锁
        SZONE_magazine_PTR_LOCK(tiny_mag_ptr);

#if CONFIG_TINY_CACHE
        //获取magazine 的lastfree，这是上次释放的地址，如果尺寸匹配
        //就清理一下，将其分配给ptr
        ptr = tiny_mag_ptr->mag_last_free;

        if (tiny_mag_ptr->mag_last_free_msize == msize) {
                // we have a winner
                tiny_mag_ptr->mag_last_free = NULL;
                tiny_mag_ptr->mag_last_free_msize = 0;
                tiny_mag_ptr->mag_last_free_rgn = NULL;
                SZONE_magazine_PTR_UNLOCK(tiny_mag_ptr);
                CHECK(szone, __PRETTY_FUNCTION__);
                if (cleared_requested) {
                        memset(ptr, 0, TINY_BYTES_FOR_MSIZE(msize));
                }
                return ptr;
        }
#endif /* CONFIG_TINY_CACHE */
        
        //last free 不好使了，开始查找free_lists
        while (1) {
                //先看看空闲链表中有没有内存了
                ptr = tiny_malloc_from_free_lists(rack, tiny_mag_ptr, mag_index, msize);
                if (ptr) {
                        SZONE_magazine_PTR_UNLOCK(tiny_mag_ptr);
                        CHECK(szone, __PRETTY_FUNCTION__);
                        if (cleared_requested) {
                                memset(ptr, 0, TINY_BYTES_FOR_MSIZE(msize));
                        }
                        return ptr;
                }
                
                //空闲链表不好使？再看看depot中有没有内存了。depot是属于rack的
                if (tiny_get_region_from_depot(rack, tiny_mag_ptr, mag_index, msize)) {
                        ptr = tiny_malloc_from_free_lists(rack, tiny_mag_ptr, mag_index, msize);
                        if (ptr) {
                                SZONE_magazine_PTR_UNLOCK(tiny_mag_ptr);
                                CHECK(szone, __PRETTY_FUNCTION__);
                                if (cleared_requested) {
                                        memset(ptr, 0, TINY_BYTES_FOR_MSIZE(msize));
                                }
                                return ptr;
                        }
                }

                // The magazine is exhausted. A new region (heap) must be allocated to satisfy this call to malloc().
                // The allocation, an mmap() system call, will be performed outside the magazine spin locks by the first
                // thread that suffers the exhaustion. That thread sets "alloc_underway" and enters a critical section.
                // Threads arriving here later are excluded from the critical section, yield the CPU, and then retry the
                // allocation. After some time the magazine is resupplied, the original thread leaves with its allocation,
                // and retry-ing threads succeed in the code just above.
                
                //上面的办法都不好使了，表示magazine里的内存用完了，这时候就去找系统要内存了，去找系统要一块儿内存。
                if (!tiny_mag_ptr->alloc_underway) {
                        void *fresh_region;

                        // time to create a new region (do this outside the magazine lock)
                        tiny_mag_ptr->alloc_underway = TRUE;
                        OSMemoryBarrier();
                        SZONE_magazine_PTR_UNLOCK(tiny_mag_ptr);
                        fresh_region = mvm_allocate_pages_securely(TINY_REGION_SIZE, TINY_BLOCKS_ALIGN, VM_MEMORY_MALLOC_TINY, rack->debug_flags);
                        SZONE_magazine_PTR_LOCK(tiny_mag_ptr);

                        // DTrace USDT Probe
                        MAGMALLOC_ALLOCREGION(TINY_SZONE_FROM_RACK(rack), (int)mag_index, fresh_region, TINY_REGION_SIZE);
                        
                        //系统也没内存了，地主家也没有余粮了。OOM了
                        if (!fresh_region) { // out of memory!
                                tiny_mag_ptr->alloc_underway = FALSE;
                                OSMemoryBarrier();
                                SZONE_magazine_PTR_UNLOCK(tiny_mag_ptr);
                                return NULL;
                        }

                        ptr = tiny_malloc_from_region_no_lock(rack, tiny_mag_ptr, mag_index, msize, fresh_region);

                        // we don't clear because this freshly allocated space is pristine
                        tiny_mag_ptr->alloc_underway = FALSE;
                        OSMemoryBarrier();
                        SZONE_magazine_PTR_UNLOCK(tiny_mag_ptr);
                        CHECK(szone, __PRETTY_FUNCTION__);
                        return ptr;
                } else {
                        SZONE_magazine_PTR_UNLOCK(tiny_mag_ptr);
                        yield();
                        SZONE_magazine_PTR_LOCK(tiny_mag_ptr);
                }
        }
        /* NOTREACHED */
}

```

> 至此，本文内容已经完成大半，再喝一杯咖啡☕️休息一下，来看看1010Byte的内存是怎么申请到手的

## 申请1010Byte
与申请261Byte相比，申请100Byte的过程几乎完全一样。先上流程图。

![申请1010B内存](https://s2.loli.net/2022/01/02/7otKx21kfAaIJbW.png)
与tiny相比，small 主要的区别在于block大小不一样以及向内核申请内存页时所传入的参数不同。其他逻辑流程都完全一样，本文这里就不再赘述了
```
//tiny
fresh_region = mvm_allocate_pages(TINY_REGION_SIZE, TINY_BLOCKS_ALIGN,
                                        MALLOC_FIX_GUARD_PAGE_FLAGS(rack->debug_flags),
                                        VM_MEMORY_MALLOC_TINY);
//small
fresh_region = mvm_allocate_pages(SMALL_REGION_SIZE, SMALL_BLOCKS_ALIGN,
                                        MALLOC_FIX_GUARD_PAGE_FLAGS(rack->debug_flags),
                                        VM_MEMORY_MALLOC_SMALL);
```

> ☕️OK，来看最后一个申请流程吧

## 申请1.1MB
在探究1.1MB的申请流程之前，先来聊一下medium 
> **Medium 逻辑分支触发条件**  
> 在tiny，small，medim 和 large 四个逻辑中 meidum 逻辑分支的开启有一定的条件。  
> 只有物理内存的总大小大于等于开启要求 magazine_medium_active_threshold （32ull * 1024 * 1024 * 1024）才能开启。
> 即设备的物理内存必须 32个GB 才支持使用medium。  
> 而我做实验的这台iMac 内存只有16GB，并不支持medium ，所以申请 1.1MB时将直接进入large_malloc 函数进行分配。

```
#if CONFIG_MEDIUM_ALLOCATOR || CONFIG_LARGE_CACHE
        // 获取设备内存
        uint64_t memsize = platform_hw_memsize();
#endif // CONFIG_MEDIUM_ALLOCATOR || CONFIG_LARGE_CACHE

#if CONFIG_MEDIUM_ALLOCATOR
        // 判断是否打开medium
        szone->is_medium_engaged = (magazine_medium_enabled &&
                        (memsize >= magazine_medium_active_threshold));
#endif // CONFIG_MEDIUM_ALLOCATOR
```

在介绍 large 的逻辑之前，还需要先了解一下 large_malloc 中的cache。
large_entry_cache 是一个可拓展的环形数组，内部存放的数据结构为large_entrys_t, 其声明如下：

```
// 数组entrys 的声明
typedef struct large_entry_s {
        vm_address_t address; //内存地址
        vm_size_t size; // size
        boolean_t did_madvise_reusable; //是否可以重用？？*待定*
} large_entry_t;
```

cache操作主要是通过下面几个变量：

```
//双指针 old和new
int large_entry_cache_oldest;
int large_entry_cache_newest;

int large_cache_depth; // 环容量
size_t large_cache_entry_limit;//环大小限制，为 (memsize >> 10) memsize 是硬件尺寸

boolean_t large_legacy_reset_mprotect; //一个特殊标记位，不影响主逻辑不去探究了

size_t large_entry_cache_reserve_bytes; // 环内剩余的byte数
size_t large_entry_cache_reserve_limit; // 环内剩余的限制数
size_t large_entry_cache_bytes; // total size of death row, bytes 
```

来看下large申请逻辑的流程图

![申请1.1MB内存](https://s2.loli.net/2022/01/02/dRTBt9AgSMYQPea.png)

1. 首先，**large_malloc** 函数会遍历 **large_entry_cache**。这是大小为64的环形数组，主要作用是存放已经释放的内存块并且提供给申请者使用。有关该环形数组的内容会在下一篇内存释放流程介绍中详细介绍
2. 如果设置了cache enable 并且申请的size <= large_cache_entry_limit。就可以进入cache逻辑了。limit取决于设备物理内存的大小，当物理内存大于32GB时，limit 为 521MB，否则为128MB。
3. 遍历large_entry_cache的逻辑非常巧妙：  
    1. 假设申请内存的大小为1.1MB，首先回去判断cache中刚好是否有1.1MB的内存，如果size刚好匹配，直接返回addr
    2. 如果没有size刚好相等的，就查看是否**1.1MB<=this_size**并且**this_size < best_size**。如果满足就会记下当前的best_size。这里设置best_size的目的是寻找一个大于等于1.1MB但是最小的内存块
    3. 如果找到了best_size。接下来还会判断best_size **是否大于size也就是1.1MB的2倍**，如果超过了2倍也不会给外部使用。
4. 如果在large_entry_cache 中没有拿到合适的addr，只能调用 **mvm_allocate_pages** 函数来申请新的内存页了。
5. 将mvm_allocate_pages返回的addr 封装为large_entry_t，存入szone的 hash表中


## 额外关注的逻辑
除去申请内存主流程之外，还有一些额外逻辑值得关注。
### libmalloc zone 的初始化
在第一次调用runtime_default_zone的时候，会去调用 inline_malloc_default_zone  函数初始化，最终会调用到 _malloc_initialize2() 函数进行初始化。函数太长就不贴了，想了解的话可以自己看源代码，这里只看主要的逻辑：
1. 获取逻辑CPU，根据CPU数设置magazine数
2. 初始化nano_zone ,调用 nano_common_configure()
3. 初始化helper_zone，并且设置各种参数信息。
4. 将initial_default_zone设置为hepler_zone
值得一提的是，调用下面的API，开发者可以初始化自己的zone来满足自己的需求。

```
your_zone = malloc_create_zone(0, 0);
malloc_set_zone_name(your_zone, "your_zone");
```
例如在腾讯的OOMDetector中，就初始化了两个zone "OOMDetector", "QQLeak"。他们分别负责OOM检测和Leak检测两个逻辑中所需要的内存分配，如创建Hash表等。这样做的好处是OOMDetector内部的内存使用行为不会被logger记录下来，也就不会影响到泄漏检测。

### MALLOC_TRACE
Libmalloc 源码中随处可见  MALLOC_TRACE(code,arg1,arg2,arg3,arg4)  宏，展开后为
```
if (malloc_traced()) { 
    kdebug_trace(code, arg1, arg2, arg3, arg4); 
}
```
其中**malloc_traced()** 函数很简单，会简单返回 tracing的开关 malloc_tracing_enabled，
这个开关会在初始化libmalloc的时候根据环境变量 *"MallocTracing"* 来决定开启或者关闭状态。

**kdebug_trace(code, arg1, arg2, arg3, arg4)**;  函数的具体实现在系统库libSystem里不得而知。从字面意义上来看是记录各种调试信息的。
此处不探究其内部实现，只看看libmalloc 是怎么使用Trace的。从上面的流程梳理不难猜出tiny，small，medium 三个分配主函数中的Trace是几乎一样的。事实也就是如此，所以抓两个典型，来看看tiny 和 large。
在tiny_malloc_should_clear 和 large_malloc 中，会在函数最开始调用TRACE。可以看到区别主要是code, arg1, arg2, arg3。

```
//tiny
MALLOC_TRACE(TRACE_tiny_malloc, (uintptr_t)rack, TINY_BYTES_FOR_MSIZE(msize), (uintptr_t)tiny_mag_ptr, cleared_requested);

//large
MALLOC_TRACE(TRACE_large_malloc, (uintptr_t)szone, num_kernel_pages, alignment, cleared_requested);
```

TRACE_tiny_malloc  和 TRACE_large_free  都是通过TRACE_CODE宏创建。本质上是一个静态常量，用来区分行为。简单罗列几个TRACE_CODE
```
// "external" trace points
TRACE_CODE(malloc, DBG_UMALLOC_EXTERNAL, 0x01);
TRACE_CODE(free, DBG_UMALLOC_EXTERNAL, 0x02);

// "internal" trace points
TRACE_CODE(nano_malloc, DBG_UMALLOC_INTERNAL, 0x1);
TRACE_CODE(tiny_malloc, DBG_UMALLOC_INTERNAL, 0x2);

TRACE_CODE(nano_free, DBG_UMALLOC_INTERNAL, 0x5);
TRACE_CODE(tiny_free, DBG_UMALLOC_INTERNAL, 0x6);

```
在tiny分配中，TRACE记录了申请内存的大小，，参与分配的rack和magazine。在large 分配中，TRACE记录了参与分配的zone和内核页。
从上可以看出，TRACE主要负责记录内存申请释放过程中的所有行为采集，为debug和各种内存调试功能提供支持。


### 向内核申请内存
在上面的申请过程中，如果depot_magazine 中获取heap region 的过程失败了，或者large 没有在cache中拿到合适的内存区域时，都会调用 mvm_allocate_pages 函数向内核申请更多的内存页
```
mvm_allocate_pages(size_t size, unsigned char align, uint32_t debug_flags, int vm_page_label)
```
mvm_allocate_pages 函数相当于一个libmalloc 和 内核沟通内存的接口。可以根据需要返回fresh_region 或者直接返回申请到的地址。其内部逻辑涉及了过多的Mach内核知识，这里就不再继续探究了。如果感兴趣可以自行查看源码。

## 总结一下
本篇文章通过申请4个size的内存探究了libmalloc的分配工作流程。了解了nano_zone, helper_zone 之间的互助关系，了解了无处不在的cache机制和libmalloc的并发机制，也大致知道了libmalloc是如何分配内存的。目前笔者对于libmalloc 的了解程度也仅仅算是 “登堂入室”，做不到“了如指掌”。倘若文中有什么错误纰漏，也望不吝赐教。

## 源码
最后，附上可以运行的libmalloc 库。该版本基于libmalloc-317.40.8 改造而来，去掉了一些无用了target，可以直接在MacOS 12 Montery 上运行调试。欢迎大家下载体验和star🌟🌟🌟
> 🌟 [libmalloc-317.40.8_runable](https://github.com/JethroYe/libmalloc-317.40.8_runable.git) 🌟
