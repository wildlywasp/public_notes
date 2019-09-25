# Kernel

1. #### `__attribute__((attribute-list))`

   GNU C允许声明函数属性（Function Attribute）、变量属性（Variable Attribute）、类型属性（Type Attribute）。

   ```c
   #define __pure				__attribute__((pure))
   #define __aligned(x)		__attribute__((aligned(x)))
   #define __printf(a, b)		__attribute__((format(printf, a, b)))
   #define __scanf(a, b)		__attribute__((format(scanf, a, b)))
   #define noinline			__attribute__((noinline))
   #define __attribute_const__ __attribute__((__const__))
   #define __maybe_unused		__attribute__((unused))
   #define __always_unused		__attribute__((unused))
   ```

2. #### 零长数组

   GUN C允许使用变长数组，这在定义数据结构时非常有用。

   ```c
   struct line{
       int length;
       char contents[0];
   }
   struct line *thisline = malloc(sizeof(struct line) + this_length);
   thisline->length = this_length;
   ```

   可通过`thisline->contents[index]`访问对应位置的数据。
   
3. #### 设备驱动

   字符驱动

   

4. #### 系统调用与POSIX标准

   ​	操作系统的API接口层POSIX（Portable Operation System Interface of UNIX）源自UNIX。应用编程接口（API）可由一个或多个系统调用实现，甚至不使用任何系统调用。

5. #### 新增系统调用？

   ​	不建议新增系统调用，新增系统调用可能缺乏移植性。Linux系统调用由Linux社区和glibc社区协同解决。一些可用的用户程序与内核的信息交互：

   - 设备节点：实现一个设备节点，可对设备进行`read()`和`write()`操作，还可通过ioctl()接口自定义操作；

   - sysfs接口：sysfs接口是一种Linux内核推荐的用户程序与内核通信的方式。

6. #### 分段与分页

   ​	基本思想都是把程序所需的内存空间的虚拟地址映射到某个物理地址空间中。区别在于颗粒度，分段机制对虚拟内存到物理内存的映射依旧以进程为单位，而分页机制将映射进一步细分为固定大小的页（Page）。内存不足时，分段机制换出到磁盘的是整个进程，而根据局部性原理，分页机制将不常用的数据和代码换出到磁盘，可以节省带宽同时常用数据和代码驻留内存可以提供比较好的性能。

   ​	分段与分页的映射都是通过CPU来完成的。对于分页机制，相应的物理内存也是以页为单位来管理，称为物理页面（Physical Page）或者页帧（Page Frame）。进程虚拟地址空间的页称为虚拟页（Virtual Page），操作系统为了管理页帧需要按物理地址顺序给每个页帧编号，称为页帧号（Page Frame Number, PFN）。

   ​	分页机制由硬件实现，CPU内部由专门的硬件单元——内存管理单元（Menmory Management Unit, MMU）负责虚拟页面到物理页面的转换。MMU内部通常包含高速缓存（TLB）和页表查询模块（Table Walk Unit, ARM），高速缓存缓存了页表转换的结果，减少内存访问时间，当高速缓存（TLB）未命中，由页表查询模块执行页表的翻译和查找。`struct page`定义在`include/linux/mm_types.h`中。

7. #### 内核image的内存空间

   - text段：_text 和 _etext为代码段的起始和结束地址，包含了编译后的内核代码；
   - init段：__init_begin 和 _init_end为init段的起始和结束地址，包含了大部分内核初始化的数据；
   - data段：_sdata 和 _edata为数据段的起始和结束地址，包含大部分内核的已初始化的变量；
   - BSS段：\__bss_start 和  __bss_stop为BSS段的起始和结束地址，包含初始化为0的所有静态全局变量。
   
