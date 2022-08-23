---
title: leveldb源码学习-2
tags:
  - 数据库
  - leveldb
date: 2022-06-10 01:42:23
categories: leveldb源码学习
---




# Arena内存管理

`char* Allocate(size_t bytes);`分配新的bytes个bytes

`char* AllocateAligned(size_t bytes);`按照malloc 提供的对齐标准分配内存。

Arena类中的私有成员`char* alloc_ptr_;`和`size_t alloc_bytes_remaining_;`标记分配内存后的状态。

`std::vector<char*> blocks_;`保存每个用new[]创建后的内存指针。
<!--more-->
从Allocate看起

```C++
inline char* Arena::Allocate(size_t bytes) {
  // The semantics of what to return are a bit messy if we allow
  // 0-byte allocations, so we disallow them here (we don't need
  // them for our internal use).
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

参数bytes如果是0将会很难处理返回值，由于内部不需要用到0的分配，所以直接assert掉非法值。

bytes如果小于`alloc_bytes_remaining_`则表示再当前的块中就有空间分配，保存指针给返回值，`alloc_ptr_`向后移动bytes，空余值减去bytes。如果当前`alloc_bytes_remaining_`小于bytes，进而调用AllocateFallback

```c++
char* Arena::AllocateFallback(size_t bytes) {
  if (bytes > kBlockSize / 4) {
    // Object is more than a quarter of our block size.  Allocate it separately
    // to avoid wasting too much space in leftover bytes.
    char* result = AllocateNewBlock(bytes);
    return result;
  }

  // We waste the remaining space in the current block.
  alloc_ptr_ = AllocateNewBlock(kBlockSize);
  alloc_bytes_remaining_ = kBlockSize;

  char* result = alloc_ptr_;
  alloc_ptr_ += bytes;
  alloc_bytes_remaining_ -= bytes;
  return result;
}
```

kBlockSize默认指定的是4096，即4KB。如果bytes大于1KB，就单独分配一个大小为bytes的块，这样`alloc_ptr_`后的空间未被浪费，调用AllocateNewBlock。

如果小于1KB，浪费掉当前的`alloc_ptr_`所指的`alloc_bytes_remaining_`的内存（可以看出也没多少剩余了），直接分配新的大小为4KB的Block，将bytes分配到新创建的Block头。

过程中调用的AllocateNewBlock代码

```c++
char* Arena::AllocateNewBlock(size_t block_bytes) {
  char* result = new char[block_bytes];
  blocks_.push_back(result);
  memory_usage_.fetch_add(block_bytes + sizeof(char*),
                          std::memory_order_relaxed);
  return result;
}
```

用new char[block_bytes]分配内存，将得到的块指针push_back进`blocks_`（每次分配的Block都用`blocks_`统一管理）。

![img](https://raw.githubusercontent.com/bevancheng/imgrepo/main/202206092334410.jpeg)



# 编码方式

整数的存储都是采用小端序（Little Endian）

## 定长整数

有32位整数和64位整数，对于32位的低8位在第一个字节，低9-16位在第二字节，即是小端序。

## 变长整数

对于小的整数频繁地使用，一直用32位整数会有些浪费空间。LevelDB有变长整数的编码方式。32位整数使用1-5字节编码，64位整数用1-10字节编码。

1个byte中只用低7位存储数据，高1位做标识，高位1的时候标识需要继续读取下个byte，高位0表示当前byte已是最后一个字节。

![img](https://raw.githubusercontent.com/bevancheng/imgrepo/main/202206092343295.jpeg)

```c++
char* EncodeVarint32(char* dst, uint32_t v) {
  // Operate on characters as unsigneds将char作为uint8_t整数处理
  uint8_t* ptr = reinterpret_cast<uint8_t*>(dst);
  static const int B = 128;//高位标记位1000 0000
  if (v < (1 << 7)) {
    *(ptr++) = v;
  } else if (v < (1 << 14)) {
    *(ptr++) = v | B;
    *(ptr++) = v >> 7;
  } else if (v < (1 << 21)) {
    *(ptr++) = v | B;
    *(ptr++) = (v >> 7) | B;
    *(ptr++) = v >> 14;
  } else if (v < (1 << 28)) {
    *(ptr++) = v | B;//从低位开始处理，小端序
    *(ptr++) = (v >> 7) | B;
    *(ptr++) = (v >> 14) | B;
    *(ptr++) = v >> 21;
  } else {
    *(ptr++) = v | B;
    *(ptr++) = (v >> 7) | B;
    *(ptr++) = (v >> 14) | B;
    *(ptr++) = (v >> 21) | B;
    *(ptr++) = v >> 28;
  }
  return reinterpret_cast<char*>(ptr);
}
```



## 字符串

用前面的EncodeVarint32编码字符串长度，长度后接字符串实际值。用长度前缀编码不需要结尾的'\0'。

```c++
void PutLengthPrefixedSlice(std::string* dst, const Slice& value) {
  PutVarint32(dst, value.size());
  dst->append(value.data(), value.size());
}
```



```c++
void PutVarint32(std::string* dst, uint32_t v) {
  char buf[5];
  char* ptr = EncodeVarint32(buf, v);
  dst->append(buf, ptr - buf);
}
```



## Slice

Slice是Leveldb自定义的存储类型，只存储了数据的指针和长度

```c++
  const char* data_;
  size_t size_;
