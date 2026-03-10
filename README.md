# glibc-2.27 malloc源码中文注释
这里分析的是glibc-2.27的源码
# 结构体
## malloc_chunk
```c
struct malloc_chunk {
  
  INTERNAL_SIZE_T      mchunk_prev_size;  /* Size of previous chunk (if free).  */
  INTERNAL_SIZE_T      mchunk_size;       /* Size in bytes, including overhead. */
  
  struct malloc_chunk* fd;         /* double links -- used only if free. */
  struct malloc_chunk* bk;
  
  /* Only used for large blocks: pointer to next larger size.  */
  struct malloc_chunk* fd_nextsize; /* double links -- used only if free. */
  struct malloc_chunk* bk_nextsize;
};
```
该结构体是chunk的结构体，可以看到一个chunk存在六个变量，分别为`prev_size`、`size`、`fd`、`bk`、`fd_nextsize`、`bk_nextsize`，分别代表`上一个chunk的size大小`、`当前chunk的size大小`、`下一个chunk的指针`、`上一个chunk的指针`、`指向下一个大块的指针`、`指向上一个大块的指针`
## malloc_state
```c
struct malloc_state
{
	__libc_lock_define (, mutex);
	int flags;
	int have_fastchunks;
	mfastbinptr fastbinsY[NFASTBINS];
	mchunkptr top;
	mchunkptr last_remainder;
	mchunkptr bins[NBINS * 2 - 2];
	unsigned int binmap[BINMAPSIZE];
	struct malloc_state *next;
	struct malloc_state *next_free;
	INTERNAL_SIZE_T attached_threads;
	INTERNAL_SIZE_T system_mem;
	INTERNAL_SIZE_T max_system_mem;
};
```
# 重要函数
### libc_free
#### 函数结构
```
+----------------------__libc_free----------------------+
|                       free_hook                       |
|---------------------------+---------------------------|
|    调用free_hook()         |        继续程序            |
+-------------------------------------------------------+
|-------------------------继续程序-----------------------|
|       由mmap分配内存        |           不是            |
|      一堆奇奇怪怪的检测      |      调用__int_free()     |
|       释放该内存区域         |                          |
+----------------------------+--------------------------+
```
#### 函数分析
##### 变量
- 函数定义了如下的变量
```c
mstate ar_ptr;              //堆分配器
mchunkptr p;                //即要释放的chunk的指针
```
##### 函数片段
- free_hook
	```c
	void (*hook) (void *, const void *)       //启用了__free_hook               
	    = atomic_forced_read (__free_hook);
	if (__builtin_expect (hook != NULL, 0))   //__free_hook不为Null
	    {
	      (*hook)(mem, RETURN_ADDRESS (0));   //执行__free_hook()       
	      return;
	    }
	```
	该段代码为free中的`free_hook`相关代码
	- 当`free_hook`为`Null`时，他不会执行`free_hook()`操作，而是接下来去执行free的操作
	- 当`free_hook`不为Null时，执行`free_hook()`操作
- free
	```c
		//如果free(0)，那么不会有任何作用
	    if (mem == 0)                              /* free(0) has no effect */  
		    return;
		p = mem2chunk (mem);        //将对应的堆内存地址转为对应的chunk
		if (chunk_is_mmapped (p))   //检测是否由mmap分配的内存
	    {
	      /* See if the dynamic brk/mmap threshold needs adjusting.
	   Dumped fake mmapped chunks do not affect the threshold.  */
		    //一堆奇奇怪怪的检测
		    if (!mp_.no_dyn_threshold
		        && chunksize_nomask (p) > mp_.mmap_threshold
		        && chunksize_nomask (p) <= DEFAULT_MMAP_THRESHOLD_MAX
		&& !DUMPED_MAIN_ARENA_CHUNK (p))
		    {
		        mp_.mmap_threshold = chunksize (p);
		        mp_.trim_threshold = 2 * mp_.mmap_threshold;
		        LIBC_PROBE (memory_mallopt_free_dyn_thresholds, 2,
	                    mp_.mmap_threshold, mp_.trim_threshold);
	        }
		  munmap_chunk (p);
	      return;
	    }

	  MAYBE_INIT_TCACHE ();
	
	  ar_ptr = arena_for_chunk (p);  //找到p对应的堆分配器
	  _int_free (ar_ptr, p, 0);      //调用free的顶层函数进行free操作
	```
	可以看到
	- 当free的传参为0时，`libc_free`函数不会有任何操作
	- 然后会对对应chunk分配的内存进行判断，如果是mmap分配的，会进行一系列奇奇怪怪的检测，然后直接释放掉对应内存
	- 如果不是则会进入`int_free`函数进行更深层次的free调用
### int_free
#### 函数结构
```
+-----------------------__int_free-----------------------+
|                        一系列检测                       |
+--------------------------------------------------------+
+------------------------tcache 部分----------------------+
|                        一系列检测                        |
|                        存入tcache                       |
|                           结束                          |
+--------------------------------------------------------+
+------------------------fastbin 部分---------------------+
|                        一系列检测                        |
|----------------------------+---------------------------|
|          单线程             |           多线程           |
|----------------------------+---------------------------|
|       double free检测       |        double free检测    |
|----------------------------|---------------------------|
|        头插入链表            |          一系列检测        |
|--------------------------------------------------------|
|                            结束                         |
+--------------------------------------------------------+
+---------------------unsorted bin 部分-------------------+
|          单线程              |            多线程         |
|-----------------------------+--------------------------|
|                             |             上锁          |
|-----------------------------+--------------------------|
|                         一大堆检测                       |
|--------------------------------------------------------|
|                       unlink 向前合并                   |
|--------------------------------------------------------|
|                       unlink 向后合并                   |
|--------------------------------------------------------|
|                          一系列检测                      |
|--------------------------------------------------------|
|                      在large bin范围内                   |
|---------------------------------------------------------|
|                             入链                         |
+---------------------------------------------------------+
+-------------------------top chunk范围--------------------+
|                             合并                         |
+---------------------------------------------------------+
+------------------------内存处理机制-----------------------+
|                    fastbin 碎片合并机制                   |
|---------------------------------------------------------|
|                         其他处理机制                      |
+---------------------------------------------------------+
```
#### 函数分析
##### 变量
```c
  INTERNAL_SIZE_T size;        //chunk的size
  mfastbinptr *fb;             //fastbin
  mchunkptr nextchunk;         //当前chunk的下一个chunk
  INTERNAL_SIZE_T nextsize;    //nextchunk的size
  int nextinuse;               //nextchunk的inuse位
  INTERNAL_SIZE_T prevsize;    //当前chunk的上一个chunk
  mchunkptr bck;               //chunk，在双向链表中使用
  mchunkptr fwd;               //chunk，在双向链表中使用
```
##### 函数片段
- 相关检测
	```c
	  size = chunksize (p);                     //去当前chunk的size值

	  /* Little security check which won't hurt performance: the
	     allocator never wrapps around at the end of the address space.
	     Therefore we can exclude some size values which might appear
	     here by accident or by "design" from some intruder.  */
	  if (__builtin_expect ((uintptr_t) p > (uintptr_t) -size, 0)                      //判断指针是否合法
	      || __builtin_expect (misaligned_chunk (p), 0))
	    malloc_printerr ("free(): invalid pointer");
	  /* We know that each chunk is at least MINSIZE bytes in size or a
	     multiple of MALLOC_ALIGNMENT.  */
	  if (__glibc_unlikely (size < MINSIZE || !aligned_OK (size)))                     //检查free的chunk的大小是否合法，且是否16字节对齐
	    malloc_printerr ("free(): invalid size");
	
	  check_inuse_chunk(av, p);                                                        //检查inuse标志位
	```
