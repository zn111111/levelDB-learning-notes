## Random类

**作用：**生成随机数。

**成员函数Next：**

```c++
//不太懂，好像就是用来随机生成一个数
uint32_t Next() {
    static const uint32_t M = 2147483647L;
    static const uint64_t A = 16807;
    
    uint64_t product = seed_ * A;

    seed_ = static_cast<uint32_t>((product >> 31) + (product & M));

    if (seed_ > M) {
      seed_ -= M;
    }
    return seed_;
  }
```

**成员函数OneIn：**

```c++
bool OneIn(int n) { return (Next() % n) == 0; }
```