```

可以很方便的和char *与std::string之间转换

```c++
  // Create a slice that refers to d[0,n-1].
  Slice(const char* d, size_t n) : data_(d), size_(n) {}

  // Create a slice that refers to the contents of "s"
  Slice(const std::string& s) : data_(s.data()), size_(s.size()) {}

  // Create a slice that refers to s[0,strlen(s)-1]
  Slice(const char* s) : data_(s), size_(strlen(s)) {}

  // Return a string that contains the copy of the referenced data.
  std::string ToString() const { return std::string(data_, size_); }

  // Return a pointer to the beginning of the referenced data
  const char* data() const { return data_; }

```



## Comparator

SortedMap的键是有序的，因此需要比较的规则。LevelDB定义了一个Comparator接口。

一个Comparator的实现必须是线程安全的，因为leveldb可能会多线程并发的使用。

```c++
class LEVELDB_EXPORT Comparator {
 public:
  virtual ~Comparator();

  // Three-way comparison.  Returns value:
  //   < 0 iff "a" < "b",
  //   == 0 iff "a" == "b",
  //   > 0 iff "a" > "b"
  virtual int Compare(const Slice& a, const Slice& b) const = 0;

  //
  // The name of the comparator.  Used to check for comparator
  // mismatches (i.e., a DB created with one comparator is
  // accessed using a different comparator.
  //
  // The client of this package should switch to a new name whenever
  // the comparator implementation changes in a way that will cause
  // the relative ordering of any two keys to change.
  //
  // Names starting with "leveldb." are reserved and should not be used
  // by any clients of this package.
  virtual const char* Name() const = 0;

  // Advanced functions: these are used to reduce the space requirements
  // for internal data structures like index blocks.

  // If *start < limit, changes *start to a short string in [start,limit).
  // Simple comparator implementations may return with *start unchanged,
  // i.e., an implementation of this method that does nothing is correct.
  virtual void FindShortestSeparator(std::string* start,
                                     const Slice& limit) const = 0;

  // Changes *key to a short string >= *key.
  // Simple comparator implementations may return with *key unchanged,
  // i.e., an implementation of this method that does nothing is correct.
  virtual void FindShortSuccessor(std::string* key) const = 0;
};

// Return a builtin comparator that uses lexicographic byte-wise
// ordering.  The result remains the property of this module and
// must not be deleted.
LEVELDB_EXPORT const Comparator* BytewiseComparator();

}  // namespace leveldb
```

FindShortestSeparator可以将start更改为位于[start,limit)的最短的字符串，用来优化SSTable里的Index Block里的索引项的长度，使得索引项更短。

FindShortSuccessor将key改为大于等于key的最短值，也是为了减少索引项长度。



## Status

Status的状态由私有成员`char* state_`定义

状态ok，`state_`为null，否则由code和message组成，`state_[0...3]`为msg长度，`state_[4]`为code，`state_[5...]`为msg

# Reference

[[LevelDB\] 基础1：中庸之道 —— arena内存管理 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/210100808)