- tcache 部分
	```c
	//tcache 部分
	#if USE_TCACHE
	  {
	    size_t tc_idx = csize2tidx (size);
	  
	    if (tcache
	  && tc_idx < mp_.tcache_bins
	  && tcache->counts[tc_idx] < mp_.tcache_count)
	      {
	  tcache_put (p, tc_idx);                                                          //存入tcache
	  return;                                                                          //直接结束
	      }
	  }
	#endif
	```
- fastbin 部分
	```c
	  // fastbin 部分
	  if ((unsigned long)(size) <= (unsigned long)(get_max_fast ())                    //size在fastbin的范围内
	  
	#if TRIM_FASTBINS
	      /*
	  If TRIM_FASTBINS set, don't place chunks
	  bordering top into fastbins
	      */
	      && (chunk_at_offset(p, size) != av->top)  //相邻块不是top chunk
	#endif
	      ) {
	  
	    if (__builtin_expect (chunksize_nomask (chunk_at_offset (p, size))
	        <= 2 * SIZE_SZ, 0)     //堆块尺寸过小
	  || __builtin_expect (chunksize (chunk_at_offset (p, size))
	           >	= av->system_mem, 0))   //堆块尺寸过大
		      {
		  bool fail = true;
		  /* We might not have a lock at this point and concurrent modifications
		     of system_mem might result in a false positive.  Redo the test after
		     getting the lock.  */
		  if (!have_lock)      //没有锁
		    {
		      __libc_lock_lock (av->mutex);    //上锁
		      fail = (chunksize_nomask (chunk_at_offset (p, size)) <= 2 * SIZE_SZ          //堆块尺寸过小
		        || chunksize (chunk_at_offset (p, size)) >= av->system_mem);               //堆块尺寸过大
		      __libc_lock_unlock (av->mutex);   //解锁
		    }
		
		  if (fail)    //尺寸不对
		    malloc_printerr ("free(): invalid next size (fast)");
		      }
		
		    free_perturb (chunk2mem(p), size - 2 * SIZE_SZ);  //填充扰动值
		
		    atomic_store_relaxed (&av->have_fastchunks, true);                             //标记存在fastbin
		    unsigned int idx = fastbin_index(size);     //获取fastbin的idx
		    fb = &fastbin (av, idx);      //获取对应地址处的fastbin
		
		    /* Atomically link P to its fastbin: P->FD = *FB; *FB = P;  */
		    mchunkptr old = *fb, old2;
		
		    if (SINGLE_THREAD_P)       //单线程
		      {
		  /* Check that the top of the bin is not the record we are going to
		     add (i.e., double free).  */
		  if (__builtin_expect (old == p, 0))   //double free检测
		    malloc_printerr ("double free or corruption (fasttop)");
		  p->fd = old;      //入链表
		  *fb = p;
		      }
		    else
		      do   //多线程
		  {
		    /* Check that the top of the bin is not the record we are going to
		       add (i.e., double free).  */
		    if (__builtin_expect (old == p, 0))  //依旧是double free检测
		      malloc_printerr ("double free or corruption (fasttop)");
		    p->fd = old2 = old;
		  }
		      while ((old = catomic_compare_and_exchange_val_rel (fb, p, old2))
		       != old2);
		
		    /* Check that size of fastbin chunk at the top is the same as
		       size of the chunk that we are adding.  We can dereference OLD
		       only if we have the lock, otherwise it might have already been
		       allocated again.  */
		    if (have_lock && old != NULL  //有锁 && 存在被释放过的fastbin
		  && __builtin_expect (fastbin_index (chunksize (old)) != idx, 0))                 //尺寸合法性校验
		      malloc_printerr ("invalid fastbin entry (free)");
		  }
	```
