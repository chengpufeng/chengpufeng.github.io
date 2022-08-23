---
title: leveldb源码学习-1
tags:
  - 数据库
  - leveldb
date: 2022-06-09 22:04:36
categories: leveldb源码学习
---


# LevelDB源码学习(1)



<!--more-->

## 安装

```bash
git clone --recurse-submodules https://github.com/google/leveldb.git
mkdir -p build && cd build
cmake -DCMAKE_BUILD_TYPE=Release .. && cmake --build .
```



```C++
// hello.cc

#include <assert.h>
#include <string.h>
#include <leveldb/db.h>
#include <iostream>
using namespace leveldb;

int main(){
    leveldb::DB* db;
    leveldb::Options options;
    options.create_if_missing = true;
    // 打开一个数据库，不存在就创建
    leveldb::Status status = leveldb::DB::Open(options, "/tmp/testdb", &db);
    assert(status.ok());

   // 插入一个键值对
    status = db->Put(leveldb::WriteOptions(), "hello", "LevelDB");
    assert(status.ok());

   // 读取键值对
    std::string value;
    status = db->Get(leveldb::ReadOptions(), "hello", &value);

    assert(status.ok());
    std::cout << value << std::endl;

    delete db;
    return 0;
}
```

编译执行

```shell
g++ hello.cc -o hello -L. -I./include -pthread -lleveldb
./hello
```



### 目录结构

`leveldb/include`函数的头文件

`leveldb/port`可移植性相关的功能

`leveldb/util`项目用到的一些功能函数

`leveldb/table`SSTable的实现

`leveldb/db`数据库实现，版本管理，Compaction，WAL和MemTable实现

LevelDB作为函数库，对外提供的接口文件及功能：

```
cache.h: 缓存接口，提供默认LRU缓存，也可自己实现
comparator.h: 定以数据库比较器的接口，用来比较键，可以使用默认的基于字节的比较，可以定义自己的比较器
dumpfile.h: 以可读文本形式导出一个文件，调试文件
export.h: 可移植性相关
iterator.h: 迭代器接口
slice.h: 实现一个字符串，存储指针和长度，指向字符串
table_builder.h: 构造一个SSTable
write_batch.h: 实现批量写入接口
c.h: 实现c语言相关的接口
db.h: 操作数据库的主要接口
env.h: 操作系统相关功能，如读写文件
filter_policy.h: 布隆过滤器接口
options.h: 配置选项
status.h: 定义数据库操作的返回状态
table.h: SSTable相关接口
```



## 基本用法

并发：当一个进程打开LevelDB时，会获取这个数据库的一个文件锁，其他进程没法获取这个文件锁。所以leveldb只能一个进程访问，但可多线程。level::DB很多方法都是线程安全的，有加锁步骤。但对于一些对象，如WriteBatch，如果多线程并发，需要同步。

### 迭代器：

```c++
  leveldb::Iterator* it = db->NewIterator(leveldb::ReadOptions());
  for (it->SeekToFirst(); it->Valid(); it->Next()) {
    std::cout << it->key().ToString() << ": " << it->value().ToString()
              << std::endl;
  }

```

### 快照：

```c++
leveldb::ReadOptions options;
options.snapshot = db->GetSnapshot();
//do modifying
db->ReleaseSnapshot(options.snapshot);
```

### 比较器：

继承Comparator，实现虚函数，再将它传入options

```c++
class TwoPartComparator : public leveldb::Comparator {
 public:
  // Three-way comparison function:
  //   if a < b: negative result
  //   if a > b: positive result
  //   else: zero result
  int Compare(const leveldb::Slice& a, const leveldb::Slice& b) const {
    int a1, a2, b1, b2;
    ParseKey(a, &a1, &a2);
    ParseKey(b, &b1, &b2);
    if (a1 < b1) return -1;
    if (a1 > b1) return +1;
    if (a2 < b2) return -1;
    if (a2 > b2) return +1;
    return 0;
  }

  // Ignore the following methods for now:
  const char* Name() const { return "TwoPartComparator"; }
  void FindShortestSeparator(std::string*, const leveldb::Slice&) const {}
  void FindShortSuccessor(std::string*) const {}
};
```

使用

```c++
TwoPartComparator cmp;
leveldb::DB* db;
leveldb::Options options;
options.create_if_missing = true;
options.comparator = &cmp;
leveldb::Status status = leveldb::DB::Open(options, "/tmp/testdb", &db);
```

