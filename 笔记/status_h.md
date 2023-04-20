## Status类

**作用：**维护一个统一的状态码，便于后期维护。

**成员变量：**

```c++
//state_为空表示OK状态
//否则state_是一个new[]，state_[0...3] == length of message，state_[4]    == code，state_[5...]  == message
const char* state_;
```

**构造函数及运算符重载：**

```c++
//默认构造
Status() noexcept : state_(nullptr) {}

//析构函数
~Status() { delete[] state_; }

//拷贝构造
//深拷贝
//state_为空直接赋值为空，否则动态分配新空间，将rhs中的字符串拷贝到分配的新空间
inline Status::Status(const Status& rhs) {
    state_ = (rhs.state_ == nullptr) ? nullptr : CopyState(rhs.state_);
}

//拷贝赋值
inline Status& Status::operator=(const Status& rhs) {
    //避免自我赋值
    if (state_ != rhs.state_) {
    delete[] state_; //释放堆区空间
    state_ = (rhs.state_ == nullptr) ? nullptr : CopyState(rhs.state_); //非空就开辟新的空间拷贝
    }
    return *this;
}

//移动构造
Status(Status&& rhs) noexcept : state_(rhs.state_) { rhs.state_ = nullptr; }

//移动赋值
inline Status& Status::operator=(Status&& rhs) noexcept {
    std::swap(state_, rhs.state_);
    return *this;
}

//私有的有参构造
//根据实参构造状态对象
Status(Code code, const Slice& msg, const Slice& msg2)
{
    assert(code != kOk);
    const uint32_t len1 = static_cast<uint32_t>(msg.size()); //第一条消息的长度
  	const uint32_t len2 = static_cast<uint32_t>(msg2.size()); //第二条消息的长度
  	const uint32_t size = len1 + (len2 ? (2 + len2) : 0); // 第二条消息长度如果不为0则多分配两个字节用于存放':'和' '
  	char* result = new char[size + 5]; //size是消息长度，5是消息长度4字节和状态码1字节
  	std::memcpy(result, &size, sizeof(size)); //将消息长度拷贝到result
  	result[4] = static_cast<char>(code); //将状态码拷贝到result
  	std::memcpy(result + 5, msg.data(), len1); //将第一条消息拷贝到result
  	if (len2) {
        result[5 + len1] = ':'; //用于分隔两条消息的':'和' '
    	result[6 + len1] = ' ';
    	std::memcpy(result + 7 + len1, msg2.data(), len2); //将第二条消息拷贝过来
    }
    state_ = result;
}
```

**成员函数OK：**

```c++
//返回成功状态对象
static Status OK() { return Status(); }
```

**成员函数NotFound：**

```c++
//返回kNotFound状态对象
static Status NotFound(const Slice& msg, const Slice& msg2 = Slice()) {
    return Status(kNotFound, msg, msg2);
}
```

**成员函数Corruption：**

```c++
//返回kCorruption状态对象
static Status Corruption(const Slice& msg, const Slice& msg2 = Slice()) {
    return Status(kCorruption, msg, msg2);
}
```

**成员函数NotSupported：**

```c++
//返回kNotSupported状态对象
static Status NotSupported(const Slice& msg, const Slice& msg2 = Slice()) {
    return Status(kNotSupported, msg, msg2);
}
```

**成员函数InvalidArgument：**

```c++
//返回kInvalidArgument状态对象
static Status InvalidArgument(const Slice& msg, const Slice& msg2 = Slice()) {
    return Status(kInvalidArgument, msg, msg2);
}
```

**成员函数IOError：**

```c++
//返回kIOError状态对象
static Status IOError(const Slice& msg, const Slice& msg2 = Slice()) {
    return Status(kIOError, msg, msg2);
}
```

**成员函数ok：**

```c++
//判断是否是kOk状态
bool ok() const { return (state_ == nullptr); }
```

**成员函数IsNotFound：**

```c++
//判断是否是kNotFound状态
bool IsNotFound() const { return code() == kNotFound; }
```

**成员函数IsCorruption：**

```c++
//判断是否是kCorruption状态
bool IsCorruption() const { return code() == kCorruption; }
```

**成员函数IsIOError：**

```c++
//判断是否是kIOError状态
bool IsIOError() const { return code() == kIOError; }
```

**成员函数IsNotSupportedError：**

```c++
//判断是否是kNotSupported状态
bool IsNotSupportedError() const { return code() == kNotSupported; }
```

**成员函数IsInvalidArgument：**

```c++
//判断是否是kInvalidArgument状态
bool IsInvalidArgument() const { return code() == kInvalidArgument; }
```

**成员函数ToString：**

```c++
//将状态信息转换成字符串用于打印
//返回提示信息+": " + 信息的格式
std::string ToString() const
{
    if (state_ == nullptr) {
        return "OK";
    } else {
        char tmp[30];
        const char* type;
        switch (code()) {
            case kOk:
                type = "OK";
                break;
            case kNotFound:
                type = "NotFound: ";
                break;
            case kCorruption:
                type = "Corruption: ";
                break;
            case kNotSupported:
                type = "Not implemented: ";
                break;
            case kInvalidArgument:
                type = "Invalid argument: ";
                break;
            case kIOError:
                type = "IO error: ";
                break;
            default:
                std::snprintf(tmp, sizeof(tmp),
                              "Unknown code(%d): ", static_cast<int>(code()));
                type = tmp;
                break;
        }
        std::string result(type);
        uint32_t length;
        std::memcpy(&length, state_, sizeof(length));
        result.append(state_ + 5, length);
        return result;
    }
}
```

**私有的枚举类型：**

```c++
//类似于宏定义，但是不会污染宏定义，并且是编译期替换
enum Code {
    kOk = 0,
    kNotFound = 1,
    kCorruption = 2,
    kNotSupported = 3,
    kInvalidArgument = 4,
    kIOError = 5
};
```

**私有的成员函数code：**

```c++
Code code() const {
    //state_为null返回kOk，否则直接从state_的下标4获取状态码
    return (state_ == nullptr) ? kOk : static_cast<Code>(state_[4]);
}
```

**私有的成员函数CopyState：**

```c++
//用于state中的内容拷贝到动态创建的堆区内存
//用于深拷贝
const char* Status::CopyState(const char* state) {
  uint32_t size; //4字节，uint32_t是个typedef，为了方便跨平台和类型一致
  std::memcpy(&size, state, sizeof(size)); //将字符串state中的前4个字节拷贝到size，state的前4个字节表示消息长度
  char* result = new char[size + 5]; //分配9字节，消息长度4字节，消息5字节
  std::memcpy(result, state, size + 5); //将state拷贝到result
  return result;
}
```