- unsorted bin 部分
	```c
	  //unsortedbin|largebin|smallbin部分
	  //其中largebin和smallbin是在unsortedbin的处理逻辑中进行处理的
	  else if (!chunk_is_mmapped(p)) {   //不是mmap分配的大块内存
	  
	    /* If we're single-threaded, don't lock the arena.  */
	    if (SINGLE_THREAD_P)   //单线程
	      have_lock = true;     //上锁
	  
	    if (!have_lock)
	      __libc_lock_lock (av->mutex);  //对多线程进行上锁
	  
	    nextchunk = chunk_at_offset(p, size); //下一个chunk的地址
	  
	    /* Lightweight tests: check whether the block is already the
	       top block.  */
	    if (__glibc_unlikely (p == av->top))                                           //double free 或者 free的是top chunk
	      malloc_printerr ("double free or corruption (top)");
	    /* Or whether the next chunk is beyond the boundaries of the arena.  */
	    if (__builtin_expect (contiguous (av)  //堆内存地址连续
	        && (char *) nextchunk              //地址越界
	        >	= ((char *) av->top + chunksize(av->top)), 0))
		  malloc_printerr ("double free or corruption (out)");                             //地址越界错误
		    /* Or whether the block is actually not marked used.  */
		    if (__glibc_unlikely (!prev_inuse(nextchunk)))                                 //放double free或者堆结构篡改
		      malloc_printerr ("double free or corruption (!prev)");
		
		    nextsize = chunksize(nextchunk);  //下一个chunk的size
		    if (__builtin_expect (chunksize_nomask (nextchunk) <= 2 * SIZE_SZ, 0)          //堆块过于小
		  || __builtin_expect (nextsize >= av->system_mem, 0)) //堆块过于大
		      malloc_printerr ("free(): invalid next size (normal)");
		
		    free_perturb (chunk2mem(p), size - 2 * SIZE_SZ); //填充扰动值
		
		    /* consolidate backward */
		    //unlink 向前合并
		    if (!prev_inuse(p)) {  //前一个堆块空闲
		      prevsize = prev_size (p);   //前一个堆块的size
		      size += prevsize;       //合并后的size
		      p = chunk_at_offset(p, -((long) prevsize)); //前一个堆块的起始地址
		      unlink(av, p, bck, fwd);    //unlink操作
		    }
		
		    if (nextchunk != av->top) {    //下一个chunk是top chunk
		      /* get and clear inuse bit */
		      nextinuse = inuse_bit_at_offset(nextchunk, nextsize);                        //获取下一个chunk的inuse位
		
		      /* consolidate forward */
		      if (!nextinuse) {    //下一个chunk空闲
		  unlink(av, nextchunk, bck, fwd);   //向后合并
		  size += nextsize;      //合并后的size
		      } else
		  clear_inuse_bit_at_offset(nextchunk, 0); //将inuse标志位记为0
		
		      /*
		  Place the chunk in unsorted chunk list. Chunks are
		  not placed into regular bins until after they have
		  been given one chance to be used in malloc.
		      */
		
		      bck = unsorted_chunks(av); //取一个unsorted bin
		      fwd = bck->fd;     //下一个chunk
		      if (__glibc_unlikely (fwd->bk != bck))  //检测是否是同一个chunk
		  malloc_printerr ("free(): corrupted unsorted chunks");
		      p->fd = fwd;      //入链
		      p->bk = bck;      //入链
		      if (!in_smallbin_range(size))   //large bin
		  {
		    p->fd_nextsize = NULL;
		    p->bk_nextsize = NULL;
		  }
		      bck->fd = p;        //入链
		      fwd->bk = p;       //入链
		
		      set_head(p, size | PREV_INUSE);  //设置头
		      set_foot(p, size);              //设置尾
		
		      check_free_chunk(av, p);
		    }
	```
	- 程序会对多进程进行上锁操作
	- 然后有一系列的检测机制
	- 然后unlink向前合并
	- 然后unlink向后合并
	- 然后进行入链
- topchunk 部分
	```c
	//topchunk 部分
	    else {                                                                         //top chunk
	      size += nextsize;                                                            //合并后的chunk
	      set_head(p, size | PREV_INUSE);                                              //设置头
	      av->top = p;                                                                 //修改top chunk的起始指针
	      check_chunk(av, p);
	    }
	```
	- 对于topchunk部分进行合并
- 其他内存处理部分
	```c
		//其他的内存处理之类的部分
	    //fastbin碎片合并机制
	    if ((unsigned long)(size) >= FASTBIN_CONSOLIDATION_THRESHOLD) {
	      if (atomic_load_relaxed (&av->have_fastchunks))
	  malloc_consolidate(av);
	
	      if (av == &main_arena) {   //是main_arena
	#ifndef MORECORE_CANNOT_TRIM
	  if ((unsigned long)(chunksize(av->top)) >=
	      (unsigned long)(mp_.trim_threshold))
	    systrim(mp_.top_pad, av);  //裁剪主堆的空闲内存，归还给操作系统
	#endif
	      } else {         //非main_arena
	  /* Always try heap_trim(), even if the top chunk is not
	     large, because the corresponding heap might go away.  */
	  heap_info *heap = heap_for_ptr(top(av));  //找到顶块所属的子堆结构
	
	  assert(heap->ar_ptr == av);
	  heap_trim(heap, mp_.top_pad);
	      }
	    }
	
	    if (!have_lock)
	      __libc_lock_unlock (av->mutex);    //多线程解锁
	  }
	  /*
	    If the chunk was allocated via mmap, release via munmap().
	  */
	
	  else {
	    munmap_chunk (p);         //mmap块释放
	  }
	```
	- 首先对fastbin中碎片化的部分进行合并
	- 随后进行一些其他的机制
### libc_malloc
#### 函数结构
```
+-------------------------libc_malloc-----------------------+
|                         malloc_hook                       |
+-----------------------------------------------------------+
|                        若启用了tcache                      |
|-----------------------------------------------------------|
|       tcache为空的            |          tcache不为空       |
|------------------------------|----------------------------|
|          初始化tcache         |      从tcache中取chunk      |
|-----------------------------------------------------------|
|            单线程             |           多线程            |
|-----------------------------------------------------------|
|                              |          寻找arena          |
|-----------------------------------------------------------|
|                 调用malloc的底层函数int_malloc              |
|-----------------------------------------------------------|
|                      返回分配的chunk的指针                   |
+------------------------------------------------------------+          
```
#### 函数分析
##### 变量
```c
  mstate ar_ptr;     //堆分配器
  void *victim;      //选中的chunk
```
##### 代码片段
- malloc_hook
	```c
	void *(*hook) (size_t, const void *)
	    = atomic_forced_read (__malloc_hook);                          //启用了malloc_hook，因此可以劫持__malloc_hook，从而实现任意执行
	  if (__builtin_expect (hook != NULL, 0))
	    return (*hook)(bytes, RETURN_ADDRESS (0));                     //调用malloc_hook
	```
