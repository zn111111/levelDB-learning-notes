**匿名命名空间：**在cache.cc源文件中，leveldb命名空间内还有一个匿名命名空间。我猜应该是因为在该源文件中定义了静态全局变量，如果不加命名空间的话，该头文件被包裹到其它文件中，只要命名空间也是leveldb，那么静态全局变量就可见，所以要加匿名命名空间。

**非成员函数NewLRUCache：**创建LRU缓存。LRU就是最近被访问的会被保存，最长不被访问的会被丢弃。

## 结构体LRUHandle

**作用：**作为哈希表中链表的结点和缓存类中的in_use和lru链表的结点。

```c++
struct LRUHandle {
    void* value;
    //用于释放结点中指向动态分配空间的指针
    void (*deleter)(const Slice&, void* value);
    //哈希表中的next使用该指针
    LRUHandle* next_hash;
    //缓存类中的in_use和lru链表的next使用该指针
    LRUHandle* next;
    LRUHandle* prev;
    //需要耗费的缓存空间
    size_t charge;
    size_t key_length;
    //是否在缓存中
    bool in_cache;
    //引用数
    uint32_t refs;
    //哈希值
    uint32_t hash;
    //存放key，分配空间会多分配key.size() - 1，key_data就是该块空间的首地址
    char key_data[1];
    
    //获取key
    Slice key() const {
        //判断是否是头结点，头结点是空结点
        assert(next != this);
        
        return Slice(key_data, key_length);
    }
};
```



## HandleTable类

**作用：**自己实现的哈希表，作者说比编译器内置哈希表更快一点。

**构造函数：**默认构造和默认析构。

```c++
//默认构造
//分配篮子默认长度
HandleTable() : length_(0), elems_(0), list_(nullptr) { Resize(); }

//析构函数
//删除篮子的空间
~HandleTable() { delete[] list_; }
```

**成员函数Lookup：**调用FindPointer查找key和hash对应的结点，没找到就返回nullptr。

```c++
LRUHandle* Lookup(const Slice& key, uint32_t hash) {
    return *FindPointer(key, hash);
}
```

**成员函数Insert：**如果待插入的结点不存在，则插入到尾部，返回nullptr。如果已存在，则使用待插入的结点替换掉旧的结点，返回指向旧的结点的指针。

```c++
LRUHandle* Insert(LRUHandle* h) {
    //查找结点是否存在
    LRUHandle** ptr = FindPointer(h->key(), h->hash);
    LRUHandle* old = *ptr;
    //如果为nullptr，则待插入的结点的next为nullptr
    //如果不为nullptr，则结点已经存在，则替换掉旧的结点，将待插入的结点的next改为旧结点的next
    h->next_hash = (old == nullptr ? nullptr : old->next_hash);
    //ptr指向的是上一个结点的next指针位置
    //将上一个结点的next修改为待插入的结点
    *ptr = h;
    if (old == nullptr) {
        //如果结点不存在，则插入的是新结点，元素个数++
        ++elems_;
        //如果元素个数大于篮子的长度，进行扩容
        if (elems_ > length_) {
            Resize();
        }
    }
    //如果插入的是新结点，则返回nullptr
    //否则返回指向旧结点的指针
    return old;
}
```

**成员函数Remove：**删除结点。如果结点不存在则返回nullptr，如果已存在则返回指向删除结点的指针。

```c++
LRUHandle* Remove(const Slice& key, uint32_t hash) {
    //查询结点是否存在
    LRUHandle** ptr = FindPointer(key, hash);
    LRUHandle* result = *ptr;
    //如果结点存在，将上一个结点的next改为待删除结点的next，然后元素个数--
    if (result != nullptr) {
        *ptr = result->next_hash;
        --elems_;
    }
    //如果结点不存在则返回nullptr
    //否则返回指向删除结点的指针
    return result;
}
```

**成员变量：**

```c++
//应该是篮子的长度
uint32_t length_;
//应该是哈希表中元素的个数
uint32_t elems_;
//应该指向的是堆区分配的数组（也就是篮子），数组中存储的是LRUHandle指针LRUHandle*
LRUHandle** list_;
```

**私有成员函数FindPointer：**找到key和hash值所对应的结点。

```c++
//没有找到对应的结点就返回nullptr
LRUHandle** FindPointer(const Slice& key, uint32_t hash) {
    //获取篮子中的指针，指向第一个结点的指针
    LRUHandle** ptr = &list_[hash & (length_ - 1)];
    //只要没有找到对应的结点就继续往下找
    while (*ptr != nullptr && ((*ptr)->hash != hash || key != (*ptr)->key())) {
        //跳转到下一个结点
        ptr = &(*ptr)->next_hash;
    }
    return ptr;
}
```