8. #### 分配和释放页面

   Linux内核的内存页面基于伙伴系统算法（Buddy System）来管理，伙伴系统是操作系统中最常用的的动态存储管理方法之一。当用户提出申请，伙伴系统分配一块大小合适的内存给用户，反之在用户释放内存块时回收。伙伴系统中的内存块大小是2的order次幂个物理页面（通常为4KB），在Linux中order最大值用MAX_ORDER来表示，通常值为11，order范围是`0 ... (MAX_OEDER-1)`，此时最大内存块大小为4MB。

   **页面分配函数：**

   ```c
   // alloc_pages()函数用来分配2的order次幂个连续物理页面，返回值是第一个物理页面的 struct page 数据结构
   // gfp_mask是分配掩码  order是内存块大小控制参数，不大于MAX_OEDER 
   static inline struct page * alloc_pages(gfp_t gfp_mask, unsigned int order)
   
   // 返回所分配内存的内核空间的虚拟地址，线性映射的物理内存会直接返回线性映射区的内核空间虚拟地址
   // 高端映射的物理内存返回动态映射的虚拟地址
   unsigned long __get_free_pages(gfp_t gfpt_mask, unsigned int order)
   
   // 高端映射的内存虚拟地址转换函数
   void * page_address(const struct page *page)
       
   // 单独分配一个物理页面
   #define allo_page(gfp_mask) alloc_pages(gfp_mask, 0)
   #define __get_free_page(gfp_mask)\
   		__get_free_pages((gfp_mask), 0)
   
   // 分配一个全填充为0的物理页面
   unsigned long get_zeroed_page(gfp_t gfp_mask)
   ```

   **页面释放函数：**

   ```c
   // struct page物理页面的地址 order页面数量控制参数
   // 错误的struct page指针或者order值回引起系统崩溃
   void __free_pages(struct page *page, unsigned int order)
   
   // 释放单个物理页面
   #define __free_page(page) __free_pages((page), 0)
   
   // addr逻辑页面地址 order页面数量控制参数
   void free_pages(unsigned long addr, unsigned int order)
   
   // 释放当个逻辑页面
   #define free_page(addr) free_pages((addr), 0)
   ```

   **分配掩码（gfp_mask）：**

   `gfp_mask`是一个`unsigned`类型的变量

   ```c
   typedef unsigned __bitwise__ gfp_t
   ```

   `gfp_mask`分配掩码定义在`include/linux/gfp.h`文件中，大致分类为：

   - 内存管理区修饰符（zone modifier）
   - 移动修饰符（mobility and placement modifier）
   - 水位修饰符（watermark modifier）
   - 页面回收修饰符（page reclaim modifier）
   - 行动修饰符（action modifier）

   内存管理区修饰符：主要用来表示应当从哪些内存管理区来分配物理内存，内存管理区修饰符使用gfp_mask的最低4个比特位来表示。

   移动修饰符：主要用来指示分配出来的页面具有迁移属性，迁移类型属性是为了解决内存外碎片化的问题。

   水位修饰符：用来控制是否可以访问系统紧急预留的内存。

   页面回收修饰符：略

   行动修饰符：略

   **内存碎片化：**

   为了减少碎片，伙伴系统算法中，伙伴内存块有以下3个基本条件：

   - 两个块大小相同
   - 两个块地址连续
   - 两个块是同一个大块中分离出来的（保证合并后不会导致外碎片）

9. #### 分配小块内存

   当内核需要分配几十字节的小块内存，若使用页面分配器分配一个页面会显得浪费资源，因此必须有一种管理小块内存的新分配机制——slab机制。对于需要经常分配的数据结构，在内存不紧张的时候创建该数据结构的对象缓存池，并预先分配好若干个空闲的对象，当内核需要该种对象时可直接使用缓存池中的预分配空闲对象。slab机制有两个变种：slob机制和slub机制。slab主要有两个缺点：slab分配器使用的元数据开销比较大，元数据可理解为管理成本；嵌入式系统中，slab分配器的代码量和复杂度都很高。

   slab描述符使用`struct kmem_cache`数据结构描述

   **slab分配机制：**
   
   ```c
   // 创建slab描述符
   // name：slab描述符  size：缓存对象大小  align：缓存对象需要对齐的字节数
   // flags：分配掩码  cotor：对象的构造函数
   struct kmem_cache *kmem_cache_create(const char *name, size_t size, size_t align,
                                        unsigned long flags, void (*ctor)(void *))
       
   // 释放描述符
   void kmem_cache_destroy(struct kmem_cache *s)
       
   // 分配缓存对象
   void *kmem_cache_alloc(struct kmem_cache *, gfp_t flags)
   
   // 释放缓存对象
   void kmem_cache_free(struct kmem_cache *, void *)
   ```
   
   **slab分配思想：**
   
   slab机制最核心的分配思想是空闲时建立缓存对象池，包括本地对象缓存池和共享对象缓存池。本地对象缓存池只能由本地CPU访问，可减少多核CPU之间的锁的竞争。共享对象缓存池由所有CPU共享，当本地缓存池没有空闲对象时，会从共享对象缓存池中取一批空闲对象搬移到本地缓存池中。
   
   **kmalloc机制：**
   
   kmalloc()函数核心是slab机制，类似伙伴系统（buddy system）机制，kmalloc机制按照内存块的大小创建多个slab描述符。系统启动时，`create_kmalloc_caches()`函数会创建名为kmalloc-32、kmalloc-64、kmalloc-128等（2的order次幂字节）的slab描述符，order取值范围在slab、slob、slub中有所不同。
   
   ```c
   // 返回逻辑地址，物理页面位于线性映射区
   // kmalloc完成后需要初始化
   void *kmalloc(size_t size, gfp_t flags)
   
   // 释放kmalloc申请的空间
   void kfree(const void*)
   ```
   
   若需要分配30字节的小内存可以用`kmalloc(30, GFP_KERNEL)`，系统会从名为“kmalloc-32”的slab描述符中分配一个对象。`kmalloc()`分配的是物理内存且物理地址连续，位于线性映射区。连续的物理内存块可用作DMA传输。
   