- 剩余部分
	```c
	#if USE_TCACHE                                                     //启用了tcache
	  /* int_free also calls request2size, be careful to not pad twice.  */
	  size_t tbytes;
	  checked_request2size (bytes, tbytes);
	  size_t tc_idx = csize2tidx (tbytes);
	  
	  MAYBE_INIT_TCACHE ();
	  
	  DIAG_PUSH_NEEDS_COMMENT;
	  if (tc_idx < mp_.tcache_bins
	      /*&& tc_idx < TCACHE_MAX_BINS*/ /* to appease gcc */
	      && tcache
	      && tcache->entries[tc_idx] != NULL)
	    {
	      return tcache_get (tc_idx);
	    }
	  DIAG_POP_NEEDS_COMMENT;
	#endif
	  
	  if (SINGLE_THREAD_P)          //单线程
	    {
	      victim = _int_malloc (&main_arena, bytes);
	      assert (!victim || chunk_is_mmapped (mem2chunk (victim)) ||
	        &main_arena == arena_for_chunk (mem2chunk (victim)));
	      return victim;
	    }
	  
	  arena_get (ar_ptr, bytes);                                       //寻找arena试图分配内存
	  
	  victim = _int_malloc (ar_ptr, bytes);                            //调用核心函数去申请内存
	  /* Retry with another arena only if we were able to find a usable arena
	     before.  */
	  if (!victim && ar_ptr != NULL)                                   //如果分配失败，会再去尝试其他的arena
	    {
	      LIBC_PROBE (memory_malloc_retry, 1, bytes);
	      ar_ptr = arena_get_retry (ar_ptr, bytes);
	      victim = _int_malloc (ar_ptr, bytes);
	    }
	  
	  if (ar_ptr != NULL)                                              //如果申请到了arena那么退出之前会解锁试
	    __libc_lock_unlock (ar_ptr->mutex);
	  
	  assert (!victim || chunk_is_mmapped (mem2chunk (victim)) ||
	          ar_ptr == arena_for_chunk (mem2chunk (victim)));
	  return victim;                                                   //判断目前的状态，要么没有申请到内存，要么是mmap了内存，要么申请到的内存分配在arena中
	```
### int_malloc
#### 函数结构
```
+--------------------------__int_malloc-------------------------+
|                        进入分配核心逻辑                         |
|---------------------------------------------------------------|
|                         检测 arena 可用性                       |
|---------------------------------------------------------------|
|----------------------------Fastbin 部分 -----------------------|
|                          一系列基础检查                         |
|---------------------------------------------------------------|
|       若命中大小        |            启用了tcache               |
|-----------------------|---------------------------------------|
|  从 Fastbin 头部取出一块 |  若对应 Tcache 没满，则将该 bin 链中    |
|  作为返回结果           |  剩余同大小 chunk 全部“搬运”进 Tcache   |
|-----------------------+---------------------------------------|
|-------------------------- Smallbin 部分 -----------------------|
|                          一系列基础检查                         |
|---------------------------------------------------------------|
|       若命中大小        |             启用了tcache              |
|-----------------------|---------------------------------------|
|  从 Smallbin 尾部取出一块|  若对应 Tcache 没满，则将该 bin 链中    |
|  作为返回结果           |  剩余同大小 chunk 全部“搬运”进 Tcache   |
|-----------------------+---------------------------------------|
|------------------- Unsorted Bin / Consolidation --------------|
|        若申请大小在 Large Bin 范围内，先调用 malloc_consolidate   |
|                           整合碎片化chunk                       |
|---------------------------------------------------------------|
|                     遍历 Unsorted Bin (核心循环)               |
|---------------------------------------------------------------|
|   情况 A: 大小精确命中  |   情况 B: 大小不匹配  |   情况 C: 满足分割 |
|-----------------------|---------------------|-----------------|
| [2.27 特性] 先丢进      |  根据大小将其“归位”到 |  若是在 Unsorted   |
| Tcache；若 Tcache 满了 |  Small Bin 或        |  末尾且满足 Last   |
| 才直接返回该 chunk      |  Large Bin          |  Remainder 则分割  |
|-----------------------+---------------------+-----------------|
|-------------------------- Large Bin 部分 ----------------------|
|         若 Unsorted Bin 找了一圈没合适的，才扫描 Large Bin         |
|         采用“最小适配”原则，找到最接近且大于申请大小的 chunk          |
|---------------------------------------------------------------|
|           分割剩余部分放入 Unsorted Bin，返回选中的块              |
|---------------------------------------------------------------|
|--------------------------- Top Chunk 部分 ---------------------|
|                       计算 Top Chunk 剩余空间                  |
|---------------------------------------------------------------|
|    剩余空间足够        |                空间不足                |
|----------------------|---------------------------------------|
|  直接从 Top Chunk 分割 |  1. 尝试合并 Fastbin 碎片 (Consolidate)|
|  并更新 Top 指针       |  2. 依然不足？调用 sysmalloc 扩容      |
|                      |  3. 扩容后再次尝试分配                  |
|----------------------+---------------------------------------|
```
#### 函数分析
##### 变量
```c
  INTERNAL_SIZE_T nb;            //标准化请求大小
  unsigned int idx;              //
  mbinptr bin;                   //关联的bin索引
  
  mchunkptr victim;              //选定的块         
  INTERNAL_SIZE_T size;          //victim的size 
  int victim_index;              //victim的bin的索引
  
  mchunkptr remainder;           //分割后的剩余部分   
  unsigned long remainder_size;  //剩余部分的大小   
  
  unsigned int block;            //位图遍历器   
  unsigned int bit;              //位图遍历器
  unsigned int map;              //  
  
  mchunkptr fwd;                 //前一个块  
  mchunkptr bck;                 //后一个块  
```
##### 代码片段
- 前置操作
	```c
	  #if USE_TCACHE
	  size_t tcache_unsorted_count;     /* count of unsorted chunks processed */
	#endif
	  
	  /*
	     Convert request size to internal form by adding SIZE_SZ bytes
	     overhead plus possibly more to obtain necessary alignment and/or
	     to obtain a size of at least MINSIZE, the smallest allocatable
	     size. Also, checked_request2size traps (returning 0) request sizes
	     that are so large that they wrap around zero when padded and
	     aligned.
	   */
	  
	  checked_request2size (bytes, nb);      //将用户申请的chunksize（即nb）转化成chunk实际的size，存入bytes函数      
	                                         //这里存在漏洞点，如果我申请了大小为-1（0xFFFFFFFFFFFFFFF0），那么经过转化会返回一个很小的值，这样系统会给你分配一个极小的chunk，然后就可以进行堆溢出操作
	  
	  /* There are no usable arenas.  Fall back to sysmalloc to get a chunk from
	     mmap.  */
	  if (__glibc_unlikely (av == NULL))           //检测arena是否可用
	    {
	      void *p = sysmalloc (nb, av);
	      if (p != NULL)
	  alloc_perturb (p, bytes);               //填充垃圾数据
	      return p;
	    }
	```
	- 这部分代码对`__int_malloc` 函数进行了一系列的前置操作，为后面继续执行程序做好准备
	- 这里的 `checked_request2size(bytes, nb)` 缺乏检测，如果申请了一个 `0xFFFFFFFFFFFFFFF0(-1)` 大小的chunk，那么经过转化会变成一个很小的值，这样系统就会给你分配一个很小的chunk，然后就可以进行堆溢出操作
	> PS: 此问题在glibc-2.30得到了解决，在这个版本的libc代码中新加入了检测，对于大小不合规的chunk，会直接返回Null