**私有成员函数Resize：**用于对哈希表进行扩容。哈希表扩容，篮子扩容两倍，然后重新计算每个元素存放的位置。

```c++
//篮子初始长度为4，当哈希表中元素的个数大于篮子的长度时进行扩容，将篮子扩容两倍，并重新计算元素存放的位置
void Resize() {
    //篮子初始长度4
    uint32_t new_length = 4;
    //当篮子中元素的个数大于篮子的长度时进行扩容，篮子扩容两倍
    while (new_length < elems_) {
        new_length *= 2;
    }
    //分配篮子的空间
    LRUHandle** new_list = new LRUHandle*[new_length];
    //空间初始化为0，这样每个位置都为nullptr
    memset(new_list, 0, sizeof(new_list[0]) * new_length);
    //统计元素个数
    uint32_t count = 0;
    //重新计算元素的位置，for循环遍历每一个篮子，while循环遍历链表中的元素，这样可以遍历所有元素
    //采用头插法，也就是先把尾弄好，最后一个元素的next为nullptr，然后在他的前面插入
    for (uint32_t i = 0; i < length_; i++) {
        //获取篮子中的指针，指向链表中的第一个结点
        LRUHandle* h = list_[i];
        //遍历链表
        while (h != nullptr) {
        //保存下一个结点的地址
        LRUHandle* next = h->next_hash;
        //当前结点的获取哈希值，以求新的哈希值，哈希值就是元素存放的位置，也就是哪一个篮子
        uint32_t hash = h->hash;
        //hash & (new_length - 1)是对new_length求余
        //也就是哈希值小于新篮子的长度就不变，大于就求余获取新的哈希值
        LRUHandle** ptr = &new_list[hash & (new_length - 1)];
        //设置当前结点的下一个结点，若为第一个结点则next为nullptr
        h->next_hash = *ptr;
        //将当前结点的地址存储在篮子中，形成链表
        *ptr = h;
        //跳转到下一个结点
        h = next;
        //结点数++
        count++;
        }
    }
    //确定所有结点都已访问
    assert(elems_ == count);
    //删除旧篮子
    delete[] list_;
    //指向新篮子
    list_ = new_list;
    //新篮子的长度
    length_ = new_length;
}
```

## LRUCache类

**构造函数：**

```c++
//默认构造
//两个结构体初始化为循环
LRUCache()
{
    lru_.next = &lru_;
    lru_.prev = &lru_;
    in_use_.next = &in_use_;
    in_use_.prev = &in_use_;
}

//析构函数，释放空间
//整个LRUCache对象需要释放的只有in_use和lru链表
//LRUCache对象要被销毁，说明肯定没有结点在被客户端使用，也就是in_use链表为空，不需要释放，只需要释放lru链表
~LRUCache()
{
    //in_use链表为空
    assert(in_use_.next == &in_use_);
    //遍历lru链表
    for (LRUHandle* e = lru_.next; e != &lru_;) {
    //保存下一个结点
    LRUHandle* next = e->next;
    //结点在缓存中
    assert(e->in_cache);
    //设为不在缓存中
    e->in_cache = false;
    //引用为1，不被客户端使用
    assert(e->refs == 1);
    //取消引用，删除结点
    Unref(e);
    //跳转到下一个结点
    e = next;
    }
}
```

**成员函数SetCapacity：**设置缓存的容量，因为缓存的内部是哈希表，空间是会一致扩容的。所以容量需要提供接口方便设置，而不是在构造函数中写死。

```c++
void SetCapacity(size_t capacity) { capacity_ = capacity; }
```

**成员函数Insert：**向缓存中插入新的结点。

```c++
Cache::Handle* Insert(const Slice& key, uint32_t hash, void* value, size_t charge,
                      void (*deleter)(const Slice& key, void* value))
{
    //加锁
    MutexLock l(&mutex_);
    //分配结点的空间，多分配- 1 + key.size()是因为，结点的结构体中对于key只分配了一个容量为1的char数组
    //所以额外分配- 1 + key.size()来存放key
    LRUHandle* e = reinterpret_cast<LRUHandle*>(malloc(sizeof(LRUHandle) - 1 + key.size()));
    e->value = value;
    e->deleter = deleter;
    e->charge = charge;
    e->key_length = key.size();
    e->hash = hash;
    e->in_cache = false;
    e->refs = 1;
    //将key拷贝过来
    std::memcpy(e->key_data, key.data(), key.size());
    
    //缓存容量不为0，则将结点插入到缓存的哈希表中，同时插入到in_use链表中
    if (capacity_ > 0) {
        //插入结点到缓存，该结点肯定正在被使用，所以引用++
        e->refs++;
        e->in_cache = true;
        //将结点插入到in_use链表中
        LRU_Append(&in_use_, e);
        usage_ += charge;
        //将结点插入到哈希表中，同时删除旧的结点
        FinishErase(table_.Insert(e));
    } else {
        //缓存容量为0则关闭缓存
        e->next = nullptr;
    }
    //如果缓存已用空间大于缓存容量则删除旧的没有被客户端使用的结点
    while (usage_ > capacity_ && lru_.next != &lru_) {
        //lru链表中第一个结点是罪旧的结点，最后一个结点是最新的结点
        LRUHandle* old = lru_.next;
        assert(old->refs == 1);
        //将结点从哈希表中移除，从lru链表中移除并删除结点
        bool erased = FinishErase(table_.Remove(old->key(), old->hash));
        if (!erased) {
            assert(erased);
        }
    }
    
    return reinterpret_cast<Cache::Handle*>(e);
}
```

