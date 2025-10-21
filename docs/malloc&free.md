该文章深入剖析了 glibc ptmalloc 内存池的核心设计思想、关键数据结构和基本工作流程。

### 核心设计思想

ptmalloc 的核心设计围绕两个基本概念：​**chunk**​（内存块）和 ​**bins**​（空闲链表数组）。其整体架构如下图所示，它清晰地展示了内存分配请求是如何在不同大小的 bins 和 top chunk 之间进行流转的：

![](https://hunyuan-plugin-private-1258344706.cos.ap-nanjing.myqcloud.com/pdf_youtu/img/1761030298_26826ae12a4340bdad3f40805d2ae54e.png?q-sign-algorithm=sha1&q-ak=AKID372nLgqocp7HZjfQzNcyGOMTN3Xp6FEA&q-sign-time=1761030318%3B2076390318&q-key-time=1761030318%3B2076390318&q-header-list=host&q-url-param-list=&q-signature=8fa734e3921392d22b5b5c67f274254e6a7a0e5d)

### 重点一：精巧的 Chunk 结构与管理

Chunk 是内存分配和释放的基本单位，其数据结构设计得非常巧妙，以支持高效的合并与分割。

1. ​**Chunk 布局与复用机制**​
    
    - Chunk 分为**头部（`struct malloc_chunk`）​**​ 和**可用内存区域**。
        
    - 头部包含 `mchunk_prev_size`（前一个chunk的大小）和 `mchunk_size`（当前chunk的大小及标志位）。
        
    - 关键之处在于：当 chunk ​**空闲**时，其可用内存区域的一部分会被复用为 `fd`, `bk`等指针，用于链接到相应的 bin 中；当 chunk ​**已分配**时，这部分空间则完全交给用户程序使用。这种设计在保证功能的同时，最大限度地减少了内存开销。
        
    
2. ​**Chunk 的合并与分割**​
    
    - ​**合并**​：当释放一个 chunk 时，ptmalloc 会检查其相邻的前后 chunk 是否也为空闲状态。通过检查前一个 chunk 的 P 位（`PREV_INUSE`）和下一个 chunk 的头部信息，可以确定能否合并。合并操作能有效减少内存碎片。下图展示了向前合并的过程：
        
        ![](https://hunyuan-plugin-private-1258344706.cos.ap-nanjing.myqcloud.com/pdf_youtu/img/1761030298_97b09208f6e84eb4935e3cd9d48be65b.png?q-sign-algorithm=sha1&q-ak=AKID372nLgqocp7HZjfQzNcyGOMTN3Xp6FEA&q-sign-time=1761030326%3B2076390326&q-key-time=1761030326%3B2076390326&q-header-list=host&q-url-param-list=&q-signature=8910fd69cbb7bfe7b13a8cfc8a76fdd390c2d4ec)
        
    - ​**分割**​：当从一个大的空闲 chunk（如 top chunk）中分配内存时，如果该 chunk 的剩余部分足够大，则会将其分割。一部分分配给用户，剩余部分形成一个新的空闲 chunk，放回内存池。此过程如下图所示：
        
        ![](https://hunyuan-plugin-private-1258344706.cos.ap-nanjing.myqcloud.com/pdf_youtu/img/1761030298_d7b3a20fa324439ca44204eb0805472b.png?q-sign-algorithm=sha1&q-ak=AKID372nLgqocp7HZjfQzNcyGOMTN3Xp6FEA&q-sign-time=1761030328%3B2076390328&q-key-time=1761030328%3B2076390328&q-header-list=host&q-url-param-list=&q-signature=50e99c2e7ddc52879d1d6da4c295b38674e9b126)
        
    

### 重点二：多级 Bins 管理策略

为了高效管理不同大小的空闲 chunk，ptmalloc 使用了多种类型的 bins，其组织结构如下图所示，它们共同构成了一个高效的空闲内存查找系统：

![](https://hunyuan-plugin-private-1258344706.cos.ap-nanjing.myqcloud.com/pdf_youtu/img/1761030298_3500715e643e4eed8f608086ced28d4e.png?q-sign-algorithm=sha1&q-ak=AKID372nLgqocp7HZjfQzNcyGOMTN3Xp6FEA&q-sign-time=1761030329%3B2076390329&q-key-time=1761030329%3B2076390329&q-header-list=host&q-url-param-list=&q-signature=737ef7b0f7bdafb089bbca51187147d135b299b2)

1. ​**Fastbins**​：用于快速分配小内存（如 32-160 字节）。它是一个**单链表**数组，使用 LIFO（后进先出）策略，且 chunk 不会被合并，以加速分配速度。
    
2. ​**Unsortedbin**​：是一个特殊的**双向链表**。刚被释放的 chunk 会首先放入这里，相当于一个缓存层，以便在下次分配时能被快速复用。
    
3. ​**Smallbins**​：管理固定大小的 chunk（如 32-1024 字节）。它也是**双向链表**数组，但采用 FIFO（先进先出）策略。Smallbins 和 Fastbins 管理的尺寸范围有重叠，但策略不同。
    
4. ​**Largebins**​：管理大于 smallbin 尺寸的 chunk。每个 bin 管理一个**大小范围**内的 chunk。为了快速查找，largebin 中的 chunk 除了用 `fd/bk`指针组成一个循环链表外，还使用 `fd_nextsize/bk_nextsize`指针将不同大小的 chunk 按序索引，避免遍历相同大小的 chunk，提升搜索效率。其快速索引机制如下：
    
    ![](https://hunyuan-plugin-private-1258344706.cos.ap-nanjing.myqcloud.com/pdf_youtu/img/1761030298_b16be4986f934347ac770b8c5341e0b0.png?q-sign-algorithm=sha1&q-ak=AKID372nLgqocp7HZjfQzNcyGOMTN3Xp6FEA&q-sign-time=1761030336%3B2076390336&q-key-time=1761030336%3B2076390336&q-header-list=host&q-url-param-list=&q-signature=83ec3a69075ae8c752c8302164ab264ba4d59389)
    

### 重点三：高效查找 - Binmap 位图

为了避免在分配内存时遍历所有可能为空的 bin，ptmalloc 引入了 `binmap`位图机制。位图中的每一位对应 bins 数组中的一个 bin，标识该 bin 中是否包含空闲 chunk。这可以快速跳过大量空的 bin，显著提高分配效率。

![](https://hunyuan-plugin-private-1258344706.cos.ap-nanjing.myqcloud.com/pdf_youtu/img/1761030298_2e6f73eb8dda499ca8a351f0f677800c.png?q-sign-algorithm=sha1&q-ak=AKID372nLgqocp7HZjfQzNcyGOMTN3Xp6FEA&q-sign-time=1761030338%3B2076390338&q-key-time=1761030338%3B2076390338&q-header-list=host&q-url-param-list=&q-signature=ac6d356222e5b2b71e786c926a2745da49077ace)

### 重点四：malloc 和 free 的工作流程总结

1. ​**`malloc`流程**​：
    
    - 将用户请求的内存大小转换为 chunk 的实际大小（包括对齐）。
        
    - ​**第一步**​：尝试从 `fastbins`中查找合适大小的 chunk。
        
    - ​**第二步**​：如果大小属于 `smallbins`，则从对应的 `smallbin`中查找。
        
    - ​**第三步**​：遍历 `unsortedbin`，若找到精确匹配的 chunk 则分配；否则将 `unsortedbin`中的 chunk 根据大小重新归类到 `smallbins`或 `largebins`。
        
    - ​**第四步**​：从 `largebins`中查找（利用 `binmap`和 `fd_nextsize/bk_nextsize`）。
        
    - ​**第五步**​：若以上都失败，则从 `top chunk`分割。
        
    - ​**最后**​：如果 `top chunk`空间不足，则通过 `sbrk`或 `mmap`向操作系统申请新的内存。
        
    
2. ​**`free`流程**​：
    
    - 判断传入的指针是否为空。
        
    - 检查该 chunk 是否是通过 `mmap`分配的，如果是，则直接用 `munmap`释放。
        
    - 否则，对于小 chunk，将其放入 `fastbins`（此时不合并）。
        
    - 对于其他 chunk，先尝试与相邻的空闲 chunk ​**合并**，然后将合并后的 chunk 放入 `unsortedbin`中。
        
    

### 核心总结

ptmalloc 的设计精髓在于通过 ​**chunk 的复用与合并/分割机制**​ 来减少内存碎片，并通过 ​**多级 bins 缓存策略**​（fastbins -> smallbins -> unsortedbin -> largebins -> top chunk）在**分配速度**和**内存利用率**之间取得平衡。理解这些底层机制对于编写高性能、低内存消耗的 C/C++ 程序至关重要。