10. #### 虚拟内存管理

    32位系统中，每个用户进程可以拥有3GB的虚拟地址空间。`malloc()`是用户态常用的内存分配接口函数，`mmap()`是用户态常用的用于建立文件映射或匿名映射的函数。这些进程地址空间在内核中使用`struct vm_area_struct`数据结构（VMA）来描述，称为进程地址空间或进程线性区（记录分配的内存在进程虚拟地址空间中的地址信息）。
    
    **进程地址空间：**
    
    每个进程都有独立的进程地址空间，地址空间之间不冲突。进程通过内核的内存管理机制动态地添加和删除内存区域（由VMA管理）。每个内存区域具有相关权限：可读、可写、可执行。进程访问了无效内存区域，或者非法访问内存区域、或者不正确方式访问内存区域，处理器会报告缺页异常，严重时报告“Segment Fault”段错误并终止进程。
    
    进程内存区域包含以下内容：
    
    - 代码段映射区域，可执行文件（如ELF文件）中包含可读可执行的程序头，如代码段和init段等
    - 数据段映射区域，可执行文件中包含可读可写的程序头，如数据段和bss段等
    - 用户进程栈区域，通常是用户空间的最高地址，由上往下。包含栈帧（`rbp`），栈内容包含局部变量和函数调用参数，进程栈独立存在并由内核维护，主要用于上下文切换。
    - MMAP映射区域，位于用户进程栈下面（地址低于栈），主要用于mmap系统调用，如映射文件内容到进程地址空间
    - 堆映射区域，`malloc()`函数分配的进程虚拟地址位于此处，地址由下往上
    
    **内存描述符mm_struct：**
    
    Linux内核需要管理每个进程所有的内存区域及其对应的页表映射，抽象出的数据结构是`mm_struct`，位于`include/linux/mm_types.h`，进程控制块（PCB）数据结构`task_struct`中有指针`mm`指向`mm_struct`数据结构。
    
    ```c
    struct mm_struct {
    	struct {
    		struct vm_area_struct *mmap;		/* list of VMAs */
    		struct rb_root mm_rb;
    		u64 vmacache_seqnum;                   /* per-thread vmacache */
    #ifdef CONFIG_MMU
    		unsigned long (*get_unmapped_area) (struct file *filp,
    				unsigned long addr, unsigned long len,
    				unsigned long pgoff, unsigned long flags);
    #endif
    		unsigned long mmap_base;	/* base of mmap area */
    ...
    		pgd_t * pgd;
    
    		/**
    		 * @mm_users: The number of users including userspace.
    		 *
    		 * Use mmget()/mmget_not_zero()/mmput() to modify. When this
    		 * drops to 0 (i.e. when the task exits and there are no other
    		 * temporary reference holders), we also release a reference on
    		 * @mm_count (which may then free the &struct mm_struct if
    		 * @mm_count also drops to 0).
    		 */
    		atomic_t mm_users;
    
    		/**
    		 * @mm_count: The number of references to &struct mm_struct
    		 * (@mm_users count as 1).
    		 *
    		 * Use mmgrab()/mmdrop() to modify. When this drops to 0, the
    		 * &struct mm_struct is freed.
    		 */
    		atomic_t mm_count;
    ...
    		int map_count;			/* number of VMAs */
    
    ...
    		struct rw_semaphore mmap_sem;
    
    		struct list_head mmlist; /* List of maybe swapped mm's.	These
                                      * are globally strung together off
                                      * init_mm.mmlist, and are protected
                                      * by mmlist_lock
                                      */
    ...
    		unsigned long total_vm;	   /* Total pages mapped */
    ...
    		unsigned long start_code, end_code, start_data, end_data;
    		unsigned long start_brk, brk, start_stack;
    ...
    }
    ```
    
    - `mmap`：进程所有的VMA形成的单链表的链表头
    - `mm_rb`：VMA红黑树的根节点
    - `get_unmapped_area`：判断虚拟内存空间是否有足够的空间，返回一段没有映射的空间的起始地址
    - `mmap_base`：指向mmap区域的起始地址（32位处理器中为`0x40000000`）
    - `pgd`：指向进程的页表PGD目录（一级目录）
    - `mm_users`：记录正在使用该进程空间的进程数目（两个进程共享则值为2）
    - `mm_count`：mm_struct结构体主引用计数
    - `mmap_sem`：保护进程地址空间VMA的读写信号量
    - `mmlist`：所有mm_struct数据结构都连接到一个双向链表，链表头为init_mm内存描述符，它是init进程的地址空间
    - `start_code, end_code`：代码段起始结束地址
    - `start_data, end_data`：数据段起始结束地址
    - `total_vm`：已使用的进程空间总和
    
    **VMA管理：**
    
    进程VMA管理通过VMA单项链表和红黑树来实现查找、插入和合并操作。
    
    **malloc分配函数：**
    
    `malloc()`是C语言标准库里封装的一个核心函数，C语言标准库处理后通过系统调用brk向系统申请内存。`malloc()`为 用户进程维护了一小部分内存，当这部分内存不够时通过brk系统调用向内核申请内存。
    
    ```c
    #SYSCALL_DEFINE1(brk, unsigned long, brk)
    ```
    
    `malloc()`分配完内存后用户空间可见虚拟内存，但没有建立虚拟内存与物理内存之间的映射关系（访问未建立映射关系的虚拟内存会触发缺页异常）。每个进程有自己独立的页表，`mm_struct`数据结构中的`pgd`成员指向这个页表的基地址。`fork()`函数创建新进程时会初始化一份页表。
    
    **mmap：**
    
    mmap/munmap接口函数是用户空间最常用的两个系统调用接口，可用于分配内存、读写大文件、链接动态库和多进程共享内存等。
    
    ```c
    #include <sys/mman.h>
    // addr:指定映射到地址空间的起始地址，为了可移植性通常设置为NULL让内核选择
    // length：映射进地址空间的大小
    // prot：用于设置内存映射区域的读写属性等
    // flags：设置内存映射属性
    // fd：文件映射的句柄，-1表示匿名映射
    // offset：文件映射的偏移量
    void *mmap(void *addr, size_t length, int prot, int flags, int fd, off_t offset)
        
    int munmap(void *addr, size_t length)
    ```
    
    **mmap映射类型：**
    
    - 私有匿名映射：常见在glibc分配大块内存中，当需要分配的内存大于MMAP_THREASHOLD(128KB)，glibc会默认调用mmap代替brk分配内存，参数设置：`fd = -1`和`flags = MAP_ANONYMOUS|MAP_PRIVATE`。
    
    - 共享匿名映射：相关进程可共享一块内存，常用于父子进程间通信，参数设置：
    
      `fd = -1`和`flags = MAP_ANONYMOUS|MAP_SHARED`。
    
    - 私有文件映射：常用于加载动态共享库，参数设置：`flags = MAP_PRIVATE`。
    
    - 共享文件映射：常用于读写文件和进程间通信，参数设置：`flags = MAP_SHARED`。
    
    **缺页异常：**
    
    缺页异常处理需要考虑很多细节，如：匿名页面、KSM页面、page cache页面、写时复制、私有映射和共享复制，缺页异常处理依赖处理器体系架构。`fork()`创建子进程时，内核不需要复制父进程完整地址空间给子进程，而是共享一个副本，只有当需要写入时数据才会被复制到子进程地址空间。
    