**成员函数Lookup：**查找key。

```c++
Cache::Handle* Lookup(const Slice& key, uint32_t hash)
{
    //加锁
    MutexLock l(&mutex_);
    //在哈希表中查找
    LRUHandle* e = table_.Lookup(key, hash);
    if (e != nullptr) {
        //找到了，引用++
        Ref(e);
    }
    return reinterpret_cast<Cache::Handle*>(e);
}
```

**成员函数Release：**解除引用，引用--。

```c++
void Release(Cache::Handle* handle)
{
    //加锁
    MutexLock l(&mutex_);
    //解除引用
    Unref(reinterpret_cast<LRUHandle*>(handle));
}
```

**成员函数Erase：**从缓存中删除结点。

```c++
void Erase(const Slice& key, uint32_t hash)
{
    //加锁
    MutexLock l(&mutex_);
    //某个结点可以删除，表示该结点不被客户端引用，所以该结点肯定在lru链表中
    //首先将结点从哈希表中移除，该操作只是改变指针指向，并不会删除结点
    //然后将结点从lru链表中移除，再删除结点，所以调用FinishErase
    FinishErase(table_.Remove(key, hash));
}
```

**成员函数Prune：**清空lru链表

```c++
void Prune()
{
    //获取锁
    MutexLock l(&mutex_);
    //清空lru
    while (lru_.next != &lru_) {
    LRUHandle* e = lru_.next;
    assert(e->refs == 1);
    //删除结点
    bool erased = FinishErase(table_.Remove(e->key(), e->hash));
        if (!erased) {
            assert(erased);
        }
    }
}
```

**成员函数TotalCharge：**获取缓冲区总开销。

```c++
size_t TotalCharge() const {
    //加锁
    MutexLock l(&mutex_);
    //获取缓冲区开销
    return usage_;
}
```

**私有成员函数LRU_Remove：**将结点从链表中移除。不一定得是lru链表。

```c++
void LRU_Remove(LRUHandle* e)
{
    e->next->prev = e->prev;
    e->prev->next = e->next;
}
```

**私有成员函数LRU_Append：**将结点e插入到结点list的前面。

```c++
void LRU_Append(LRUHandle* list, LRUHandle* e)
{
    e->next = list;
    e->prev = list->prev;
    e->prev->next = e;
    e->next->prev = e;
}
```

**私有成员函数Ref：**增加结点的引用。如果增加前结点的引用为1并且在缓存中，则将结点加入到in_use链表中。

```c++
void Ref(LRUHandle* e)
{
    //如果结点的引用为1，并且在缓存中，将结点插入到in_use链表中
    if (e->refs == 1 && e->in_cache) {
        LRU_Remove(e);
        LRU_Append(&in_use_, e);
    }
    //引用++
    e->refs++;
}
```

**私有成员函数Unref：**引用减1。如果引用等于0旧删除结点，如果等于1就将结点从in_use中删除，并插入到lru中，如果大于1就不用管，说明结点正在被客户端使用。

```c++
void LRUCache::Unref(LRUHandle* e) {
    //引用必须大于0
    assert(e->refs > 0);
    //取出一个引用
    e->refs--;
    //如果引用等于0，说明结点可以被删除
    if (e->refs == 0) {
    //结点不在缓存中
    assert(!e->in_cache);
    //回调函数，释放结点中指向动态分配内存的指针
    (*e->deleter)(e->key(), e->value);
    //释放结点的空间
    free(e);
    //如果结点在缓存中，并且引用等于1
    //则将结点从in_use链表中删除，并插入到lru链表中
    } else if (e->in_cache && e->refs == 1) {
        LRU_Remove(e);
        LRU_Append(&lru_, e);
    }
}
```

**私有成员函数FinishErase：**删除结点。