- fastbin 部分
	```c
	  #define REMOVE_FB(fb, victim, pp)     \
	  do              \
	    {             \
	      victim = pp;          \
	      if (victim == NULL)       \
	  break;            \
	    }             \
	  while ((pp = catomic_compare_and_exchange_val_acq (fb, victim->fd, victim)) \
	   != victim);          \          //无锁弹出，但是缺乏完整性校验，可以通过伪造chunk来实现任意的malloc，从而实现任意地址写
	  
	  if ((unsigned long) (nb) <= (unsigned long) (get_max_fast ()))   //在fastbin范围内
	    {
	      idx = fastbin_index (nb);                //获取对应nb大小的fastbin的idx
	      mfastbinptr *fb = &fastbin (av, idx);    //取一个fastbin
	      mchunkptr pp;
	      victim = *fb;
	  
	      if (victim != NULL)
	  {
	    if (SINGLE_THREAD_P)    //单线程
	      *fb = victim->fd;
	    else                    //多线程
	      REMOVE_FB (fb, pp, victim);
	    if (__glibc_likely (victim != NULL))
	      {
	        size_t victim_idx = fastbin_index (chunksize (victim));
	        if (__builtin_expect (victim_idx != idx, 0))
	    malloc_printerr ("malloc(): memory corruption (fast)");
	        check_remalloced_chunk (av, victim, nb);
	#if USE_TCACHE                                       //将fastbin中剩余的同大小chunk分配进tcache
	        /* While we're here, if we see other chunks of the same size,
	     stash them in the tcache.  */
	        size_t tc_idx = csize2tidx (nb);
	        if (tcache && tc_idx < mp_.tcache_bins)
	    {
	      mchunkptr tc_victim;
	  
	      /* While bin not empty and tcache not full, copy chunks.  */
	      while (tcache->counts[tc_idx] < mp_.tcache_count
	       && (tc_victim = *fb) != NULL)
	        {
	          if (SINGLE_THREAD_P)
	      *fb = tc_victim->fd;
	          else
	      {
	        REMOVE_FB (fb, pp, tc_victim);    //链表头部取chunk，即用于从指定的fastbin中在链表头部取chunk
	        if (__glibc_unlikely (tc_victim == NULL))
	          break;
	      }
	          tcache_put (tc_victim, tc_idx);              //将剩余的同大小chunk放入tcache
	        }
	    }
	#endif                                                 //在内存中取一块地址分配给新的chunk
	        void *p = chunk2mem (victim);
	        alloc_perturb (p, bytes);
	        return p;
	      }
	  }
	    }
	```
	- 首先该部分定义了一个宏 `REMOVE_FB(fb, victim, pp)` 用于从fastbin中移除内存块
	- 这里存在这一个和tcache相关的重要操作，当在fastbin中取了一个chunk之后，程序会对tcache进行检测，若tcahce不满，则会从剩余fastbin中取大小相同的chunk，直至取完或者tcache填满
	- 如果命中了Fastbin的范围，那么进行检查有无可用的Fastbin，如果可用，那么初始化相关变量，先检查是否对齐，再进行对Fastbin的取出操作，取出后检查取出的Fastbin的idx是否前后一致，打乱后返回。
- smallbin 部分
	```c
	  //从small bins中分配
	  if (in_smallbin_range (nb))                            //判断是否在small bins范围之内
	    {
	      idx = smallbin_index (nb);                         //取smallbins的idx
	      bin = bin_at (av, idx);                            //取第idxsmallbin的起始地址
	  
	      if ((victim = last (bin)) != bin)
	        {
	          bck = victim->bk;                               //victim连接的上一个chunk
	    if (__glibc_unlikely (bck->fd != victim))             //校验双向链表的完整性，防止unlink操作
	      malloc_printerr ("malloc(): smallbin double linked list corrupted");
	          set_inuse_bit_at_offset (victim, nb);           //设置inuse位标志
	          //脱链操作
	          bin->bk = bck;                                  
	          bck->fd = bin;
	  
	          if (av != &main_arena)
	      set_non_main_arena (victim);
	          check_malloced_chunk (av, victim, nb);
	#if USE_TCACHE
	    /* While we're here, if we see other chunks of the same size,
	       stash them in the tcache.  */
	    size_t tc_idx = csize2tidx (nb);                       //与fastbin中的代码目的一致，将相同大小的small bins中的chunk给放入tcache中
	    if (tcache && tc_idx < mp_.tcache_bins)
	      {
	        mchunkptr tc_victim;
	  
	        /* While bin not empty and tcache not full, copy chunks over.  */
	        while (tcache->counts[tc_idx] < mp_.tcache_count
	         && (tc_victim = last (bin)) != bin)
	    {
	      if (tc_victim != 0)
	        {
	          bck = tc_victim->bk;
	          set_inuse_bit_at_offset (tc_victim, nb);
	          if (av != &main_arena)
	      set_non_main_arena (tc_victim);
	          bin->bk = bck;
	          bck->fd = bin;
	  
	          tcache_put (tc_victim, tc_idx);
	              }
	    }
	      }
	#endif
	          void *p = chunk2mem (victim);
	          alloc_perturb (p, bytes);
	          return p;
	        }
	    }
	```
	- 对于smallbin范围内的chunk，程序会对其进行完整性检测，从而防止unlink操作，随后会执行和fastbin中一样的tcache操作，最后返回取出最后一个可用的内存块作为分配的内存块
