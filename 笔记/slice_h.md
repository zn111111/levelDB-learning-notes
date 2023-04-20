## slice.h头文件

**class LEVELDB_EXPORT Slice {}：**这种类声明方法是方便使用动态链接库里的类，宏定义LEVELDB_EXPORT还配套有一系列的宏定义，在不同人以不同方法使用动态链接库的类时，LEVELDB_EXPORT会被替换为不同的东西，方便使用。详情见[(27条消息) Class和类名称之间的宏定义作用_class+宏+类名_u012903992的博客-CSDN博客](https://tianyalu.blog.csdn.net/article/details/110121769?spm=1001.2101.3001.6650.2&utm_medium=distribute.pc_relevant.none-task-blog-2~default~CTRLIST~Rate-2-110121769-blog-81981031.235^v27^pc_relevant_multi_platform_whitelistv3&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2~default~CTRLIST~Rate-2-110121769-blog-81981031.235^v27^pc_relevant_multi_platform_whitelistv3&utm_relevant_index=5)

## Slice类

**作用：**操纵字符串的类，类似于string。但是C++中的string接口太少，不好用，所以这里重写了一个slice。

**成员变量：**

```c++
private:
	const char* data_; //字符串
	size_t size_; //字符串长度
```

**构造函数以及运算符重载：**

```c++
//默认构造
Slice() : data_(""), size_(0) {}

//有参构造
Slice(const char* d, size_t n) : data_(d), size_(n) {}

//拷贝构造
Slice(const std::string& s) : data_(s.data()), size_(s.size()) {}

//一个参数的有参构造
Slice(const char* s) : data_(s), size_(strlen(s)) {}

//拷贝构造
Slice(const Slice&) = default;

//拷贝赋值
Slice& operator=(const Slice&) = default;

//下表运算符重载
char operator[](size_t n) const {
    assert(n < size());
    return data_[n];
}

//==运算符重载
inline bool operator==(const Slice& x, const Slice& y) {
  return ((x.size() == y.size()) &&
          (memcmp(x.data(), y.data(), x.size()) == 0));
}

//!=运算符重载
inline bool operator!=(const Slice& x, const Slice& y) { return !(x == y); }
```

**成员函数data：**

```c++
//获取slice对象中存储的字符串
const char* data() const { return data_; }
```

**成员函数size：**

```c++
//获取字符串长度
size_t size() const { return size_; }
```

**成员函数empty：**

```c++
//判断字符串是否为空
bool empty() const { return size_ == 0; }
```

**成员函数clear：**

```c++
//将字符串置空
void clear() {
    data_ = "";
    size_ = 0;
}
```

**成员函数remove_prefix：**

```c++
//删除字符串的前n个字符
void remove_prefix(size_t n) {
    assert(n <= size());
    data_ += n;
    size_ -= n;
}
```

**成员函数ToString：**

```c++
//将const char*转成string
std::string ToString() const { return std::string(data_, size_); }
```

**成员函数compare：**

```c++
//将两个字符串按字典序进行比较
//memcmp是按字典序比较两个内存，< 0表示第一个小于第二个，= 0表示二者相等，> 0表示第一个大于第二个
int compare(const Slice& b) const
{
    const size_t min_len = (size_ < b.size_) ? size_ : b.size_;
  	int r = memcmp(data_, b.data_, min_len);
  	if (r == 0) {
    if (size_ < b.size_)
        r = -1;
    else if (size_ > b.size_)
        r = +1;
    }
    return r;
}
```

**成员函数starts_with：**

```c++
//判断x是否是当前对象的前缀
bool starts_with(const Slice& x) const {
    return ((size_ >= x.size_) && (memcmp(data_, x.data_, x.size_) == 0));
}
```

