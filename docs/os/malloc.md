# Linux高性能编程：malloc主要原理
## 一、ptmalloc 基本架构

ptmalloc 是 glibc 中实现 `malloc`/`free`的内存管理模块，其核心结构包括：

- ​**`malloc_state`**​：全局内存分配区，包含 fastbins、top chunk、bins 等关键组件。
    
- ​**`malloc_chunk`**​：内存管理的基本单位，通过链表组织空闲内存块。
    

### 1.1 核心组件说明

#### （1）`malloc_state`结构

```
struct malloc_state {
    __libc_lock_define(, mutex);          // 互斥锁
    int flags;                           // 标志位
    mfastbinptr fastbinsY[NFASTBINS];    // 快速分配链表（16-160字节）
    mchunkptr top;                       // 顶部空闲内存块
    mchunkptr bins[NBINS * 2 - 2];       // 常规空闲链表（unsorted/small/large bins）
    unsigned int binmap[BINMAPSIZE];     // 空闲链表位图
    struct malloc_state *next;           // 多分配区链表指针
};
```

- ​**主分配区（main_state）​**​：通过 `brk`系统调用从堆内存申请内存。
    
- ​**非主分配区（thread_state）​**​：多线程环境下通过 `mmap`申请内存，减少锁竞争。
    