```c++
//EXCLUSIVE_LOCKS_REQUIRED(mutex_)表示执行该函数时，线程必须已经取得锁，否则会出错
bool FinishErase(LRUHandle* e) EXCLUSIVE_LOCKS_REQUIRED(mutex_)
{
    if (e != nullptr) {
    assert(e->in_cache);
    //结点虽然已经从哈希表中移除，但是还在lru链表中
    //所以从链表中移除
    LRU_Remove(e);
    e->in_cache = false;
    usage_ -= e->charge;
    //解除引用，删除结点
    Unref(e);
    }
    return e != nullptr;
}
```

**成员变量：**

```c++
//互斥锁
mutable port::Mutex mutex_;
//GUARDED_BY是数据成员的属性，读可以同时进行，写必须互斥
size_t usage_ GUARDED_BY(mutex_);
//头节点，没被客户端使用的结点存储在该双向循环链表中
//第一个结点是最新的结点，最后一个结点是最旧的结点
LRUHandle lru_ GUARDED_BY(mutex_);
//头结点，被客户端使用的结点存储在这里
//无序
LRUHandle in_use_ GUARDED_BY(mutex_);
//哈希表
HandleTable table_ GUARDED_BY(mutex_);
```

## ShardedLRUCache类

**作用：**将一块缓存分为16块LRU缓存分片。将数据插入到缓存中时，先计算应该插入到哪个LRU分片，然后插入到对应的LRU分片中。查找也是类似。

**构造函数：**

```c++
//有参构造函数
//为每一个LRUCache设置容量
explicit ShardedLRUCache(size_t capacity) : last_id_(0) {
    const size_t per_shard = (capacity + (kNumShards - 1)) / kNumShards;
    for (int s = 0; s < kNumShards; s++) {
        shard_[s].SetCapacity(per_shard);
    }
}

//析构函数
~ShardedLRUCache() override {}
```

**成员函数Insert：**将结点插入到缓存中。

```c++
//1、计算hash值
//2、计算出应该插入到哪个LRU分片，然后插入到对应的LRU分片
Handle* Insert(const Slice& key, void* value, size_t charge,
               void (*deleter)(const Slice& key, void* value)) override {
    const uint32_t hash = HashSlice(key);
    return shard_[Shard(hash)].Insert(key, hash, value, charge, deleter);
}
```

**成员函数Lookup：**查找key。

```c++
//1、获取hash值
//2、计算出key对应结点存放在哪个LRU缓存分片中，然后取对应LRU分片查找。
Handle* Lookup(const Slice& key) override {
    const uint32_t hash = HashSlice(key);
    return shard_[Shard(hash)].Lookup(key, hash);
}
```

**成员函数Release：**释放结点引用。

```c++
//1、取出hash值，调用对应LRU分片的接口
void Release(Handle* handle) override {
    LRUHandle* h = reinterpret_cast<LRUHandle*>(handle);
    shard_[Shard(h->hash)].Release(handle);
}
```

**成员函数Erase：**删除结点。

```c++
//1、计算出hash
//2、调用对应LRU分片的接口
void Erase(const Slice& key) override {
    const uint32_t hash = HashSlice(key);
    shard_[Shard(hash)].Erase(key, hash);
}
```

**成员函数Value：**获取结点的value。

```c++
void* Value(Handle* handle) override {
    return reinterpret_cast<LRUHandle*>(handle)->value;
}
```

**成员函数NewId：**不知道是干嘛用的。

```c++
uint64_t NewId() override {
    MutexLock l(&id_mutex_);
    return ++(last_id_);
}
```

**成员函数Prune：**清空所有LRU分片的lru链表。

```c++
void Prune() override {
    for (int s = 0; s < kNumShards; s++) {
        shard_[s].Prune();
    }
}
```

**成员函数TotalCharge：**计算所有LRU分片的缓存使用量

```c++
size_t TotalCharge() const override {
    size_t total = 0;
    for (int s = 0; s < kNumShards; s++) {
        total += shard_[s].TotalCharge();
    }
    return total;
}
```

**私有成员函数HashSlice：**

```c++
//通过key获取hash值
//Hash是一个哈希算法
static inline uint32_t HashSlice(const Slice& s) {
    return Hash(s.data(), s.size(), 0);
}
```

**私有成员函数Shard：**获取结点在哪个LRU缓存分片中。

```c++
//为什么这么算就不知道了
static uint32_t Shard(uint32_t hash) { return hash >> (32 - kNumShardBits); }
```

**非成员函数NewLRUCache：**动态创建ShardedLRUCache对象。

```c++
Cache* NewLRUCache(size_t capacity) { return new ShardedLRUCache(capacity); }
```

## Cache类

**作用：**作为父类，用于被继承，里面声明了很多纯虚函数。

