
---

# 📘 深度解析：FFmpeg 内存管理核心 (`AVBuffer` / `AVBufferPool`)

> 💡 **架构师视角**：FFmpeg 的内存管理采用了经典的 **引用计数 (Reference Counting)** + **内存池 (Memory Pool)** 模式。
> 
> - **`AVBuffer`** 是数据的物理载体，负责管理引用计数。
>     
> - **`AVBufferRef`** 是数据的逻辑句柄，用户通过它操作数据。
>     
> - **`AVBufferPool`** 是高性能场景下的内存复用池，避免频繁的 `malloc/free` 系统调用。
>     

---

## 1. 核心数据结构 (Data Structures)

### 1.1 `AVBuffer`：数据的物理容器 (Internal)

这是不对外暴露的内部结构，承载了实际的内存块和引用计数器。

```C++
struct AVBuffer {
    // 指向实际数据缓冲区的指针
    uint8_t *data; 
    size_t size; 

    /**
     * 🔥 核心机制：引用计数
     * 这是一个原子变量，保证线程安全。
     * - 创建时为 1。
     * - av_buffer_ref() 时 +1。
     * - av_buffer_unref() 时 -1。
     * - 减为 0 时，调用 free() 释放 data。
     */
    atomic_uint refcount;

    // 自定义释放回调，用于内存池回收或特殊内存管理
    void (*free)(void *opaque, uint8_t *data);
    void *opaque;

    // 标志位：BUFFER_FLAG_REALLOCATABLE, BUFFER_FLAG_NO_FREE 等
    int flags;
    int flags_internal;
};
```

### 1.2 `AVBufferRef`：用户的操作句柄 (Public)

用户持有的对象，类似于 C++ 的 `std::shared_ptr`。

```C++
typedef struct AVBufferRef {
    AVBuffer *buffer; // 指向内部管理的 AVBuffer

    /**
     * 数据指针。通常等于 buffer->data。
     * 只有当引用计数为 1 时，此数据才被认为是“可写”的。
     */
    uint8_t *data;
    size_t   size;
} AVBufferRef;
```

### 1.3 `BufferPoolEntry`：内存池的链表节点

这是内存池中被复用的单元。

```C++
typedef struct BufferPoolEntry {
    uint8_t* data; // 实际复用的内存块

    // 保存原始 AVBuffer 的回调信息，以便最终释放时使用
    void* opaque;
    void (*free)(void* opaque, uint8_t* data);

    AVBufferPool* pool; // 指向所属的池
    struct BufferPoolEntry* next; // 单链表结构

    // 预分配的 AVBuffer 结构体，复用时直接拿来用，省去 malloc(sizeof(AVBuffer))
    AVBuffer buffer;
} BufferPoolEntry;
```

### 1.4 `AVBufferPool`：内存池管理器

用于管理一堆固定大小的缓冲区。

```C++
struct AVBufferPool {
    AVMutex mutex;
    // 空闲缓冲区的链表头（栈结构，后进先出）
    BufferPoolEntry *pool;

    /**
     * 🔥 核心机制：池的生命周期管理
     * 初始为 1。
     * - get() 时 +1 (借出)。
     * - release() 时 -1 (归还)。
     * - 只有当所有借出的 buffer 都归还，且池本身被 uninit 时，才会真正释放池内存。
     */
    atomic_uint refcount;

    size_t size;
    void *opaque;
    
    // 自定义的内存分配器
    AVBufferRef* (*alloc)(size_t size);
    AVBufferRef* (*alloc2)(void *opaque, size_t size);
    void         (*pool_free)(void *opaque);
};
```

---

## 2. 核心 API 详解 (`AVBufferRef`)

### 2.1 基础操作

```C++
// 1. 分配内存
// 成功返回 AVBufferRef，引用计数为 1。
AVBufferRef *av_buffer_alloc(int size);

// 2. 包装现有内存
// 将现有的 data 包装成 AVBufferRef。
// 成功后，data 的所有权转移给 AVBuffer，用户不能再手动 free(data)。
AVBufferRef *av_buffer_create(uint8_t *data, int size,
                              void (*free)(void *opaque, uint8_t *data),
                              void *opaque, int flags);

// 3. 增加引用 (浅拷贝)
// 创建一个新的 AVBufferRef，指向同一个 AVBuffer，引用计数 +1。
AVBufferRef *av_buffer_ref(AVBufferRef *buf);

// 4. 释放引用
// 引用计数 -1。如果减为 0，则调用 buffer->free() 释放内存。
// buf 指针会被置为 NULL。
void av_buffer_unref(AVBufferRef **buf);
```

### 2.2 写时复制 (Copy-On-Write) 机制

```C++

// 检查是否只有当前一个引用。如果是，则可安全写入。
int av_buffer_is_writable(const AVBufferRef *buf);

// 确保 buffer 可写。
// 如果引用计数 > 1，会分配新的内存块，拷贝数据，并将当前 ref 指向新块（引用计数分离）。
int av_buffer_make_writable(AVBufferRef **buf);
```

---

