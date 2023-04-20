## 内存序

**指令重排：**为了提高计算机资源利用率和性能，编译器和CPU会对指令进行重排。这在单线程环境中没有任何问题，但是在多线程环境中会出问题。

```c++
int a = 1;
int b = 2;

//可能会被重排为
int b = 2;
int a = 1;
```

**C++11中的六种内存序：**可以根据自己的需求使用不同的同步程度（同步就是某条指令必须在某条指令之前执行，不同步就是乱序的，随便哪条先执行都行）。

**memory_order_relaxed：**只保证原子操作，不保证同步关系。

**memory_order_consume：**只保证具有依赖关系的操作具有同步关系。

```c++
//memory_order_release可以保证n的赋值一定在memory_order_release之前执行
//memory_order_consume指令后的读取操作与p有依赖关系，因此，执行n = *p时，memory_order_release前的指令可见
std::atomic<int*> val;
int n = 10;
val.store(&n, memory_order_release);


int* p = val.load(std::memory_order_consume);
n = *p;
```

**memory_order_acquire：**配合memory_order_release使用，可以保证读取值的时候，memory_order_release之前的指令可见。

```c++
//执行val.load(std::memory_order_acquire)指令时，val.store(&n, memory_order_release)之前的指令是可见的
std::atomic<int*> val;
int n = 10;
val.store(&n, memory_order_release);


int* p = val.load(std::memory_order_acquire);
```

**memory_order_release：**保证memory_order_release之前的指令一定在memory_order_release之前执行，需配合memory_order_acquire使用。

**下面两个比较复杂，用到时再研究：**

memory_order_acq_rel

memory_order_seq_cst

## Node类中使用的atomic模板类介绍

**作用：**

**成员函数store：**修改包含的值，原子操作。

**成员函数load：**获取包含的值。

## SkipList类中的节点类Node

```c++
template <typename Key, class Comparator>
struct SkipList<Key, Comparator>::Node {
  explicit Node(const Key& k) : key(k) {}

  Key const key;

  //获取第n层的下一个结点的地址，memory_order_acquire保证同步
  Node* Next(int n) {
    assert(n >= 0);

    return next_[n].load(std::memory_order_acquire);
  }
    
  //设置第n层的下一个结点，std::memory_order_release保证同步    
  void SetNext(int n, Node* x) {
    assert(n >= 0);

    next_[n].store(x, std::memory_order_release);
  }

  //获取第n层的下一个结点，memory_order_relaxed不保证同步，效率更高
  Node* NoBarrier_Next(int n) {
    assert(n >= 0);
    return next_[n].load(std::memory_order_relaxed);
  }
  
  //设置第n层的下一个结点，memory_order_relaxed不保证同步，效率更高
  void NoBarrier_SetNext(int n, Node* x) {
    assert(n >= 0);
    next_[n].store(x, std::memory_order_relaxed);
  }

 private:
  //next数组，一个结点在不同层有不同的next结点
  //之所以数组长度为1是因为在分配结点空间时不仅分配该节点的大小，还要多分配（height - 1）个std::atomic<Node*>大小，
  //height是这个结点的level，这么做是为了减小Node结点的大小
    
  //如图1所示
  //之所以next_可以访问0以外的下标是因为NewNode分配结点空间时，多分配了（height - 1）个std::atomic<Node*>空间，
  //next_是next_[1]这个数组的首地址，也就是这块连续空间的首地址，因此可以访问0以外的下标
  std::atomic<Node*> next_[1];
};
```

![NewNode分配的空间](D:\编程语言相关学习\C++学习\项目\leveldb\笔记\笔记里的图\NewNode分配的空间.png)

图1 NewNode分配的空间

## SkipList类中的迭代器类Iterator

**作用：**遍历跳表。

**为什么定义在类内：**可以使用类的私有接口，估计是因为这个。

**成员变量：**

```c++
const SkipList* list_; //指向跳表的指针
Node* node_; //指向跳表中结点的指针
```

**构造函数：**

```c++
//有参构造
explicit Iterator(const SkipList* list)
{
    list_ = list;
  	node_ = nullptr;
}
```

**成员函数Valid：**

```c++
//判断迭代器是否有效
bool Valid() const
{
    return node_ != nullptr;
}
```

**成员函数key：**

```c++
//获取key
const Key& key() const
{
    assert(Valid());
  	return node_->key;
}
```

**成员函数Next：**

```c++
//将迭代器指向结点第0层（level1）的下一个结点的位置
void Next()
{
    assert(Valid());
  	node_ = node_->Next(0);
}
```

**成员函数Prev：**

```c++
//获取当前结点的前驱（第0层）
void Prev()
{
    assert(Valid());
    node_ = list_->FindLessThan(node_->key);
    if (node_ == list_->head_) {
        node_ = nullptr;
    }
}
```

**成员函数Seek：**

```c++
//跳转到大于target的最小的结点
void Seek(const Key& target)
{
    node_ = list_->FindGreaterOrEqual(target, nullptr);
}
```

**成员函数SeekToFirst：**

```c++
//跳转到跳表中的最小的结点（第0层的第一个结点）
void SeekToFirst()
{
    node_ = list_->head_->Next(0);
}
```

**成员函数SeekToLast：**

```c++
//跳转到跳表中的最大的结点（第0层的最后一个结点）
void SeekToLast()
{
    node_ = list_->FindLast();
    if (node_ == list_->head_) {
    node_ = nullptr;
    }
}
```

## 模板类SkipList

**作用：**内存中的Memory Table就是用跳表实现的。

