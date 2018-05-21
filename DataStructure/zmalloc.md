# 内存分配

## Redis对内存分配的封装
在内存分配上，Redis对C语言的内存分配函数进行了进一步的封装，以满足利用内存池等手段提高内存分配的性能；以便于掌握更多的内存信息，便于Redis虚拟内存(VM)等功能中决定何时将数据swap到磁盘。
Redis通过自己的方法管理内存,，主要方法有`zmalloc()`,`zrealloc()`，`zcalloc()`和`zfree()`, 分别对应C中的`malloc()`, `realloc()`、`calloc()`和`free()`。相关代码在zmalloc.h和zmalloc.c中。

C语言中系统中常见的内存分配函数功能如下：

`malloc()`分配一块指定大小的内存区域，并返回指向区域开头的指针，若分配失败，则返回NULL。

`calloc()`与`malloc()`一样，分配一块指定大小的内存区域，成功时返回区域头指针，失败返回NULL。区别在于，`calloc()`的输入参数为count和size，即分配的项的数目及每一项的大小，形如`malloc(size)`和`calloc(count,size)`。`calloc()`在成功分配内存空间后，会将空间内所有值置0。

`realloc()`修改已分配的内存块的大小。若已分配的内存块后没有足够的空间用于扩展内存块，则重新申请一块满足需要的内存块，并将旧的数据拷贝到新位置，释放旧的内存块，返回指向新的内存块的指针；否则直接扩展原有的内存块。若分配失败，返回NULL。

`free()`释放已分配的内存块。

## 代码分析
在头文件zmalloc.h中可以看到Redis封装后定义的的API。
```c
void *zmalloc(size_t size); //
void *zcalloc(size_t size);
void *zrealloc(void *ptr, size_t size);
void zfree(void *ptr);
char *zstrdup(const char *s);
size_t zmalloc_used_memory(void);
void zmalloc_enable_thread_safeness(void);
void zmalloc_set_oom_handler(void (*oom_handler)(size_t));
float zmalloc_get_fragmentation_ratio(size_t rss);
size_t zmalloc_get_rss(void);
size_t zmalloc_get_private_dirty(void);
void zlibc_free(void *ptr);
```
在文件zmalloc.c中还定义了几个很重要的变量。
```c
static size_t used_memory = 0;  //全局维护的变量，以此来获取系统使用的内存大小
static int zmalloc_thread_safe = 0;  //线程安全状态位：区分线程安全和非线程安全
pthread_mutex_t used_memory_mutex = PTHREAD_MUTEX_INITIALIZER;  //初始化了一个互斥锁
```
**默认内存溢出函数zmalloc_default_oom()**

当出现oom时调用此函数，函数比较简单，先向文件流写入错误信息然后刷新缓冲区，然后从调用的地方跳出。
```c
static void zmalloc_default_oom(size_t size) {
    fprintf(stderr, "zmalloc: Out of memory trying to allocate %zu bytes\n",
        size);
    fflush(stderr);
    abort();
}
```

**内存分配函数zmalloc()**

Redis的内存分配函数完成了空间的分配，此外还需要维护全局变量used_memory，因此每次分配释放内存都要进行更新
```c

void *zmalloc(size_t size) {
    void *ptr = malloc(size+PREFIX_SIZE);  //实际还是调用malloc，不过申请的空间比size=size+PREFIX_SIZE

    if (!ptr) zmalloc_oom_handler(size);  //分配失败，表示内存溢出直接调用内存溢出函数

//更新使用的内存变量used_memory,不再列出代码
//如果是线程安全模式，对变量used_memory_mutex加锁pthread_mutex_lock，然后再更新used_memory
//如果是非线程安全模式，直接更新变量used_memory
#ifdef HAVE_MALLOC_SIZE
    update_zmalloc_stat_alloc(zmalloc_size(ptr));
    return ptr;
#else
    *((size_t*)ptr) = size;
    update_zmalloc_stat_alloc(size+PREFIX_SIZE);
    return (char*)ptr+PREFIX_SIZE;
#endif
}
```

**内存分配函数zcalloc()**
```c
void *zcalloc(size_t size) {
    //calloc()和malloc()参数不一样，效果是一样的，分配[size+PREFIX_SIZE]*1的空间
    void *ptr = calloc(1, size+PREFIX_SIZE);

    if (!ptr) zmalloc_oom_handler(size);
#ifdef HAVE_MALLOC_SIZE
    update_zmalloc_stat_alloc(zmalloc_size(ptr));
    return ptr;
#else
    *((size_t*)ptr) = size;
    update_zmalloc_stat_alloc(size+PREFIX_SIZE);
    return (char*)ptr+PREFIX_SIZE;
#endif
}
```

**内存分配函数zrealloc()**
```c
void *zrealloc(void *ptr, size_t size) {
#ifndef HAVE_MALLOC_SIZE
    void *realptr;
#endif
    size_t oldsize;
    void *newptr;

    if (ptr == NULL) return zmalloc(size);
#ifdef HAVE_MALLOC_SIZE
    oldsize = zmalloc_size(ptr);
    newptr = realloc(ptr,size);
    if (!newptr) zmalloc_oom_handler(size);

    update_zmalloc_stat_free(oldsize);
    update_zmalloc_stat_alloc(zmalloc_size(newptr));
    return newptr;
#else
    realptr = (char*)ptr-PREFIX_SIZE;
    oldsize = *((size_t*)realptr);
    newptr = realloc(realptr,size+PREFIX_SIZE);
    if (!newptr) zmalloc_oom_handler(size);

    *((size_t*)newptr) = size;
    update_zmalloc_stat_free(oldsize);
    update_zmalloc_stat_alloc(size);
    return (char*)newptr+PREFIX_SIZE;
#endif
}
```

**内存释放函数zfree()**

内存释放函数也需要更新used_memory，和内存分配函数是相反的，就是用used_memory减去释放的内存的大小。
```c
void zfree(void *ptr) {
#ifndef HAVE_MALLOC_SIZE
    void *realptr;
    size_t oldsize;
#endif

    if (ptr == NULL) return;
#ifdef HAVE_MALLOC_SIZE
    update_zmalloc_stat_free(zmalloc_size(ptr));
    free(ptr);
#else
    realptr = (char*)ptr-PREFIX_SIZE;
    oldsize = *((size_t*)realptr);
    update_zmalloc_stat_free(oldsize+PREFIX_SIZE);
    free(realptr);
#endif
}
```

****
```c
```