11. #### 内存短缺

    **页面置换算法**：LRU（Least Recently Used）、二次机会法

    LRU假定最近不使用的页在较短时间内不会频繁使用，内存不足时会被换出到磁盘。内核中共有5个LRU链表：

    - 不活跃匿名页面链表`LRU_INACTIVE_ANON`
    - 活跃匿名页面链表`LRU_ACTIVE_ANON`
    - 不活跃文件映射页面链表`LRU_INACTIVE_FILE`
    - 活跃文件映射页面链表`LRU_ACTIVE_FILE`
    - 不可回收页面链表`LRU_UNEVICTABLE`

    内存紧缺时文件缓存页面总是被优先换出，而不是匿名页表，因大多数情况文件缓存不需要写回磁盘。

    **二次机会法**：LRU的基础上引入访问状态位，第一次检查置0，再次被使用置1，检查为零的尾页面换出。

    **OOM Killer**：选择占用内存比较高的进程终止。
    
12. #### 进程

    程序指完成指定任务的一系列指令集和或者一个可执行文件，包含可运行的CPU指令和相应数据。进程是一段执行中的程序，包含程序数据和运行状态。进程是操作系统分配内存、CPU时间片等资源的基本单位。

    **进程描述符：**

    进程是操作系统调度的实体，进程控制块（Process Control Block，PCB）用来描述这个实体，Linux内核中描述进程控制块的结构体是`task_struct`，定义在`include/linux/sched.h`中。

    `task_struct`数据结构中的内容主要有：

    - 进程属性
    - 进程间关系
    - 进程调度相关信息
    - 内存相关信息
    - 文件管理相关信息
    - 信号相关信息
    - 资源限制相关信息

    **进程的状态：**

    Linux的进程有五种状态：

    | 状态                   | 描述                                                         |
    | ---------------------- | ------------------------------------------------------------ |
    | `TASK_RUNNING`         | 可运行态或就绪态：进程处于可执行的状态，正在执行或者在就绪队列中等待执行。 |
    | `TASK_INTERRUPTABLE`   | 可中断睡眠态：进程进入睡眠状态（被阻塞）来等待某些资源或条件到位，一旦资源或条件到位，内核立即将进程状态设置为`TASK_RUNNING`。 |
    | `TASK_UNINTERRUPTABLE` | 不可中断态：与`TASK_INTERRUPTABLE`类似，但不同的是，该状态下进程睡眠不受干扰，不响应信号。`ps`命令看到进程被标记`D`状态，该状态也被称为深度睡眠状态。 |
    | `__TASK_STOPPED`       | 终止态：进程停止运行了。                                     |
    | `EXIT_ZOMBIE`          | 僵尸态：进程已消亡，但`task_struct`数据结构仍未释放。        |

    进程状态的设置可通过赋值完成：

    ```c
    p->state  = TASK_RUNNING
    ```

    Linux内核提供了两个API函数来设置进程状态，且均会考虑SMP多核环境下的竞争情况：

    ```c
    #define set_task_state(tsk, state_value)	\
    	set_mb((tsk)->state, (state_value))
    
    #define set_current_state(state_value)	\
    	set_mb(current->state, (state_value))
    ```

    **进程标志：**

    进程被创建时会分配唯一的标识码PID（Process Identifier），PID是`int`类型，默认最大值是32768，Linux内核中的bitmap机制管理分配PID编号，保证循环使用PID和每个进程PID唯一。同时Linux内核引入了线程组的概念，每个线程有自己唯一的PID但共享相同的TGID，`getpid()`系统调用返回的也是当前进程的TGID，而不是线程的PID，`gettid()`返回线程的PID。

    **进程间关系：**

    Linux内核启动时会有一个init_task进程，称为0号进程、idle进程或swapper进程，当系统没有进程需要调度时，调度器就会去执行idle进程。idle进程在内核启动（`start_kernel()`）时静态创建，所有的核心数据结构都预先静态赋值。初始化完成后会创建init进程，这个进程开始参与调度。进程的数据结构`task_struct`通过`list_head`类型双向链表连接在一起，进程链表头是init_task进程。