比较器的Name会在创建数据库时就附加到数据库中，之后每次打开时都会检查，如果名称更改，则调用失败。只有当且仅当有新的键格式、或比较功能与现有数据库不兼容、或可以丢弃所有数据时，才可以更改名称。

比较好的解决方法：可以在所有的键尾加上1byte的版本号，保持比较器Name不变，有了新的键变更版本号，比较函数的实现中根据版本号不同具体分析。

### 数据校验：

打开时校验或读取时校验

> `bool leveldb::Options::paranoid_checks`
>
> If true, the implementation will do aggressive checking of the data it is processing and will stop early if it detects any errors. This may have unforeseen ramifications: for example, a corruption of one DB entry may cause a large number of entries to become unreadable or for the entire DB to become unopenable.
>
> `bool leveldb::ReadOptions::verify_checksums`
>
> If true, all data read from underlying storage will be
> verified against corresponding checksums.

## 性能

基本都会用到`include/options.h`

### 缓存：

提供LRU Cache，设置缓存一定空间大小，缓存最近使用的Data Block。也可以提供自己的缓存策略，只需要实现Cache接口即可。默认8MB LRU Cache。

### 布隆过滤器：

先读取布隆过滤器，如果告诉这个键不存在，就不需要再读取这个Data Block了。

```c++
  options.filter_policy = NewBloomFilterPolicy(10);
```

参数10表示每个键使用10bit空间构造布隆过滤器。10bit指的是，如果布隆过滤器说一个键不存在，那么这个键一定不存在，如果这个键存在，99%概率是存在的，1%概率是不存在的，假阳率1%。10是比较好的数，平衡了布隆过滤器所占的空间和假阳率。

布隆过滤器的数据要写入SSTable，当SSTable打开，布隆过滤器的数据常驻内存，知道SSTable关闭。

需要注意的是，过滤器和比较器需要兼容，如果比较器比较key时忽略了结尾空白符，NewBloomFilterPolicy是不兼容的，需要提供自己的fliter policy，同样忽略掉结尾空白符。

```c++
class CustomFilterPolicy : public leveldb::FilterPolicy {
 private:
  leveldb::FilterPolicy* builtin_policy_;

 public:
  CustomFilterPolicy() : builtin_policy_(leveldb::NewBloomFilterPolicy(10)) {}
  ~CustomFilterPolicy() { delete builtin_policy_; }

  const char* Name() const { return "IgnoreTrailingSpacesFilter"; }

  void CreateFilter(const leveldb::Slice* keys, int n, std::string* dst) const {
    // Use builtin bloom filter code after removing trailing spaces
    std::vector<leveldb::Slice> trimmed(n);
    for (int i = 0; i < n; i++) {
      trimmed[i] = RemoveTrailingSpaces(keys[i]);
    }
    builtin_policy_->CreateFilter(trimmed.data(), n, dst);
  }
};
```



### 压缩：

`options.commpression = leveldb::kNoCompression;`

### 键布局：

磁盘和缓存的单位都是块，相邻键通常放在同一块。

如果存放文件，由metadata和data，如果只想扫描metadata，不想扫描巨大的文件内容，可以在文件名前加1个字符，如“/”，或数字“0”。

## 架构