![](https://hunyuan-plugin-private-1258344706.cos.ap-nanjing.myqcloud.com/pdf_youtu/img/1761492707_0db858d2318d4b2b819efef075d8b6f2.png?q-sign-algorithm=sha1&q-ak=AKID372nLgqocp7HZjfQzNcyGOMTN3Xp6FEA&q-sign-time=1761492721%3B2076852721&q-key-time=1761492721%3B2076852721&q-header-list=host&q-url-param-list=&q-signature=77a44d37aa4e95c7ff34392d3dc0b09d857002bd)

#### （2）`malloc_chunk`结构

```
struct malloc_chunk {
    INTERNAL_SIZE_T mchunk_prev_size;  // 前一个 chunk 的大小
    INTERNAL_SIZE_T mchunk_size;      // 当前 chunk 大小（含标志位）
    struct malloc_chunk *fd;          // 链表后驱指针
    struct malloc_chunk *bk;          // 链表前驱指针
    // ...（large bins 专用指针）
};
```

- ​**chunk 头部**​：记录块大小与状态（A: 主分配区、M: mmap 分配、P: 前一个 chunk 使用中）。
    
- ​**空闲时**​：借用用户数据区存储链表指针。
    

![](https://hunyuan-plugin-private-1258344706.cos.ap-nanjing.myqcloud.com/pdf_youtu/img/1761492707_a52ce62492314a64a62f358c05819d68.png?q-sign-algorithm=sha1&q-ak=AKID372nLgqocp7HZjfQzNcyGOMTN3Xp6FEA&q-sign-time=1761492724%3B2076852724&q-key-time=1761492724%3B2076852724&q-header-list=host&q-url-param-list=&q-signature=18b3c0c6bea8b8e71d4bacbc0a8f6bcb41723e79)

### 1.2 空闲链表分类

- ​**fastbins**​：单链表存储 16-160 字节的小块内存（不合并碎片）。
    
- ​**bins**​：
    
    - ​**unsortedbin**​：临时缓存合并后的空闲 chunk。
        
    - ​**smallbins**​：双链表存储 32-1008 字节的 chunk。
        
    - ​**largebins**​：存储大于 1024 字节的 chunk，按大小排序。
        
    

![](https://hunyuan-plugin-private-1258344706.cos.ap-nanjing.myqcloud.com/pdf_youtu/img/1761492707_c6cd0acc65ed4e29a868e13fe46d9173.png?q-sign-algorithm=sha1&q-ak=AKID372nLgqocp7HZjfQzNcyGOMTN3Xp6FEA&q-sign-time=1761492725%3B2076852725&q-key-time=1761492725%3B2076852725&q-header-list=host&q-url-param-list=&q-signature=d12ec12ee0de03613ef12c6ff06f058ac7bd2509)

### 1.3 内存分配与释放流程

#### （1）`malloc`流程

1. 检查 fastbins → smallbins → unsortedbin → largebins 是否有匹配 chunk。
    
2. 若空闲链表无可用内存，从 top chunk 分割。
    
3. top chunk 不足时，通过 `brk`/`mmap`向内核申请内存。
    

![](https://hunyuan-plugin-private-1258344706.cos.ap-nanjing.myqcloud.com/pdf_youtu/img/1761492707_88e73e68ab684f0d8320f2b62b41db8b.png?q-sign-algorithm=sha1&q-ak=AKID372nLgqocp7HZjfQzNcyGOMTN3Xp6FEA&q-sign-time=1761492727%3B2076852727&q-key-time=1761492727%3B2076852727&q-header-list=host&q-url-param-list=&q-signature=04ed8a8b5989c0313beb76fb1520c6f87f96e2d8)

#### （2）`free`流程

1. 释放的 chunk 优先插入 fastbins（不合并）。
    
2. 大块内存或合并后 chunk 放入 unsortedbin，必要时合并后转移至 small/large bins。
    
3. 大块内存可能通过 `munmap`直接归还内核。
    

![](https://hunyuan-plugin-private-1258344706.cos.ap-nanjing.myqcloud.com/pdf_youtu/img/1761492707_040261668ff64e20a3dc69e1b1f472bd.png?q-sign-algorithm=sha1&q-ak=AKID372nLgqocp7HZjfQzNcyGOMTN3Xp6FEA&q-sign-time=1761492729%3B2076852729&q-key-time=1761492729%3B2076852729&q-header-list=host&q-url-param-list=&q-signature=bd2d5f2956d4b17d8ad95f278e76e84adde0bd64)

---

## 二、高并发场景下的问题

### 2.1 性能瓶颈

- ​**锁竞争**​：多线程通过同一分配区申请内存时需频繁加锁（观察 `futex`系统调用）。
    
- ​**内存碎片**​：长期占用的小块内存阻碍大块内存合并，导致有效内存减少。
    

### 2.2 测试代码与结果

#### （1）测试代码

```
#include <stdlib.h>
#include <stdio.h>
#include <time.h>
#include <pthread.h>
#include <stdatomic.h>

#define THREAD_NUM (8)       // 线程数
#define SIZE_THREHOLD (1024) // 单次申请内存上限
#define LEAK_THREHOLD (64)   // 内存泄露阈值（≤此值不释放）

atomic_int leak_size;        // 泄露内存统计

int get_size() {
    srand(time(0));
    return rand() % SIZE_THREHOLD;
}

void* do_malloc(void* arg) {
    while (1) {
        int size = get_size();
        char* p = malloc(size);
        if (size >= LEAK_THREHOLD) {
            free(p);
        } else {
            atomic_fetch_add(&leak_size, size);
            printf("leak size:%d KB\n", leak_size / 1024);
        }
    }
    return NULL;
}

int main() {
    atomic_init(&leak_size, 0);
    pthread_t th[THREAD_NUM];
    for (int i = 0; i < THREAD_NUM; i++) {
        pthread_create(&th[i], NULL, do_malloc, NULL);
    }
    // ...（等待线程结束）
}
```

#### （2）测试结果

- ​**锁竞争**​：`strace`追踪显示频繁调用 `futex`加锁。
    
- ​**内存碎片**​：
    
    - 初始泄露 1.8MB，系统可用内存 2724MB。
        
    - 泄露达 162MB 时，系统可用内存 2474MB，实际使用 250MB，​**内存碎片达 88MB**。
        
    

![](https://hunyuan-plugin-private-1258344706.cos.ap-nanjing.myqcloud.com/pdf_youtu/img/1761492707_76d8c68e1ff8465fa19ef32bbab8d7b1.png?q-sign-algorithm=sha1&q-ak=AKID372nLgqocp7HZjfQzNcyGOMTN3Xp6FEA&q-sign-time=1761492735%3B2076852735&q-key-time=1761492735%3B2076852735&q-header-list=host&q-url-param-list=&q-signature=8f2e0f61a4bdc3772d753bb336a3f8b281325a0d)

---

## 三、结论

- ​**适用场景**​：`malloc`/`free`适合并发要求低、内存申请频率不高的场景。
    
- ​**高并发场景问题**​：
    
    - 频繁加锁导致性能下降。
        
    - 内存碎片降低内存利用率。
        
    
- ​**解决方案**​：高并发场景建议使用**高效内存池（如 tcmalloc、jemalloc）​**​ 替代 ptmalloc。
    