13. #### 进程的创建与终止

    shell执行程序是通过调用`fork()`来创建一个新的进程，而后调用`execve()`加载执行程序。Linux中的进程创建和执行通常是由两个单独的函数去完成的，`fork()`和`execve()`。`fork()`通过写时复制技术复制当前进程的相关信息来复制和创建一个全新的子进程，此时父进程与子进程区别在于PID、PPID和某些资源以及统计量，但共享相同的进程地址空间。`execve()`负责读取可执行文件，并将其装入子进程的地址空间中并开始运行。Linux内核提供了相应的的系统调用，比如`sys_fork`、`sys_exec`、`sys_vfork`、`sys_clone`，C语言库中有对应的的函数封装。`fork()`、`vfork()`、`clone()`以及内核线程都是通过调用`do_fork()`函数完成的。

    ```c
    // fork()
    do_fork(SIGCHLD, 0, 0, NULL, NULL)
    
    // vfork()
    do_fork(CLONE_VFORK|CLONE_VM|SIGCHLD, 0, 0, NULL, NULL)
    
    // clone
    do_fork(clone_flags, newsp, 0, parent_tidptr, child_tidptr)
    
    // kernel thread
    do_fork(flags|CLONE_VM|CLONE_UNTRACED, (unsigned long)fn, (unsigned long)arg, NULL, NULL)
    ```

    **写时复制技术(COW)：**

    写时复制技术在进程创建中，父进程不需要复制进程地址空间给子进程，只需将父进程的进程地址空间的页表复制给子进程，从而父子进程共享同一进程地址空间。进程地址空间的共享是以只读的方式进行的，当任意一方需要修改某个物理页面时会触发写保护的缺页异常，此时将共享页面的内容复制出来。进程空间的只读是对父子进程双方的，只有写入时才发生复制，可以推迟甚至避免复制数据，减小系统开销。

    **`fork()`：**

    使用`fork()`创建子进程，子进程会从父进程继承整个地址空间（子进程复制了父进程的页表），包括进程上下文、进程堆栈、内存信息、打开的文件描述符、进程优先级、根目录、资源限制、控制终端等。不同点在于：

    - 子进程与父进程的PID不一样
    - 子进程不会继承父进程的内存方面的锁，如`mlock()`
    - 子进程不会继承父进程的一些定时器，如`setitimer()`、`alarm()`、`timer_create()`

    ```c
    #include <unistd.h>
    
    pid_t fork(void);
    
    SYSCALL_DEFINE0(fork){
        return do_fork(SIGCHLD, 0, 0, NULL, NULL);
    }
    ```

    `fork()`函数有两次返回，一次在父进程中，返回值为子进程PID，一次在子进程中，返回值为`0`，返回`-1`则说明创建失败。`fork()`使用了写时复制技术。

    **`vfork()`：**

    通过`vfork()`创建的子进程直接使用父进程的进程空间（页表也不复制），同时阻塞父进程，直至子进程调用`execve()`或`exit()`，子进程需避免修改全局数据结构或全局变量中的任何信息，因此子进程也不可从当前栈框架中返回（return会销毁栈）。`vfork()`为了解决没有写时复制技术的空间浪费和效率低下提出。

    ```c
    #include <sys/types.h>
    #include <unistd.h>
    
    pid_t vfork(void);
    
    SYSCALL_DEFINE0(vfork){
        return do_fork(CLONE_VFORK|CLONE_VM|SIGCHLD, 0, 0, NULL, NULL);
    }
    ```

    **`clone()`：**

    `clone()`常用来创建用户线程，Linux内核中线程与进程的定位一致，都通过`task_struct`数据结构描述，没有特殊的数据结构和调度算法来描述线程。`clone()`功能强大参数众多，可有选择的继承父进程的资源，甚至创建兄弟关系进程。

    **内核线程：**

    内核线程是独立运行在内核空间的进程，与普通用户进程的区别在于，内核线程没有独立的进程地址空间，其`task_struct`中`mm`指针设置为NULL，只能运行在内核空间，和普通进程一样参与到系统调度中。常见的内核线程有页面回收线程“kswapd”。

    ```c
    kthread_create(threadfn, data, namefmt, arg, ...)
    
    kthread_run(threadfn, data, namefmt, ...)
    ```

    `kthread_create()`创建的内核线程处于不可运行状态，需要调用`wake_up_process()`将其唤醒并加入就绪队列，`kthread_run()`可创建一个马上可以运行的内核线程。内核线程最终还是通过`do_fork()`来实现的。

    **`do_fork()`：**

    `do_fork()`主要调用`copy_process()`创建子进程的`task_struct`数据结构，以及完成从父进程复制必要内容到子进程的`task_struct`数据结构，完成子进程的创建。
    
    ```c
    // kernel/fork.c
    // clone_flags：标志位集合
    // stack_start：用户栈起始位置
    // stack_size：用户栈大小，通常设置为0
    // parent_tidptr child_tidptr：分别指向父子进程的PID
    long do_fork(unsigned long clone_flags,
    	      unsigned long stack_start,
    	      unsigned long stack_size,
    	      int __user *parent_tidptr,
	      int __user *child_tidptr)
    ```
    
    **终止进程：**
    
    进程主动终止：
    
    - 从main函数返回，链接程序会自动添加对`exit()`系统调用
    - 主动调用`exit()`
    
    进程被动终止：
    
    - 进程收到无法处理的信号
    - 进程在内核态执行时产生了一个异常
    - 进程收到SIGKILL等终止信号
    
    进程终止时，Linux内核会释放它所占有的资源，并通知父进程，此时有两种情况：
    
    - 子进程先于父进程终止，子进程进入僵尸状态（EXIT_ZOMBIE），直到父进程调用`wait()`才能最终销毁
    - 子进程后于父进程终止，init进程称为子进程新的父进程
    
    僵尸状态进程除进程描述符外的所有资源已归还给内核，Linux内核将终止进程的清理工作和释放进程描述符的工作分开，是为了系统知道子进程终止原因等信息。
    