## 3. 内存池机制 (`AVBufferPool`)

内存池的设计是为了解决频繁分配固定大小内存（如视频帧）带来的性能开销。

### 3.1 内存池操作 API

```C++
// 初始化池：设定每个 buffer 的大小
AVBufferPool *av_buffer_pool_init(int size, AVBufferRef* (*alloc)(int size));

// 从池中获取 buffer
// 优先从 pool 链表取，取不到则调用 alloc 分配新的
AVBufferRef *av_buffer_pool_get(AVBufferPool *pool);

// 销毁池
// 标记为可释放。实际释放要等到所有 buffer 都归还后。
void av_buffer_pool_uninit(AVBufferPool **pool);
```

### 3.2 源码级流程分析

#### A. 从池中获取 (`av_buffer_pool_get`)


```C++
AVBufferRef* av_buffer_pool_get(AVBufferPool* pool)
{
    AVBufferRef* ret;
    BufferPoolEntry* buf;

    ff_mutex_lock(&pool->mutex);
    buf = pool->pool; // 尝试从链表头取一个

    if (buf) {
        // --- 命中缓存 (Cache Hit) ---
        // 1. 从链表移除该节点
        pool->pool = buf->next;
        buf->next = NULL;

        // 2. 复用 BufferPoolEntry 中的数据
        // 注意：这里传入了自定义释放函数 pool_release_buffer
        ret = buffer_create(&buf->buffer, buf->data, pool->size,
                            pool_release_buffer, buf, 0);
        
        // 标记为 NO_FREE，因为数据还要还给池子，不能真释放
        if (ret) {
            buf->buffer.flags_internal |= BUFFER_FLAG_NO_FREE;
        }
    } else {
        // --- 未命中 (Cache Miss) ---
        // 3. 调用 alloc 分配新的，并包装成 BufferPoolEntry
        ret = pool_alloc_buffer(pool);
    }
    ff_mutex_unlock(&pool->mutex);

    // 借出计数 +1
    if (ret)
        atomic_fetch_add_explicit(&pool->refcount, 1, memory_order_relaxed);

    return ret;
}
```

#### B. 归还回池 (`pool_release_buffer`)

这是 `AVBuffer` 引用计数归零时调用的回调。

```C++
static void pool_release_buffer(void* opaque, uint8_t* data)
{
    BufferPoolEntry* buf = opaque;
    AVBufferPool* pool = buf->pool;

    // 1. 加锁，将 buffer 挂回链表头 (头插法)
    ff_mutex_lock(&pool->mutex);
    buf->next = pool->pool;
    pool->pool = buf;
    ff_mutex_unlock(&pool->mutex);

    // 2. 借出计数 -1
    // 如果计数为 1 (只剩池本身持有) 且池已被 uninit，则真正销毁池
    if (atomic_fetch_sub_explicit(&pool->refcount, 1, memory_order_acq_rel) == 1)
        buffer_pool_free(pool);
}
```

---

## 4. 进阶应用：`FFFramePool` (帧池)

`FFFramePool` 是 FFmpeg 内部基于 `AVBufferPool` 封装的高级结构，用于管理复杂的 `AVFrame`（可能包含多个 Plane，如 YUV）。

> ⚠️ **注意**：这不是公开 API，但在滤镜开发（如 scale, filter）中非常有用。

### 4.1 结构设计

```C++
struct FFFramePool {
    int width, height;
    int format;
    int align;
    
    // 一个视频帧可能需要多个 pool (例如 YUV420P 可能需要独立的 Y、U、V 池，或合用的池)
    // 但通常根据 linesize 和 height 计算出总大小，用一个 pool 管理一个 Plane
    int linesize[4];
    AVBufferPool* pools[4]; 
};
```

### 4.2 初始化流程 (`ff_frame_pool_video_init`)

1. **参数检查**：宽高、格式是否合法。
    
2. **计算 Linesize**：根据对齐要求 (`align`) 计算每行的步长。
    
3. **计算 Plane Size**：计算每个分量所需的连续内存大小。
    
4. **创建 Pool**：为每个非空的 Plane 创建一个 `AVBufferPool`。
    

### 4.3 获取帧流程 (`ff_frame_pool_get`)

1. `av_frame_alloc()` 创建空壳。
    
2. 设置宽、高、格式。
    
3. 遍历 4 个 Plane，分别从对应的 `pools[i]` 中调用 `av_buffer_pool_get()` 获取内存。
    
4. 将获取的 `AVBufferRef` 填入 `frame->buf[i]`，将数据指针填入 `frame->data[i]`。
    

---

## 🚀 总结 (Summary)

1. **原子引用计数**是核心，保证了多线程下的内存安全。
    
2. **AVBufferRef** 实现了**浅拷贝**，极大地提升了帧传递（如解码->滤镜->编码）的效率。
    
3. **AVBufferPool** 通过**链表复用**机制，消除了反复 `malloc` 大块内存（如 4K 帧）造成的性能抖动和碎片。
    
4. **自定义释放回调 (`pool_release_buffer`)** 是连接引用计数系统和内存池系统的桥梁。