- Unsorted Bin / Consolidation 部分
	```c
	  else
	    {
	      idx = largebin_index (nb);
	      if (atomic_load_relaxed (&av->have_fastchunks))
	        malloc_consolidate (av);
	    }
	  
	  /*
	     Process recently freed or remaindered chunks, taking one only if
	     it is exact fit, or, if this a small request, the chunk is remainder from
	     the most recent non-exact fit.  Place other traversed chunks in
	     bins.  Note that this step is the only place in any routine where
	     chunks are placed in bins.
	  
	     The outer loop here is needed because we might not realize until
	     near the end of malloc that we should have consolidated, so must
	     do so and retry. This happens at most once, and only when we would
	     otherwise need to expand memory to service a "small" request.
	   */
	  
	#if USE_TCACHE
	  INTERNAL_SIZE_T tcache_nb = 0;
	  size_t tc_idx = csize2tidx (nb);              //获取nb的tcache的idx
	  if (tcache && tc_idx < mp_.tcache_bins)
	    tcache_nb = nb;
	  int return_cached = 0;
	  
	  tcache_unsorted_count = 0;
	#endif
	  
	  /*
	  在UnsortedBin大遍历后，如果找到了对应的chunk，会返回，如果没有找到，依旧会把UnsortedBin中所有的chunk放入chunk对应的bin中。
	  */
	  for (;; )
	    {
	      int iters = 0;
	      while ((victim = unsorted_chunks (av)->bk) != unsorted_chunks (av))            //如果unsotred bins不为空，进入unsorted bins去处理一些数据
	        {
	          bck = victim->bk;                                                          //victim连接的上一个chunk，即unsorted bins中最倒数第二个chunk
	          if (__builtin_expect (chunksize_nomask (victim) <= 2 * SIZE_SZ, 0)        
	              || __builtin_expect (chunksize_nomask (victim)
	           > 	av->system_mem, 0))
		            malloc_printerr ("malloc(): memory corruption");
		          size = chunksize (victim);                                                 //选取的chunk的size
		          /*
		             If a small request, try to use last remainder if it is the
		             only chunk in unsorted bin.  This helps promote locality for
		             runs of consecutive small requests. This is the only
		             exception to best-fit, and applies only when there is
		             no exact fit for a small chunk.
		           */
		          // last remainder 分割部分
		          if (in_smallbin_range (nb) &&           //用户申请的大小命中smallbin
		              bck == unsorted_chunks (av) &&      //victim是unsorted bins中的chunk
		              victim == av->last_remainder &&     //检查当前内存块中是否存在最近分配内存块的剩余
		              //检查victim是否可以被分割
		              (unsigned long) (size) > (unsigned long) (nb + MINSIZE))
		            {
		              /* split and reattach remainder */
		              remainder_size = size - nb;                                             //分割后的大小
		              remainder = chunk_at_offset (victim, nb);                               //分割small request chunk
		              unsorted_chunks (av)->bk = unsorted_chunks (av)->fd = remainder;        //将分割出来的块纳入链表
		              av->last_remainder = remainder;                                         //更新last_remainder
		              remainder->bk = remainder->fd = unsorted_chunks (av);                   //维护remainder的链表	
		              if (!in_smallbin_range (remainder_size))                                //如果分割后的remainder不在smallbin中，则维护其在largebin的链表
		                {
		                  remainder->fd_nextsize = NULL;
		                  remainder->bk_nextsize = NULL;
		                }
		              //设置chunk head
		              set_head (victim, nb | PREV_INUSE |
		                        (av != &main_arena ? NON_MAIN_ARENA : 0));
		              set_head (remainder, remainder_size | PREV_INUSE);
		              set_foot (remainder, remainder_size);
		
		              check_malloced_chunk (av, victim, nb);
		              void *p = chunk2mem (victim);
		              alloc_perturb (p, bytes);
		              return p;
		            }
		          //移除victim
		          /* remove from unsorted list */
		          unsorted_chunks (av)->bk = bck;
		          bck->fd = unsorted_chunks (av);
		
		          /* Take now instead of binning if exact fit */
		          //大小恰好合适，直接取出victim
		          if (size == nb)
		            {
		              set_inuse_bit_at_offset (victim, size);
		              if (av != &main_arena)
		    set_non_main_arena (victim);
		#if USE_TCACHE
		        /* Fill cache first, return to user only if cache fills.
		     We may return one of these chunks later.  */
		        if (tcache_nb                                                      //tcache启用
		      && tcache->counts[tc_idx] < mp_.tcache_count)                        //tcache链表未满
		    {
		      tcache_put (victim, tc_idx);                                         //将victim放入tcache
		      return_cached = 1;
		      continue;
		    }
		        else                                                               //tcache未启用或tcache链表已满，直接分配victim
		    {
		#endif
		              check_malloced_chunk (av, victim, nb);
		              void *p = chunk2mem (victim);
		              alloc_perturb (p, bytes);
		              return p;
		#if USE_TCACHE
		    }
		#endif
		            }
		
		          /* place chunk in bin */
		
		          if (in_smallbin_range (size))                              //如果size命中smallbin
		            {
		              victim_index = smallbin_index (size);
		              bck = bin_at (av, victim_index);
		              fwd = bck->fd;
		            }
		          else                                                       //如果size命中largebin
		            {
		              victim_index = largebin_index (size);
		              bck = bin_at (av, victim_index);
		              fwd = bck->fd;
		
		              /* maintain large bins in sorted order */
		              if (fwd != bck)                                         //large bins不为空
		                {
		                  /* Or with inuse bit to speed comparisons */
		                  size |= PREV_INUSE;
		                  /* if smaller than smallest, bypass loop below */
		                  //bck->bk 存储着相应 large bin 中最小的chunk。
		                  //如果遍历的chunk比当前最小的chunk还要小，直接插入尾部
		                  //判断是否在main_arena
		                  assert (chunk_main_arena (bck->bk));
		                  if ((unsigned long) (size)
		          < (unsigned long) chunksize_nomask (bck->bk))
		                    {
		                      fwd = bck;                                                 //令fwd指向largebin头部
		                      bck = bck->bk;                                             //令 bck 指向 largin bin 尾部 chunk
		                      //插入victim到链表尾部
		                      victim->fd_nextsize = fwd->fd;                           
		                      victim->bk_nextsize = fwd->fd->bk_nextsize;
		                      fwd->fd->bk_nextsize = victim->bk_nextsize->fd_nextsize = victim;
		                    }
		                  else                                                           //当前要插入的 victim 的大小大于最小的 chunk
		                    {
		                      assert (chunk_main_arena (fwd));
		                      //从链表头部开始找到不比 victim 大的 chunk
		                      while ((unsigned long) size < chunksize_nomask (fwd))
		                        {
		                          fwd = fwd->fd_nextsize;
		        assert (chunk_main_arena (fwd));
		                        }
		
		                      if ((unsigned long) size                                   //如果找到了一个和 victim 一样大的 chunk，
		                      //那就直接将 chunk 插入到该chunk的后面，并不修改 nextsize 指针。
		        == (unsigned long) chunksize_nomask (fwd))
		                        /* Always insert in the second position.  */
		                        fwd = fwd->fd;
		                      else                                                       //如果找到的chunk和当前victim大小不一样
		                        {
		                          //那么就需要构造 nextsize 双向链表了
		                          victim->fd_nextsize = fwd;
		                          victim->bk_nextsize = fwd->bk_nextsize;
		                          fwd->bk_nextsize = victim;
		                          victim->bk_nextsize->fd_nextsize = victim;
		                        }
		                      bck = fwd->bk;
		                    }
		                }
		              else                                                               //如果空的话，直接简单使得 fd_nextsize 与 bk_nextsize 构成一个双向链表即可。
		                victim->fd_nextsize = victim->bk_nextsize = victim;
		            }
		          //构成bck <-> victim <-> fwd
		          mark_bin (av, victim_index);
		          victim->bk = bck;
		          victim->fd = fwd;
		          fwd->bk = victim;
		          bck->fd = victim;
		
		#if USE_TCACHE
		      /* If we've processed as many chunks as we're allowed while
		   filling the cache, return one of the cached ones.  */
		      ++tcache_unsorted_count;
		      if (return_cached
		    && mp_.tcache_unsorted_limit > 0
		    && tcache_unsorted_count > mp_.tcache_unsorted_limit)
		  {
		    return tcache_get (tc_idx);
		  }
		#endif
		
		#define MAX_ITERS       10000
		          if (++iters >= MAX_ITERS)
		            break;
		        }
		
		#if USE_TCACHE
		      /* If all the small chunks we found ended up cached, return one now.  */
		      if (return_cached)
		  {
		    return tcache_get (tc_idx);
		  }
		#endif
	```
	- 首先程序会判断chunk是否在large bin中，如果在整合碎片化的chunk
	- 然后程序会遍历整个unsorted bin，主要有3个操作
		- last remainder 分割
			当满足 `用户请求的chunk在smallbin范围内 && 当前遍历到的chunk是unsorted bin范围内 && 存在last_remainder && victim可分割` 的条件时，会对last remainder进行分割并取出，然后将分割后的chunk放入unsorted bin链表中
		- 大小不匹配时
			会根据victim的大小将chunk放入 `large bin/small bin` 
		- 大小刚好时
			直接取出