14. #### 进程调度

    根据占用处理器的情况，进程可分为：CPU消耗型（CPU-Bound）、I/O消耗型（I/O-Bound）。Linux内核使用0-139表示进程优先级，数值越低优先级越高，0-99对应实时进程，100-139对应普通进程。`nice()`函数可调整进程静态优先级`static_prio`，`normal_prio`基于`static_prio`和调度策略计算得出，`prio`保持着进程的动态优先级。早期Linux采用固定时间片调度进程，现在采用CFS调度器根据进程权重比的方法公平划为CPU时间。

    **Linux CFS调度算法：**

    CFS调度算法引入虚拟时钟概念，每个进程的虚拟时间是是实际时间相对nice值为0的权重的比例值。nice值越小的进程，优先级高且权重大，其虚拟时钟比真实时钟跑得慢，可以获得更多运行时间，反之，nice值越大的进程，优先级低且权重小，其虚拟时钟跑得比真实时钟跑得快，获得的更少运行时间。CFS调度器选择下一个进程的规则简单，挑选vruntime值最小的进程执行，同时CFS使用红黑树来组织就绪队列，可快速找到vruntime最小的那个进程，只需查找树中最左侧的叶子节点即可。CFS调度器通过`pick_next_task_fair()`来调用调度类中的`pick_next_task()`方法。

    以下是进程权重、优先级和vruntime的计算方法。

    ```c
    // kernel/sched/fair.c
    static inline u64 calc_delta_fair(u64 delta, struct sched_entity *se)
    {
    	if (unlikely(se->load.weight != NICE_0_LOAD))
    		delta = __calc_delta(delta, NICE_0_LOAD, &se->load);
    
    	return delta;
    }
    
    static u64 __calc_delta(u64 delta_exec, unsigned long weight, struct load_weight *lw)
    {
    	u64 fact = scale_load_down(weight);
    	int shift = WMULT_SHIFT;
    
    	__update_inv_weight(lw);
    
    	if (unlikely(fact >> 32)) {
    		while (fact >> 32) {
    			fact >>= 1;
    			shift--;
    		}
    	}
    
    	/* hint to use a 32x32->64 mul */
    	fact = (u64)(u32)fact * lw->inv_weight;
    	// 防止后一步计算溢出
    	while (fact >> 32) {
    		fact >>= 1;
    		shift--;
    	}
    	// 没有前一步可能溢出u64
    	return mul_u64_u32_shr(delta_exec, fact, shift);
    }
    ```

    **调度类：**
    
    内核主要实现了4套调度策略，分别是`SCHED_FAIR`、`SCHED_RT`、`SCHED_DEADLINE`、`SCHED_IDLE`，且均按sched_class类来实现。4种调度类通过next指针串联，可通过调度策略API函数（`sched_setscheduler()`）来设定用户进程的调度策略。`SCHED_NORMAL`和`SCHED_BATCH`使用CFS调度器，`SCHED_FIFO`和`SCHED_RR`使用realtime调度器，`SCHED_IDLE`指idle调度器，`SCHED_DEADLINE`指deadline调度器。
    
    ```c
    // include/uapi/linux/sched.h
    
    /* Scheduling policies */
    #define SCHED_NORMAL		0
    #define SCHED_FIFO		1
    #define SCHED_RR		2
    #define SCHED_BATCH		3
    /* SCHED_ISO: reserved but not implemented yet */
    #define SCHED_IDLE		5
    #define SCHED_DEADLINE		6
    ```
    
    **进程切换：**
    
    `__schedule()`是调度器的核心函数，其作用是让调度器选择和切换到一个合适的进程并运行。触发调度的情况有以下三种：
    
    - 阻塞操作：互斥量、信号量、等待队列等
    - 中断返回前和系统调用返回用户空间时：检查TIF_NEED_RESCHED重调度标志位以判断是否需要调度
    - 将要被唤醒的进程不会立即调用`schedlue()`要求被调度，而是加入CFS就绪队列并设置TIF_NEED_RESCHED重调度标志位
    
    ```c
    // kernel/sched/core.c
    // line:4125
    
    asmlinkage __visible void __sched schedule(void)
    {
    	struct task_struct *tsk = current;		// 当前进程结构体
    
    	sched_submit_work(tsk);		// 避免死锁
    	do {
    		preempt_disable();		// 关闭内核抢占
    		__schedule(false);		// 调度
    		sched_preempt_enable_no_resched();		// 开启内核抢占
    	} while (need_resched());		// 如果设置了TIF_NEED_RESCHED标志，则重新调度
    	sched_update_worker(tsk);
    }
    EXPORT_SYMBOL(schedule);
    ```
    
    `__schedule()`函数实现中，主要包含选择下一进程的`pick_next_task()`和执行上下文切换的`context_switch()`
    
    ```c
    // kernel/sched/core.c
    // line:2777
    
    static __always_inline struct rq *
    context_switch(struct rq *rq, struct task_struct *prev,
    	       struct task_struct *next, struct rq_flags *rf)
    {
    	struct mm_struct *mm, *oldmm;
    
    	prepare_task_switch(rq, prev, next);
    
    	mm = next->mm;
    	oldmm = prev->active_mm;
    	/*
    	 * For paravirt, this is coupled with an exit in switch_to to
    	 * combine the page table reload and the switch backend into
    	 * one hypercall.
    	 */
    	arch_start_context_switch(prev);
    
    	/*
    	 * If mm is non-NULL, we pass through switch_mm(). If mm is
    	 * NULL, we will pass through mmdrop() in finish_task_switch().
    	 * Both of these contain the full memory barrier required by
    	 * membarrier after storing to rq->curr, before returning to
    	 * user-space.
    	 */
    	if (!mm) {
    		next->active_mm = oldmm;
    		mmgrab(oldmm);
    		enter_lazy_tlb(oldmm, next);
    	} else
    		switch_mm_irqs_off(oldmm, mm, next);
    
    	if (!prev->mm) {
    		prev->active_mm = NULL;
    		rq->prev_mm = oldmm;
    	}
    
    	rq->clock_update_flags &= ~(RQCF_ACT_SKIP|RQCF_REQ_SKIP);
    
    	prepare_lock_switch(rq, next, rf);
    
    	/* Here we just switch the register state and the stack. */
    	switch_to(prev, next, prev);
    	barrier();
    
    	return finish_task_switch(prev);
    }
    ```
    
    `context_switch`主要包含两部分，`switch_mm_irqs_off()`的将新进程的页表基地址设置到页目录表基地址的寄存器中，完成进程空间的切换，`switch_to()`切换到新进程的内核堆栈和硬件上下文，完成进程切换。硬件上下文：进程恢复执行前必须装入的CPU寄存器的数据。<u>重点：新旧进程的切换点在`switch_to()`内部</u>。
    
    **多核调度：**
    
    SMP（Sysmmetrical Multi-Processing）全称”对称多处理“技术。根据处理器的实际物理属性，CPU域分成如下几类：
    
    | CPU分类                                  | Linux内核分类    | 描述                                                         |
    | ---------------------------------------- | ---------------- | ------------------------------------------------------------ |
    | 超线程（Simultaneous Multi Thread，SMT） | CONFIG_SCHED_SMT | 单个物理核心可以有两个执行线程，超线程可使用相同CPU资源且共享L1缓存，迁移进程不影响缓存利用率 |
    | 多核（MC）                               | CONFIG_SCHED_MC  | 每个物理核心独享L1缓存，多个核心组成一个簇，簇中CPU共享L2缓存 |
    | 处理器（SoC）                            | DIE              | SOC级别                                                      |
    
    
    
