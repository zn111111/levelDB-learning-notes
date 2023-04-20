**作用：**根据不同的平台导入不同的头文件。

```c++
//#if defined和#ifdef的用法一致
#if defined(LEVELDB_PLATFORM_POSIX) || defined(LEVELDB_PLATFORM_WINDOWS)
#include "port/port_stdcxx.h"
#elif defined(LEVELDB_PLATFORM_CHROMIUM)
#include "port/port_chromium.h"
#endif
```