![img](https://raw.githubusercontent.com/bevancheng/imgrepo/main/202206091801110.jpeg)

按储存来分有三个组件：

- MemTable
- SSTable
- WAL

### MemTable

一个内存数据结构，是个SortedMap：

提供Get接口，也有Put接口；键是有序的，可以按照键的顺序迭代Map，或者范围查找。

内存数据结构不需磁盘IO，读写速度快。

问题：数据库实例崩溃、宕机或停机维护时数据就会消失，因此需要持久化。

### WAL

一种日志技术，当修改数据时，想把对数据的修改写到磁盘上，然后再MemTable里修改。日志记录每个操作，因此只要对日志进行重放就可以恢复MemTable。保证了实例崩溃、宕机后的数据不丢失。

类似的技术有Redis里的Aof，Innodb的Redo Log，称为WAL（Write Ahead Log），在写前先写入日志。

日志写都是append，写磁盘效率很高，因此也可以满足写多的场景。

控制日志写同步的策略在性能和可靠性之间做折中：

- 每次写入都sync，可靠性高，不会丢数据，性能最低
- 每次写入都不sync，数据库崩溃不丢数据，但机器崩溃会丢数据，性能高。
- 每次写入不sync，每1s做一次sync，数据库崩溃不丢数据，机器崩溃丢1s数据。

存在问题：

- 日志量大（通常最大4MB），每次重启数据库恢复的时间长
- 内存优先，MemTable超过容量就无法写入

因此还需要将MemTable的镜像写入磁盘，并清空MemTable。

### SSTable

MemTable镜像写入磁盘的要求：需要快速地从数据库查询一个键。

B+Tree的更新开销太大，不太合适。如果更新MemTable镜像的方式是原地更新已经存到磁盘的MemTable，那么和B+Tree的方式类似，更新开销也很大。所以LevelDB每次都将MemTable写入一个新镜像。

SSTable（Sorted String Table），数据按照键顺序存储到磁盘上。键可以二分查找，每个磁盘块对应一个索引表明这个磁盘块中键的范围，查找时先查索引，在读取对应的磁盘块。

后台线程定时将MemTable的镜像写入磁盘，而正常写入数据还是向MemTable和Log写入。SSTable和Log不在一个磁盘上的话，依旧符合顺序写，效率不会受太大影响。当把镜像写入MemTable后，之前的日志就可以淘汰，因为之前的数据已经持久化到磁盘。解决了日志量随写入量增加而增加导致的数据库恢复时间长的问题，解决了MemTable容量的问题。

新的问题：读变慢。现在读的时候，除了直接读内存中的MemTable还需要读SSTable，而且可能要读取多个SSTable，需要多次随机读，效率低。

### Compaction

多个小的SSTable合并一个大的SSTable，可以解决查找效率问题。磁盘中文件分两部分：一部分MemTable直接写入的镜像SSTable（Level-0)，另一部分是大SSTable（Level-1）。Level-0文件的键范围可能有重叠，Level-1~n不会有重叠。读取时需要读取Level-0和Level-1，限制文件数量的话，磁盘IO次数也是常数范围。Level-1文件会越来越大，这时候就需要合并到Level-2了。

预设的Level-0的数目上限是4，每2MB数据会创建一个Level-1文件。n>=1层的文件同层之间不会有键重叠，当n层的文件总大小超过$10^nMB$​时（10MB-Level-1,100MB-Level-2,...），一个Level-n和所有与其有重叠的Level-n+1文件就会合并成一系列的n+1级文件。对于n\==0，需要把所有key范围有重叠的文件全部找到。因此对于n\==0，n层输入文件可能多个，n+1层也可能多个。

实际上Compaction有两种：

#### minor compaction

将immutable MemTable写入磁盘，就是生成Level-0的过程。

#### major compaction

这种compaction的输入全是磁盘文件，合并level-n和level - n+1的若干文件，新产生文件到level - n+1。

- size compaction：某个level的总文件大小超过设定阈值。
- seek compaction： 某个文件的访问次数超过阈值触发，优先级低于size compaction。
- manual compaction： 手动触发，可以指定某个level的一定key范围的文件和下一层进行compaction，业务中一般不应调用该接口。



### MANIFEST

对于当前数据库有哪些SSTable，每个SSTable属于哪一层、键范围、文件大小等信息，需要持久化到磁盘，每次打开数据库，这些数据需要被从磁盘加载到内存，这些持久化数据存在MANIFEST文件。

Compaction进行后，元数据会改变，因此也要改变MANIFEST。恢复元数据时，用初始MANIFEST和各个操作恢复出最终元数据。改变太多，MANIFEST太大，恢复就会很耗时，这时候直接舍弃旧的，写入到新的MANIFEST。CURRENT文件存储了当前使用的MANIFEST文件。





# Reference

[(14条消息) leveldb源码分析（五）Compaction_kdb_viewer的博客-CSDN博客](https://blog.csdn.net/kdb_viewer/article/details/115866773)

[LevelDB设计与实现 - Compaction - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/51573929)

[Leveldb解析之四：Compaction - 简书 (jianshu.com)](https://www.jianshu.com/p/0f216c6a397a)

[[LevelDB\] 原理：知其然知其所以然——LevelDB基本原理 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/206608102)

[leveldb/impl.md at main · bevancheng/leveldb (github.com)](https://github.com/bevancheng/leveldb/blob/main/doc/impl.md)

[LSM树详解 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/181498475)