15. #### 同步管理

    多个内核路径同时访问和操作数据称为并发访问，可能导致相互覆盖                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        共享数据，造成被访问数据不一致的情况。内核路径可以是一个内核执行路径、中断处理程序或者内核线程等。临界区指访问和操作共享数据的代码段，这些资源无法同时被多个执行线程访问，访问临界区的执行线程或代码路径称为并发源，整个临界区是一个不可分割的整体。原则：<u>保护资源或者数据，而不是保护代码</u>，包括局部静态变量、全局变量、共享的数据结构、缓存、链表、红黑树等各种形式的数据。

    **原子操作：**

    原子操作适用于单一变量的保护，相较于锁机制有更低的开销，同时Linux内核提供了很多原子变量操作函数。

    ```c
    // include/linux/types.h
    typedef struct {
    	int counter;
    } atomic_t;
    ```

    **内存屏障：**

    | 内存屏蔽接口函数             | 描述                                                         |
    | ---------------------------- | ------------------------------------------------------------ |
    | `barrier()`                  | 编译优化屏障，阻止编译器为了性能优化而进行指令重排           |
    | `mb()`                       | 内存屏障（读写），用于SMP和UP                                |
    | `rmb()`                      | 读内存屏障，用于SMP和UP                                      |
    | `wmb()`                      | 写内存屏障，用于SMP和UP                                      |
    | `smp_mb()`                   | 用于SMP的内存屏障。对UP不存在内存顺序问题（对汇编指令），在UP上就是一个优化屏障，确保汇编和C代码的内存顺序一致 |
    | `smp_rmb()`                  | 用于SMP读内存屏障                                            |
    | `smp_wmb()`                  | 用于SMP写内存屏障                                            |
    | `smp_read_barrier_depends()` | 读依赖屏障                                                   |

    **自旋锁：**

    自旋锁的特性：

    - 忙等待的锁机制（锁机制分类：忙等待、睡眠等待）
    - 同一时刻只能有一个内核代码路径可获得锁
    - 自旋锁持有者需尽快完成临界区的执行任务
    - 自旋锁可在中断上下文中使用

    自旋锁的原则：<u>拥有自旋锁的临界区代码必须是原子操作，不能休眠和主动调度</u>。`spin_lock()`与`raw_spin_lock()`：RT-patch补丁使得自旋锁变成可抢占和睡眠的锁，此时禁止抢占和睡眠可用`raw_spin_lock()`。Linux 4.2内核引入队列自旋锁（Queued Spinlock）机制，相比”FIFO ticket-based“有更好的性能表现。

    **信号量：**

    自旋锁是忙等待的锁，信号量则允许进程进入睡眠状态。生产者消费者问题：假设生产者生产商品，消费者购买商品，消费者需要到商店购买。对应于计算机线程，生产者线程生产商品时发现没有空闲内存，必须等待消费者线程释放一个空闲内存，反之，消费者线程购买商品时发现发现没货，必须等待生产者线程。

    ```c
    // include/linux/semaphore.h
    
    // lock：自旋锁变量
    // count：允许进入临界区的内核执行路径个数
    // wait_list: 为获取到锁的进程进入睡眠，并记录在这个链表上
    
    /* Please don't access any members of this structure directly */
    struct semaphore {
    	raw_spinlock_t		lock;
    	unsigned int		count;
    	struct list_head	wait_list;
    };
    
    extern void down(struct semaphore *sem);	// 不可进入睡眠
    extern int __must_check down_interruptible(struct semaphore *sem);	// 争取信号量失败可进入睡眠
    extern int __must_check down_killable(struct semaphore *sem);
    extern int __must_check down_trylock(struct semaphore *sem);	// 获取成功返回0，失败返回1
    extern int __must_check down_timeout(struct semaphore *sem, long jiffies);
    extern void up(struct semaphore *sem);
    ```

    `count>1`表示有多个锁持有者，此时信号量称为计数信号量，`count=1`表示同一时刻仅允许一个持有者，称为互斥量。Linux多用`count=1`的信号量，相比自旋锁，信号量是允许睡眠的锁，适用于一些情况复杂、加锁时间较长的场景。

    **互斥锁（Mutex）：**
    
     MCS锁机制保证只有一个进程自旋等待锁持有者释放锁，避免CPU上的高速缓存行反复失效（缓存一致性）。
    
    ```c
    // include/linux/mutex.h
    
    struct mutex {
    	atomic_long_t		owner;
    	spinlock_t		wait_lock;	// 自旋锁，保护wait_list睡眠等待队列
    #ifdef CONFIG_MUTEX_SPIN_ON_OWNER
    	struct optimistic_spin_queue osq; /* Spinner MCS lock */	// MCS锁机制
    #endif
    	struct list_head	wait_list;
    #ifdef CONFIG_DEBUG_MUTEXES
    	void			*magic;
    #endif
    #ifdef CONFIG_DEBUG_LOCK_ALLOC
    	struct lockdep_map	dep_map;
    #endif
    };
    ```
    
    互斥锁比信号量的实现要高效很多：
    
    - 互斥锁最先实现自旋等待机制
    - 互斥锁在睡眠之前尝试获取锁
    - 互斥锁实现MCS锁避免多个CPU争用锁而导致CPU高速缓存行颠簸现象
    
    由于互斥锁的简洁性和高效性，互斥锁使用场景比信号量更严格，有以下约束条件：
    
    - 同一时刻只有一个线程可以持有互斥锁
    - 只有锁持有者可以解锁
    - 不允许递归加锁
    - 进程持有互斥锁时不可退出
    - 互斥锁必须使用官方API初始化
    - 互斥体可以睡眠，不允许在中断处理程序或者中断下半部中使用（中断处理程序不允许睡眠）
    
    在中断上下文中选择自旋锁，如果临界区有睡眠、隐含睡眠的动作和内核API，应避免使用自旋锁。信号量与互斥锁的选择中，除非不满足上述条件，优先使用互斥锁。
    
    **读写锁：**
    
    不同于信号量不区分临界区的读写属性，读写锁允许多个线程并发地读访问临界区，而写访问限制为一个线程，有效提高了并发性。读写锁特性：
    
    - 允许多个读者进程进入临界区，同时禁止写者进程进入
    - 同一时刻只允许一个写者进程进入临界区
    - 读者和写者进程不能同时进入临界区
    
    RCU（read-copy-update）：
    
    RCU机制要实现读者进程几乎没有开销，即可畅通无阻的访问，而把需要同步的任务交给写者进程，写者进程等待所有读者进程完成后才将旧数据销毁。RCU机制的原理概括为：RCU记录了所有指向共享数据的指针的使用者，当需要修改该共享数据时，首先创建一个副本，并在副本中修改，当所有读者进程都离开临界区后，使用者的指针指向新修改后的副本。
    
    等待队列：
    
    等待队列本质上时一个双向链表，当运行中的进程需要获取某一资源而该资源暂时不能提供时，将进程挂入等待队列中等待该资源的释放，进程进入睡眠状态。