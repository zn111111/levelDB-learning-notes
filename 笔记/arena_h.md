## 内存池Arena类

**作用：**用于分配和释放内存。适用于频繁分配和释放内存的场景。可以避免频繁的系统调用、避免内存碎片（按需分配会导致外部内存碎片）和减少cookie（malloc会有cookie）。

**成员变量：**

```c++
char* alloc_ptr_; //指向空闲内存
size_t alloc_bytes_remaining_; //剩余空间
//存储着指向所有分配过的空间的指针
//一块空间用完了，就需要分配新的空间，然后将alloc_ptr_指向该空间，但是用完的空间是需要释放的，否则会导致内存泄漏，所有需要存起来
std::vector<char*> blocks_;
std::atomic<size_t> memory_usage_; //已经使用的堆区内存，不单单指的是内存池分配的内存，vector使用的内存也是
```

**构造函数：**

```c++
//默认构造
Arena()
    : alloc_ptr_(nullptr), alloc_bytes_remaining_(0), memory_usage_(0) {}

//删除拷贝构造和拷贝赋值

//析构函数
//vector对象blocks_里存储着所有分配的内存的指针，所有只需要释放该vector中的指针指向的空间即可
~Arena()
{
    for (size_t i = 0; i < blocks_.size(); i++) {
        delete[] blocks_[i];
    }
}
```

**成员函数Allocate：**

```c++
//分配空间，若当前块的空间足够则直接分配，否则重新分配
char* Allocate(size_t bytes)
{
    assert(bytes > 0);
    if (bytes <= alloc_bytes_remaining_) {
    char* result = alloc_ptr_;
    alloc_ptr_ += bytes;
    alloc_bytes_remaining_ -= bytes;
    return result;
    }
    return AllocateFallback(bytes);
}
```

**成员函数AllocateAligned：**

```c++
//内存对齐分配
char* AllocateAligned(size_t bytes)
{
    //按8字节对齐
    const int align = (sizeof(void*) > 8) ? sizeof(void*) : 8;
    //判断对齐数是否为2的k次幂
    static_assert((align & (align - 1)) == 0, "Pointer size should be a power of 2");
    //指针对8求余
    //不为0则没有按8字节对齐，分配内存时需要向后移动align - current_mod
    //位运算更快，所有求余用位运算
    size_t current_mod = reinterpret_cast<uintptr_t>(alloc_ptr_) & (align - 1);
    size_t slop = (current_mod == 0 ? 0 : align - current_mod);
    size_t needed = bytes + slop;
    char* result;
    if (needed <= alloc_bytes_remaining_) {
        result = alloc_ptr_ + slop;
        alloc_ptr_ += needed;
        alloc_bytes_remaining_ -= needed;
    } else {
        result = AllocateFallback(bytes); //空间不足，重新分配，当然不需要带上内存对齐的空间
    }
    assert((reinterpret_cast<uintptr_t>(result) & (align - 1)) == 0);
    return result;
}
```

**成员函数MemoryUsage：**

```c++
//获取已经使用的内存量
//memory_order_relaxed内存序，只保证原子操作，没有同步
size_t MemoryUsage() const
{
    return memory_usage_.load(std::memory_order_relaxed);
}
```

**私有成员函数AllocateFallback：**

```c++
//用于内存池的空间不足时分配新的空间
//如果要分配的空间大于kBlockSize / 4则按需分配，否则分配kBlockSize的空间
char* AllocateFallback(size_t bytes)
{
    if (bytes > kBlockSize / 4) {
    char* result = AllocateNewBlock(bytes);
    return result;
    }
    alloc_ptr_ = AllocateNewBlock(kBlockSize);
    alloc_bytes_remaining_ = kBlockSize;
    
    char* result = alloc_ptr_;
    alloc_ptr_ += bytes;
    alloc_bytes_remaining_ -= bytes;
    return result;
}
```

**私有成员函数AllocateNewBlock：**

```c++
//分配新的空间，返回空间的首地址，然后将地址存储在vector中便于后续对空间的释放
//更新已用空间，增加量为分配的空间和存储在vector中的指针的和
char* AllocateNewBlock(size_t block_bytes)
{
    char* result = new char[block_bytes];
    blocks_.push_back(result);
    memory_usage_.fetch_add(block_bytes + sizeof(char*),
                          std::memory_order_relaxed);
    return result;
}
```

