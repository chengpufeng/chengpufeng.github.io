---
title: leveldb源码学习-3
tags:
---

# SSTable

![img](https://raw.githubusercontent.com/bevancheng/imgrepo/main/202206102119285.jpeg)

文件头到尾：Data Block、Filter Block、Meta Index Block、Index Block、Footer

Meta Index Block存储了指向Filter Block的指针，键是Meta Block的名称，值是BlockHandle，一个文件指针

Index Block存储指向每个Data Block的指针的数组，键大于等于对应的Data Block的最后一个键，并小于下一个Data Block的第一个键，通过Index Block可以进行二分查找，快速定位键属于哪个Data Block。

Footer大小固定，保存两个BlockHandle，分别指向Meta Index Block和Index Block。

## BlockHandle

部分代码：

```c++
// BlockHandle is a pointer to the extent of a file that stores a data
// block or a meta block.
class BlockHandle {
 public:
  // Maximum encoding length of a BlockHandle
  enum { kMaxEncodedLength = 10 + 10 };

  void EncodeTo(std::string* dst) const;
  Status DecodeFrom(Slice* input);

 private:
  uint64_t offset_;
  uint64_t size_;
};

void BlockHandle::EncodeTo(std::string* dst) const {
  // Sanity check that all fields have been set
  assert(offset_ != ~static_cast<uint64_t>(0));
  assert(size_ != ~static_cast<uint64_t>(0));
  PutVarint64(dst, offset_);
  PutVarint64(dst, size_);
}
```

位于table/format.h的BlockHandle类，私有成员`uint64_t offset_;`和`uint64_t size_;`

kMaxEncodedLength是两个uint64_t编码后varint64最大的长度20Bytes。`offset_`指向了文件偏移量，`size_`表明了指向的Block大小。存储数据时会用EncodeTo将offset\_和size\_编码到dst。

## Footer

部分代码：

```c++
// Footer encapsulates the fixed information stored at the tail
// end of every table file.
class Footer {
 public:
  // Encoded length of a Footer.  Note that the serialization of a
  // Footer will always occupy exactly this many bytes.  It consists
  // of two block handles and a magic number.
  enum { kEncodedLength = 2 * BlockHandle::kMaxEncodedLength + 8 };
  void EncodeTo(std::string* dst) const;
 private:
  BlockHandle metaindex_handle_;
  BlockHandle index_handle_;
};

void Footer::EncodeTo(std::string* dst) const {
  const size_t original_size = dst->size();
  metaindex_handle_.EncodeTo(dst);
  index_handle_.EncodeTo(dst);
  dst->resize(2 * BlockHandle::kMaxEncodedLength);  // Padding
  PutFixed32(dst, static_cast<uint32_t>(kTableMagicNumber & 0xffffffffu));
  PutFixed32(dst, static_cast<uint32_t>(kTableMagicNumber >> 32));
  assert(dst->size() == original_size + kEncodedLength);
  (void)original_size;  // Disable unused variable warning.
}
```

一个Footer固定为48字节，EncodeTo成员函数将`metaindex_handle_`和`index_handle_`进行编码，然后用`dst->resize`将空间补齐到40字节，再将kTableMagicNumber加入dst，即使得dst固定为48字节。magic number提供了校验功能。拿到一个SSTable后，读取最后48字节，就得到Footer，根据信息可以定位到MetaIndexBlock和IndexBlock。

## BlockContents

除了Footer，其他4部分都用一种Block，用BlockHandle可以读出BlockContents

```c++
// 1-byte type + 32-bit crc
static const size_t kBlockTrailerSize = 5;

struct BlockContents {
  Slice data;           // Actual contents of data
  bool cachable;        // True iff data can be cached
  bool heap_allocated;  // True iff caller should delete[] data.data()
};

// Read the block identified by "handle" from "file".  On failure
// return non-OK.  On success fill *result and return OK.
Status ReadBlock(RandomAccessFile* file, const ReadOptions& options,
                 const BlockHandle& handle, BlockContents* result);

// Implementation details follow.  Clients should ignore,

```

调用ReadBlock可以根据handle找到file的具体位置，然后将内容读到result。每个Block后边会跟5个字节，一个字节的类型信息，4个字节的crc用于校验。

在ReadBlock中，会在读取时，进行一些处理，如果options.verify_checksums被设定，就会用crc检验；如果设定了压缩，会进行SnappyCompression。在内部的file->Read后，有可能会判断出data已经被缓存（创建的缓存和读到的data指针不同），这时候cachable和heap_allocated就会被置false。

## Table

Table::Open

```c++
  static Status Open(const Options& options, RandomAccessFile* file,
                     uint64_t file_size, Table** table);
```

首先判断文件大小，不可小于48Bytes，即一个Footer大小。

```c++
  if (size < Footer::kEncodedLength) {
    return Status::Corruption("file is too short to be an sstable");
  }
```

读取Footer，`file->Read`直接从文件尾读取48Bytes到`Slice footer_input`，然后DecodeFrom到`Footer footer`。

```c++
  char footer_space[Footer::kEncodedLength];
  Slice footer_input;
  Status s = file->Read(size - Footer::kEncodedLength, Footer::kEncodedLength,
                        &footer_input, footer_space);
  if (!s.ok()) return s;

  Footer footer;
  s = footer.DecodeFrom(&footer_input);
  if (!s.ok()) return s;
```

然后读取Index Block，用`footer.index_handle()`作为指针读取到`index_block_contents`，读取成功后，用它创建Block实例`index_block`，将元数据装载到rep，如file、options、index_block指针、metaindex_handle，在`Table **table`用rep初始化创建Table实例，调用`ReadMeta`

```c++
  // Read the index block
  BlockContents index_block_contents;
  ReadOptions opt;
  if (options.paranoid_checks) {
    opt.verify_checksums = true;
  }
  s = ReadBlock(file, opt, footer.index_handle(), &index_block_contents);
  if (s.ok()) {
    // We've successfully read the footer and the index block: we're
    // ready to serve requests.
    Block* index_block = new Block(index_block_contents);
    Rep* rep = new Table::Rep;
    rep->options = options;
    rep->file = file;
    rep->metaindex_handle = footer.metaindex_handle();
    rep->index_block = index_block;
    rep->cache_id = (options.block_cache ? options.block_cache->NewId() : 0);
    rep->filter_data = nullptr;
    rep->filter = nullptr;
    *table = new Table(rep);
    (*table)->ReadMeta(footer);
  }
```

函数原型`void Table::ReadMeta(const Footer& footer)`

因为MetaIndexBlock指向的是Filter，如果options未设置filter_policy，就直接返回，代表MetaIndexBlock实际上是空的。如果设置了Filter，读取块，创建`Block* meta`来存储读到的信息

```c++
  ReadOptions opt;
  if (rep_->options.paranoid_checks) {
    opt.verify_checksums = true;
  }
  BlockContents contents;
  if (!ReadBlock(rep_->file, opt, footer.metaindex_handle(), &contents).ok()) {
    // Do not propagate errors since meta info is not needed for operation
    return;
  }
  Block* meta = new Block(contents);
```

用meta创建一个新的迭代器`Iterator* iter`，key值设`"filter.***"`，用迭代器寻找key，找到后调用`ReadFilter(iter->value())`

```c++
  Iterator* iter = meta->NewIterator(BytewiseComparator());
  std::string key = "filter.";
  key.append(rep_->options.filter_policy->Name());
  iter->Seek(key);
  if (iter->Valid() && iter->key() == Slice(key)) {
    ReadFilter(iter->value());
  }
  delete iter;
  delete meta;
```

函数原型`void Table::ReadFilter(const Slice& filter_handle_value) `

将filtervalue解码至`BlockHandle filter_handle`，用`filter_handle`调用ReadBlock找到block，即Filter Block，将filter_data装载到rep，在`rep_->filter`上创建实例`FilterBlockReader`

至此一个Table就Open成功了，实际上只加载了Index Block和Meta Index Block，接下来使用其中的index Block查找具体数值的过程，首先看InternalGet

函数原型

```c++
Status Table::InternalGet(const ReadOptions& options, const Slice& k, void* arg,
                          void (*handle_result)(void*, const Slice&,
                                                const Slice&))
```

创建一个内部的迭代器iiter，然后调用`iiter->Seek(k)`。Seek会在restart point数组上进行二分查找来确定dataBlock

```c++
  Iterator* iiter = rep_->index_block->NewIterator(rep_->options.comparator);
  iiter->Seek(k);
```

if判断条件，当且仅当用到了filter、handler_value成功解码，且filter中KeyMayMatch不命中，表明绝对不存在这个key，否则就要继续查找。用BlockReader，传入iiter的value，创建一个block_iter，调用Seek，如果找到了即可用函数指针handle_result处理读到的kv。

```c++
    Slice handle_value = iiter->value();
    FilterBlockReader* filter = rep_->filter;
    BlockHandle handle;
    if (filter != nullptr && handle.DecodeFrom(&handle_value).ok() &&
        !filter->KeyMayMatch(handle.offset(), k)) {
      // Not found
    } else {
      Iterator* block_iter = BlockReader(this, options, iiter->value());
      block_iter->Seek(k);
      if (block_iter->Valid()) {
        (*handle_result)(arg, block_iter->key(), block_iter->value());
      }
      s = block_iter->status();
      delete block_iter;
    }
```















# Reference

[这可能是全网最详细的LevelDB SSTable结构解析(配合读取过程源代码讲解) - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/148326950)

[LevelDB源码剖析 - 知乎 (zhihu.com)](https://www.zhihu.com/column/c_1282795241104465920)