**成员变量：**

```c++
Comparator const compare_;
Arena* const arena_; //为结点分配空间
Node* const head_; //指向头节点的指针
std::atomic<int> max_height_; //当前层数
Random rnd_;
```

**构造函数和运算符重载：**

```c++
//有参构造
//成员变量初始化，将所以层的头节点的下一个结点设为null
explicit SkipList(Comparator cmp, Arena* arena)
    : compare_(cmp),
      arena_(arena),
      head_(NewNode(0 /* any key will do */, kMaxHeight)),
      max_height_(1),
      rnd_(0xdeadbeef) {
  for (int i = 0; i < kMaxHeight; i++) {
    head_->SetNext(i, nullptr);
  }
}

//拷贝构造和拷贝赋值删除，无法拷贝
```

**成员函数Insert：**

```c++
//插入新的结点
//首先找到key的所有前驱存储在prev中
//然后计算出高度，也就是插入新的结点后高度是否需要增长
//如果高度增长了，新增的level的key的前驱就是head，并更新当前level值max_height_
//然后创建新的结点，新结点level为height，因为不同结点有不同的level，计算好后就不再变，最高为kMaxHeight
//然后设置新结点的后继，然后更新前驱的后继
void Insert(const Key& key)
{
    Node* prev[kMaxHeight]; //用于存储key的前驱
    Node* x = FindGreaterOrEqual(key, prev);

  	assert(x == nullptr || !Equal(key, x->key)); //确保key不重复

  	int height = RandomHeight(); //计算当前结点的level
  	if (height > GetMaxHeight()) {
        for (int i = GetMaxHeight(); i < height; i++) { //新增的level的key的前驱都为head
            prev[i] = head_;
        }
 
        //memory_order_relaxed只原子不同步，作者说可以这样那就这样
        max_height_.store(height, std::memory_order_relaxed); //更新当前level值max_height_
    }

    x = NewNode(key, height); //创建新节点
    for (int i = 0; i < height; i++) {
    x->NoBarrier_SetNext(i, prev[i]->NoBarrier_Next(i)); //设置新结点的后继
    prev[i]->SetNext(i, x); //设置前驱结点的后继
  }
}
```

**成员函数Contains：**

```c++
//判断跳表中是否有key结点存在
bool Contains(const Key& key) const
{
    Node* x = FindGreaterOrEqual(key, nullptr); //返回的结点可能大于key，也可能等于key
    if (x != nullptr && Equal(key, x->key)) {
    return true;
    } else {
    return false;
  }
}
```

**私有成员函数NewNode：**

```c++
//创建新的结点
//节点空间包括Node的空间，还有用于保存其它level下一个结点的指针的空间，因为Node里有一个next，所以是height - 1
//使用placement new构造对象
Node* NewNode(const Key& key, int height)
{
    char* const node_memory = arena_->AllocateAligned(
        sizeof(Node) + sizeof(std::atomic<Node*>) * (height - 1));
    return new (node_memory) Node(key);
}
```

**私有成员函数RandomHeight：**

```c++
//1/4的概率豪赌增长
//height起始为1，OneIn是生成一个随机数与kBranching（4）求余，等于0才++，概率为1/4
//这个函数作用就是在插入新的结点时，调用该函数，判断level是否增长，为1就不长
int RandomHeight()
{
    static const unsigned int kBranching = 4;
      int height = 1;
  	while (height < kMaxHeight && rnd_.OneIn(kBranching)) {
    	height++;
    }
  	assert(height > 0);
  	assert(height <= kMaxHeight);
  	return height;
}
```

**私有成员函数Equal：**

```c++
//判断两个key是否相等
bool Equal(const Key& a, const Key& b) const { return (compare_(a, b) == 0); }
```

**私有成员函数KeyIsAfterNode：**

```c++
//判断key是否应该在结点n之后
bool KeyIsAfterNode(const Key& key, Node* n) const
{
    return (n != nullptr) && (compare_(n->key, key) < 0);
}
```

**私有成员函数FindGreaterOrEqual：**

```c++
//找到所有level的key的前一个结点，存储在prev中
//返回的结点要么大于key，要么等于key，要么为nullptr
Node* FindGreaterOrEqual(const Key& key, Node** prev) const
{
    Node* x = head_;
    int level = GetMaxHeight() - 1;
    while (true) {
    Node* next = x->Next(level);
    if (KeyIsAfterNode(key, next)) {
        x = next;
    } else {
        if (prev != nullptr) prev[level] = x;
        if (level == 0) {
            return next;
        } else {
            level--;
        }
    }
  }
}
```

**私有成员函数FindLessThan：**

```c++
//找到小于key的最大的结点，也就是前驱（第0层的前驱）
Node* FindLessThan(const Key& key) const
{
    Node* x = head_;
    int level = GetMaxHeight() - 1;
    while (true) {
        assert(x == head_ || compare_(x->key, key) < 0);
        Node* next = x->Next(level);
        if (next == nullptr || compare_(next->key, key) >= 0) {
            if (level == 0) {
                return x;
            } else {
                level--;
            }
        } else {
            x = next;
        }
    }
}
```

**私有成员函数FindLast：**

```c++
//找到跳表中的最大的结点（第0层的最后一个结点）
Node* FindLast() const
{
    Node* x = head_;
    int level = GetMaxHeight() - 1;
    while (true) {
        Node* next = x->Next(level);
        if (next == nullptr) {
            if (level == 0) {
                return x;
            } else {
                level--;
            }
        } else {
            x = next;
        }
    }
}
```