- large bin 部分
	```c
	  // largebin 部分
	      if (!in_smallbin_range (nb))                                                    //在large bin
	        {
	          bin = bin_at (av, idx);                                                     //取large bin起始地址
	
	          /* skip scan if empty or largest chunk is too small */
	          // 如果对应的 bin 为空或者其中的chunk最大的也很小，那就跳过
	          // first(bin)=bin->fd 表示当前链表中最大的chunk
	          if ((victim = first (bin)) != bin                                           //这行代码检查链表是否为空
	        && (unsigned long) chunksize_nomask (victim)
	          >= (unsigned long) (nb))                                                    //这行代码检查链表中第一个内存块的大小是否大于或等于请求的内存大小
	            {
	              victim = victim->bk_nextsize;
	              // 反向遍历链表，直到找到第一个不小于所需chunk大小的chunk
	              while (((unsigned long) (size = chunksize (victim)) <
	                      (unsigned long) (nb)))
	                victim = victim->bk_nextsize;
	                // 如果最终取到的chunk不是该bin中的最后一个chunk，并且该chunk与其前面的chunk
	                // 的大小相同，那么我们就取其前面的chunk，这样可以避免调整bk_nextsize,fd_nextsize
	                // 链表。因为大小相同的chunk只有一个会被串在nextsize链上。
	
	              /* Avoid removing the first entry for a size so that the skip
	                 list does not have to be rerouted.  */
	              if (victim != last (bin)
	      && chunksize_nomask (victim)
	        == chunksize_nomask (victim->fd))
	                victim = victim->fd;
	
	              remainder_size = size - nb;
	              // 取出victim
	              unlink (av, victim, bck, fwd);
	
	              /* Exhaust */
	              // 如果剩余的chunk不足以当作一个chunk
	              // 则不对chunk进行分割，直接将整个chunk拿来使用
	              if (remainder_size < MINSIZE)
	                {
	                  set_inuse_bit_at_offset (victim, size);
	                  if (av != &main_arena)
	        set_non_main_arena (victim);
	                }
	              /* Split */
	              // 分割的情况
	              else
	                {
	                  // 获取剩余的指针
	                  remainder = chunk_at_offset (victim, nb);
	                  /* We cannot assume the unsorted list is empty and therefore
	                     have to perform a complete insert here.  */
	                  // 插入unsorted bins中
	                  bck = unsorted_chunks (av);
	                  fwd = bck->fd;
	                  // 检查unsorted bins是否被破坏
	      if (__glibc_unlikely (fwd->bk != bck))
	        malloc_printerr ("malloc(): corrupted unsorted chunks");
	                  remainder->bk = bck;
	                  remainder->fd = fwd;
	                  bck->fd = remainder;
	                  fwd->bk = remainder;
	                  //large chunk的情况
	                  if (!in_smallbin_range (remainder_size))
	                    {
	                      remainder->fd_nextsize = NULL;
	                      remainder->bk_nextsize = NULL;
	                    }
	                  set_head (victim, nb | PREV_INUSE |
	                            (av != &main_arena ? NON_MAIN_ARENA : 0));
	                  set_head (remainder, remainder_size | PREV_INUSE);
	                  set_foot (remainder, remainder_size);
	                }
	              check_malloced_chunk (av, victim, nb);
	              void *p = chunk2mem (victim);
	              alloc_perturb (p, bytes);
	              return p;
	            }
	        }
	
	      /*
	         Search for a chunk by scanning bins, starting with next largest
	         bin. This search is strictly by best-fit; i.e., the smallest
	         (with ties going to approximately the least recently used) chunk
	         that fits is selected.
	
	         The bitmap avoids needing to check that most blocks are nonempty.
	         The particular case of skipping all bins during warm-up phases
	         when no chunks have been returned yet is faster than it might look.
	       */
	
	      ++idx;
	      bin = bin_at (av, idx);                       // 获取对应的bin
	      block = idx2block (idx);                      // Binmap按block管理，每个block为一个int，共32个bit，可以表示32个bin中是否有空闲chunk存在
	      map = av->binmap[block];
	      bit = idx2bit (idx);
	
	      for (;; )
	        {
	          /* Skip rest of block if there are no more set bits in this block.  */
	          // 如果bit>map，则表示该 map 中没有比当前所需要chunk大的空闲块
	          // 如果bit为0，那么说明，上面idx2bit带入的参数为0。
	          if (bit > map || bit == 0)
	            {
	              do
	                {
	                  // 寻找下一个block，直到其对应的map不为0。如果已经不存在的话，那就只能使用top chunk了
	                  if (++block >= BINMAPSIZE) /* out of bins */
	                    goto use_top;                                      // 使用top chunk
	                }
	              while ((map = av->binmap[block]) == 0);
	
	              bin = bin_at (av, (block << BINMAPSHIFT));
	              bit = 1;
	            }
	
	          /* Advance to bin with set bit. There must be one. */
	          // 从当前map的最小的bin一直找，直到找到合适的bin。
	          // 这里是一定存在的
	          while ((bit & map) == 0)
	            {
	              bin = next_bin (bin);
	              bit <<= 1;
	              assert (bit != 0);
	            }
	
	          /* Inspect the bin. It is likely to be non-empty */
	          // 检查并取出chunk
	          victim = last (bin);
	
	          /*  If a false alarm (empty bin), clear the bit. */
	          if (victim == bin)
	            {
	              av->binmap[block] = map &= ~bit; /* Write through */
	              bin = next_bin (bin);
	              bit <<= 1;
	            }
	
	          else
	            {
	              size = chunksize (victim);
	
	              /*  We know the first chunk in this bin is big enough to use. */
	              assert ((unsigned long) (size) >= (unsigned long) (nb));
	
	              remainder_size = size - nb;
	
	              /* unlink */
	              unlink (av, victim, bck, fwd);
	
	              /* Exhaust */
	              if (remainder_size < MINSIZE)
	                {
	                  set_inuse_bit_at_offset (victim, size);
	                  if (av != &main_arena)
	        set_non_main_arena (victim);
	                }
	
	              /* Split */
	              else
	                {
	                  remainder = chunk_at_offset (victim, nb);
	
	                  /* We cannot assume the unsorted list is empty and therefore
	                     have to perform a complete insert here.  */
	                  bck = unsorted_chunks (av);
	                  fwd = bck->fd;
	      if (__glibc_unlikely (fwd->bk != bck))
	        malloc_printerr ("malloc(): corrupted unsorted chunks 2");
	                  remainder->bk = bck;
	                  remainder->fd = fwd;
	                  bck->fd = remainder;
	                  fwd->bk = remainder;
	
	                  /* advertise as last remainder */
	                  if (in_smallbin_range (nb))
	                    av->last_remainder = remainder;
	                  if (!in_smallbin_range (remainder_size))
	                    {
	                      remainder->fd_nextsize = NULL;
	                      remainder->bk_nextsize = NULL;
	                    }
	                  set_head (victim, nb | PREV_INUSE |
	                            (av != &main_arena ? NON_MAIN_ARENA : 0));
	                  set_head (remainder, remainder_size | PREV_INUSE);
	                  set_foot (remainder, remainder_size);
	                }
	              check_malloced_chunk (av, victim, nb);
	              void *p = chunk2mem (victim);
	              alloc_perturb (p, bytes);
	              return p;
	            }
	        }
	```
	如果请求的 chunk 在 large chunk 范围内，就在对应的 bin 中从小到大进行扫描，找到第一个合适的。
- Top chunk 部分
	```c
	    use_top:
	      /*
	         If large enough, split off the chunk bordering the end of memory
	         (held in av->top). Note that this is in accord with the best-fit
	         search rule.  In effect, av->top is treated as larger (and thus
	         less well fitting) than any other available chunk since it can
	         be extended to be as large as necessary (up to system
	         limitations).
	  
	         We require that av->top always exists (i.e., has size >=
	         MINSIZE) after initialization, so if it would otherwise be
	         exhausted by current request, it is replenished. (The main
	         reason for ensuring it exists is that we may need MINSIZE space
	         to put in fenceposts in sysmalloc.)
	       */
	      // 获取当前chunk并计算大小
	      victim = av->top;
	      size = chunksize (victim);
	      // 如果分割后top chunk 大小仍然满足 chunk 的最小大小，那么就可以直接进行分割。
	      if ((unsigned long) (size) >= (unsigned long) (nb + MINSIZE))
	        {
	          // 计算剩余大小
	          remainder_size = size - nb;
	          // 获取剩余指针
	          remainder = chunk_at_offset (victim, nb);
	          // 更新top指针
	          av->top = remainder;
	          set_head (victim, nb | PREV_INUSE |
	                    (av != &main_arena ? NON_MAIN_ARENA : 0));
	          set_head (remainder, remainder_size | PREV_INUSE);
	  
	          check_malloced_chunk (av, victim, nb);
	          void *p = chunk2mem (victim);
	          alloc_perturb (p, bytes);
	          return p;
	        }
	  
	      /* When we are using atomic ops to free fast chunks we can get
	         here for all block sizes.  */
	      // 清理内存碎片
	      else if (atomic_load_relaxed (&av->have_fastchunks))
	        {
	          malloc_consolidate (av);
	          /* restore original bin index */
	          if (in_smallbin_range (nb))
	            idx = smallbin_index (nb);
	          else
	            idx = largebin_index (nb);
	        }
	  
	      /*
	         Otherwise, relay to handle system-dependent cases
	       */
	      // 堆内存不够时，sysmalloc申请到libc上
	      else
	        {
	          void *p = sysmalloc (nb, av);
	          if (p != NULL)
	            alloc_perturb (p, bytes);
	          return p;
	        }
	    }
	}
	```
	如果所有的 bin 中的 chunk 都没有办法直接满足要求（即不合并），或者说都没有空闲的 chunk。那么我们就只能使用 top chunk 了。