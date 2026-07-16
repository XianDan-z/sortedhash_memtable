# RocksDB HashSpdb Memtable 详细设计文档

## 1. 概述

### 1.1 背景

RocksDB 默认使用 SkipList 作为 memtable 数据结构。SkipList 提供优秀的有序遍历（scan）性能，但点查询（point lookup）需要 O(log N) 遍历跳表节点链。

HashSpdb memtable 旨在通过 **双数据结构** 设计同时优化点查询和有序遍历：
- **Hash Table**：提供 O(1) 平均时间复杂度的点查询
- **InlineSkipList**：提供 O(log N) 的有序遍历和并发安全的 scan 迭代

### 1.2 版本信息

- **RocksDB 版本**：6.26.1
- **基线 commit**：`4d57a393a` (Release 6.26.1)
- **新增 commits**：
  - `0f4926728` — add hashspdb memtable（初始实现，使用 SpdbVectorContainer 排序向量+归并堆）
  - `73c3f519d` — refactor: replace SpdbVectorContainer with InlineSkipList for scan（重构 scan 路径）
  - `96af879de` — fix: store full key+value encoding in skip list for correct flush（修复 flush crash）

### 1.3 变更文件清单

| 文件 | 变更类型 | 说明 |
|------|---------|------|
| `memtable/hash_spdb_rep.cc` | 新增 | HashSpdb memtable 核心实现 |
| `memtable/spdb_sorted_vector.h` | 新增 | 原始版排序向量数据结构（重构后仅保留，不再被 hash_spdb_rep.cc 引用） |
| `include/rocksdb/memtablerep.h` | 修改 | 新增 `IsRefreshIterSupported()`、`PreCreateMemTableRep()`、`PostCreateMemTableRep()` 接口；`GetIterator()` 增加 `part_of_flush` 参数；新增 `IsEmpty()` 虚方法；新增 `NewHashSpdbRepFactory()` 声明 |
| `table/plain/plain_table_factory.cc` | 修改 | 注册 `HashSpdbRepFactory`/`hash_spdb` 到 MemTableRepFactory 工厂注册表 |
| `CMakeLists.txt` | 修改 | 添加 `memtable/hash_spdb_rep.cc` 到 SOURCES |
| `src.mk` | 修改 | 添加 `memtable/hash_spdb_rep.cc` 到 LIB_SOURCES |
| `memtable/stl_wrappers.h` | 修改 | `Compare` 结构体新增 `operator()(const char*, const Slice&)` 重载 |
| `memtable/hash_linklist_rep.cc` | 修改 | `GetIterator()` 签名适配 `part_of_flush` 参数 |
| `memtable/hash_skiplist_rep.cc` | 修改 | 同上 |
| `memtable/skiplistrep.cc` | 修改 | 同上 |
| `memtable/vectorrep.cc` | 修改 | 同上 |
| `db/db_memtable_test.cc` | 修改 | MockMemTableRep 适配新 `GetIterator()` 签名 |
| `test_util/testutil.cc` | 修改 | SpecialMemTableRep 适配新 `GetIterator()` 签名 |
| `java/CMakeLists.txt` | 修改 | 添加 `HashSpdbMemTableConfig.java` 到 Java 编译列表和 JNI header 生成列表 |
| `java/rocksjni/memtablejni.cc` | 修改 | 添加 `HashSpdbMemTableConfig_newMemTableFactoryHandle` JNI 方法 |
| `java/src/main/java/org/rocksdb/HashSpdbMemTableConfig.java` | 新增 | Java 层 HashSpdb memtable 配置类 |

**变更统计**：16 个文件，+968 行 / -10 行

---

## 2. 架构设计

### 2.1 整体架构

```
HashSpdbRep (继承 MemTableRep)
├── SpdbHashTable          ← 点查询路径 (Get / Contains)
│   └── vector<BucketHeader>
│       └── BucketHeader
│           ├── RWMutex         (读写锁，保护并发访问)
│           ├── atomic<SpdbKeyHandle*> items_  (链表头)
│           └── atomic<uint32_t> elements_num_
│
└── InlineSkipList         ← 有序遍历路径 (GetIterator / scan / flush)
    └── CAS-based 无锁并发跳表
        ├── InsertConcurrently()  (写入)
        └── Iterator              (读取/遍历)
```

### 2.2 数据流向

```
写入路径 (InsertKey):
  MemTable::Add()
    → Allocate(len) → 分配 SpdbKeyHandle + key data (memtable arena)
    → 用户填充 key data: [varint key_size][key][varint val_size][value]
    → InsertKey(handle)
        ├── SpdbHashTable::Add(spdb_handle)        ← 插入 hash table (WriteLock)
        │   └── BucketHeader 链表有序插入
        └── skip_list_.AllocateKey(total_len)      ← 分配 skip list key
            memcpy(sl_key, src_key, total_len)       ← 拷贝完整 key+value 编码
            skip_list_.InsertConcurrently(sl_key)   ← CAS 无锁插入

点查询路径 (Get / Contains):
  MemTable::Get()
    → SpdbHashTable::Get()                          ← 只走 hash table
        └── BucketHeader::Get() (ReadLock)
            └── 链表线性扫描 (桶内有序)

有序遍历路径 (GetIterator):
  MemTableIterator 构造
    → GetIterator(arena)
        └── new SpdbSkipListIterator(&skip_list_)   ← 只走 skip list
            └── InlineSkipList::Iterator
                ├── Seek(key)    O(log N)
                ├── Next()       O(1)
                └── Prev()       O(1)
```

### 2.3 关键设计决策

#### 2.3.1 双数据结构 vs 单数据结构

| 方案 | 点查 | Scan | 写入开销 | 内存 |
|------|------|------|---------|------|
| 纯 SkipList（默认） | O(log N) | O(log N) seek + O(1) next | 1 次 CAS 插入 | 1 份 |
| 纯 Hash Table | O(1) | 需全排序 O(N log N) | 1 次链表插入 | 1 份 |
| **HashSpdb（双结构）** | **O(1) hash** | **O(log N) skip list** | **2 次插入** | **2 份** |

选择双数据结构的理由：点查和 scan 各走最优路径，互不干扰。代价是写入时维护两份索引、内存翻倍。

#### 2.3.2 InlineSkipList 替换 SpdbVectorContainer（重构核心）

**原始实现（commit `0f4926728`）**使用 SpdbVectorContainer：
- 数据按 10000 条/批写入排序向量（SpdbVector）
- 迭代器创建时：锁 + 拷贝所有向量引用 + 为每个向量分配 SortHeapItem
- Seek 时：对每个向量排序 + 二分搜索 + 构建二叉堆归并
- **问题**：32 线程并发 scan 时，InitIterator 的锁竞争 + Sort/SeekIter/析构占 54% CPU，性能极差

**重构实现（commit `73c3f519d`）**使用 InlineSkipList：
- 迭代器创建：O(1)，直接构造 InlineSkipList::Iterator
- Seek：O(log N)，标准跳表搜索
- Next/Prev：O(1)，指针跳转
- **优势**：无锁并发读，迭代器创建/销毁零开销

#### 2.3.3 Key 编码格式

memtable 中的 key 格式为：`[varint32 internal_key_size][internal_key][varint32 value_size][value]`

**关键修复（commit `96af879de`）**：skip list 中必须存储**完整的 key+value 编码**，而非仅 key 部分。因为 RocksDB 的 flush 迭代器从同一指针解析 key 和 value。仅存 key 部分会导致 flush 时读取 value 越界，引发 segfault。

解决方案：在 `SpdbKeyHandle` 中新增 `key_len` 字段记录完整编码长度，`InsertKey` 时 `memcpy` 完整数据到 skip list。

---

## 3. 核心数据结构

### 3.1 SpdbKeyHandle

```cpp
struct SpdbKeyHandle {
  SpdbKeyHandle* GetNextBucketItem();      // 原子读取 next 指针
  void SetNextBucketItem(SpdbKeyHandle*);  // 原子写入 next 指针
  std::atomic<SpdbKeyHandle*> next_{nullptr};  // hash table 链表指针
  size_t key_len{0};                            // 完整 key+value 编码长度
  char key_[1];                                 // flexible array, key 数据起始
};
```

- 用于 hash table 的有序链表节点
- `next_` 使用 `std::atomic` 支持并发读取
- `key_len` 记录 `[varint][key][varint][value]` 的完整长度，用于 skip list 拷贝
- `key_` 是 flexible array member，实际数据紧跟其后

### 3.2 BucketHeader

```cpp
struct BucketHeader {
  port::RWMutex rwlock_;                    // 读写锁
  std::atomic<SpdbKeyHandle*> items_{nullptr};  // 链表头
  std::atomic<uint32_t> elements_num_{0};   // 元素计数

  bool Contains(const char* key, const KeyComparator&, bool needs_lock);
  bool Add(SpdbKeyHandle* handle, const KeyComparator&);
  void Get(const LookupKey& k, const KeyComparator&, void*, callback, bool needs_lock);
};
```

- 每个 hash bucket 一个 BucketHeader
- 链表按 key 有序排列，支持提前终止扫描
- `Add` 使用 WriteLock，`Get`/`Contains` 使用 ReadLock
- `elements_num_` 快速判断空桶，避免锁开销

### 3.3 SpdbHashTable

```cpp
struct SpdbHashTable {
  std::vector<BucketHeader> buckets_;

  bool Add(SpdbKeyHandle*, const KeyComparator&);
  bool Contains(const char* key, const KeyComparator&, bool needs_lock) const;
  void Get(const LookupKey&, const KeyComparator&, void*, callback, bool needs_lock) const;

  // hash: MurmurHash(user_key_without_timestamp, 0)
  // bucket index: hash % buckets_.size()
};
```

- 固定大小 bucket 数组，默认 1,000,000 个 bucket
- 使用 MurmurHash 对 user key（不含 timestamp）做 hash
- 每个 bucket 内部是有序链表

### 3.4 SpdbSkipListIterator

```cpp
class SpdbSkipListIterator : public MemTableRep::Iterator {
  SpdbSkipList::Iterator iter_;  // InlineSkipList::Iterator
  std::string tmp_;              // EncodeKey 临时缓冲区

  // 委托给 InlineSkipList::Iterator:
  void Seek(const Slice& internal_key, const char* memtable_key);
  void SeekForPrev(const Slice& internal_key, const char* memtable_key);
  void Next(); / void Prev();
  void SeekToFirst(); / void SeekToLast();
  bool Valid() const; / const char* key() const;
};
```

- 轻量级包装器，将 `MemTableRep::Iterator` 接口委托给 `InlineSkipList::Iterator`
- 创建开销：O(1)（无锁、无内存分配）
- 支持并发读取（InlineSkipList 的 CAS 设计保证）

### 3.5 HashSpdbRep

```cpp
class HashSpdbRep : public MemTableRep {
  SpdbHashTable spdb_hash_table_;     // 点查数据结构
  SpdbSkipList skip_list_;            // 有序遍历数据结构（直接成员）
  const MemTableRep::KeyComparator& cmp_;
  std::atomic<bool> immutable_{false};

  // 点查 → hash table
  void Get(const LookupKey&, void*, callback) override;
  bool Contains(const char* key) const override;

  // 有序遍历 → skip list
  MemTableRep::Iterator* GetIterator(Arena*, bool part_of_flush) override;

  // 写入 → hash table + skip list
  bool InsertKey(KeyHandle handle) override;

  // 并发写入支持
  bool IsInsertConcurrentlySupported() const override { return true; }
  bool CanHandleDuplicatedKey() const override { return false; }
  bool IsRefreshIterSupported() const override { return true; }
};
```

**关键设计点**：
- `skip_list_` 是直接成员（非 `unique_ptr`），确保生命周期与 HashSpdbRep 对象一致，避免 flush 迭代器悬空指针
- `MarkReadOnly()` 设置 `immutable_` 标志，停止 hash table 写入

---

## 4. 接口设计

### 4.1 C++ 接口

#### 工厂创建

```cpp
// include/rocksdb/memtablerep.h
extern MemTableRepFactory* NewHashSpdbRepFactory(
    size_t bucket_count = 1000000,
    bool use_merge = true);
```

#### Options 文件配置

```ini
# INI 格式
memtable_factory=hash_spdb              # 默认 bucket_count=1000000
memtable_factory=hash_spdb:400000        # 指定 bucket_count
memtable_factory=HashSpdbRepFactory      # 使用类名
```

#### db_bench 命令行

```bash
./db_bench --memtablerep=hash_spdb --hash_bucket_count=1000000
```

### 4.2 Java/JNI 接口

```java
// Java 层
HashSpdbMemTableConfig config = new HashSpdbMemTableConfig()
    .setBucketCount(1000000);
Options options = new Options().setMemTableConfig(config);
```

```cpp
// JNI 层 (java/rocksjni/memtablejni.cc)
jlong Java_org_rocksdb_HashSpdbMemTableConfig_newMemTableFactoryHandle(
    JNIEnv* env, jobject, jlong jbucket_count) {
  return reinterpret_cast<jlong>(
      NewHashSpdbRepFactory(static_cast<size_t>(jbucket_count)));
}
```

### 4.3 YCSB 绑定接口

```bash
# YCSB 命令行
./bin/ycsb load rocksdb -p rocksdb.memtable=hash_spdb -p rocksdb.memtable.buckets=400000
./bin/ycsb run rocksdb  -p rocksdb.memtable=hash_spdb
```

```java
// RocksDBClient.java 中的 memtable 选择逻辑
case "hash_spdb":
  return new HashSpdbMemTableConfig().setBucketCount(buckets);
```

### 4.4 工厂注册

```cpp
// table/plain/plain_table_factory.cc
library.Register<MemTableRepFactory>(
    AsRegex("HashSpdbRepFactory", "hash_spdb"),
    [](const std::string& uri, ...) {
      auto colon = uri.find(":");
      if (colon != std::string::npos) {
        guard->reset(NewHashSpdbRepFactory(ParseSizeT(uri.substr(colon + 1))));
      } else {
        guard->reset(NewHashSpdbRepFactory());
      }
      return guard->get();
    });
```

支持通过 `hash_spdb`（昵称）或 `HashSpdbRepFactory`（类名）匹配，可选 `:bucket_count` 后缀。

---

## 5. 并发安全设计

### 5.1 写入并发

| 操作 | 机制 | 说明 |
|------|------|------|
| `SpdbHashTable::Add` | `BucketHeader::rwlock_` WriteLock | 按 bucket 粒度加锁，不同 bucket 无竞争 |
| `skip_list_.InsertConcurrently` | CAS 无锁 | InlineSkipList 的 `Insert<true>` 使用 CAS 原子操作 |
| `IsInsertConcurrentlySupported` | `true` | 声明支持并发写入，RocksDB 启用 `allow_concurrent_memtable_write` |

### 5.2 读取并发

| 操作 | 机制 | 说明 |
|------|------|------|
| `SpdbHashTable::Get/Contains` | `BucketHeader::rwlock_` ReadLock | 多线程可并发读同一 bucket |
| `SpdbSkipListIterator` | 无锁 | InlineSkipList 迭代器天然支持并发读取 |
| flush `GetIterator` | 无锁 | flush 线程创建 skip list 迭代器，不阻塞写入线程 |

### 5.3 读写并发

写入线程执行 `InsertKey`（hash table Add + skip list InsertConcurrently），同时：
- flush 线程通过 `GetIterator` 创建 skip list 迭代器 → 安全（CAS 保证）
- 读线程通过 `Get` 查询 hash table → 安全（RWLock 保证）

### 5.4 MarkReadOnly 机制

```cpp
void MarkReadOnly() override { immutable_.store(true); }
```

- flush 前调用，设置 `immutable_` 标志
- `Contains`/`Get` 始终使用锁（`needs_lock=true`），不依赖 `immutable_` 做无锁快速路径
- 写入操作不显式检查 `immutable_`（由 RocksDB 上层保证 flush 后不再写入）

---

## 6. 内存管理

### 6.1 内存分配

所有内存通过 RocksDB 的 `Allocator`（通常是 Arena）分配：

| 分配点 | Allocator | 大小 | 用途 |
|--------|-----------|------|------|
| `Allocate()` | `allocator_->AllocateAligned` | `sizeof(SpdbKeyHandle) + len` | hash table 节点 + key data |
| `InsertKey() skip_list_.AllocateKey()` | `skip_list_` 内部 allocator（同一个 Arena） | `Node header + total_len` | skip list 节点 + key data 副本 |

### 6.2 内存生命周期

- hash table key 和 skip list key 是**不同的内存**（hash table 存 `SpdbKeyHandle->key_`，skip list 存 memcpy 副本）
- 两者都在 memtable 的 Arena 上分配，随 memtable 释放时统一回收
- `ApproximateMemoryUsage()` 返回 0（内存由 allocator 管理）

### 6.3 内存开销

相比纯 SkipList，HashSpdb 额外内存开销：
- 每个 key 多一份 `SpdbKeyHandle` 头部（`sizeof(SpdbKeyHandle)` = 24 bytes）
- 每个 key 在 skip list 中多一份完整拷贝（`total_len` bytes）
- SpdbHashTable 的 bucket 数组（`bucket_count * sizeof(BucketHeader)`）

---

## 7. 构建集成

### 7.1 CMake 构建

```cmake
# CMakeLists.txt - SOURCES 列表
set(SOURCES
    ...
    memtable/hash_spdb_rep.cc    # 新增
    ...
)
```

### 7.2 Makefile 构建

```makefile
# src.mk - LIB_SOURCES 列表
LIB_SOURCES += \
    memtable/hash_spdb_rep.cc    # 新增
```

### 7.3 JNI 构建

```cmake
# java/CMakeLists.txt
set(JAVA_MAIN_CLASSES
    ...
    src/main/java/org/rocksdb/HashSpdbMemTableConfig.java  # 新增
    ...
)
# JNI header 生成列表也同步新增
```

### 7.4 编译命令

```bash
# C++ db_bench
make -j$(nproc) db_bench DEBUG_LEVEL=0 \
    DISABLE_JEMALLOC=1 ROCKSDB_DISABLE_TCMALLOC=1

# JNI jar
make -j$(nproc) rocksdbjava DEBUG_LEVEL=0 \
    DISABLE_JEMALLOC=1 ROCKSDB_DISABLE_TCMALLOC=1
```

---

## 8. 原始实现与重构对比

### 8.1 原始实现（commit `0f4926728`）

**Scan 数据结构**：SpdbVectorContainer（排序向量 + 归并堆）

```
SpdbVectorContainer
├── list<SpdbVectorPtr> spdb_vectors_     (多个排序向量)
│   └── SpdbVector (vector<const char*>)
│       ├── Add(key) → 原子 fetch_add 写入
│       └── Sort() → std::sort + WriteLock
├── SortThread                           (后台排序线程)
└── InitIterator() → 创建 SortHeapItem per vector
    SeekIter() → Sort each vector + binary search + BinaryHeap merge
```

**问题**：
1. 迭代器创建需锁（`spdb_vectors_mutex_`）+ 为每个 vector 分配 `SortHeapItem`（new）
2. Seek 需对每个 vector 排序 + 二分搜索 + 堆构建
3. 析构需 delete 所有 SortHeapItem
4. 32 线程并发时锁竞争严重

### 8.2 重构实现（commit `73c3f519d` + `96af879de`）

**Scan 数据结构**：InlineSkipList（无锁并发跳表）

```
InlineSkipList
├── InsertConcurrently(key) → CAS 无锁插入
└── Iterator
    ├── Seek(key) → O(log N)
    ├── Next() → O(1)
    └── 创建/销毁 → O(1)，无锁无分配
```

**改进**：
1. 迭代器创建 O(1)，无锁无内存分配
2. Seek O(log N)，标准跳表搜索
3. 析构 O(1)（arena 管理）
4. 无锁并发读，无竞争

### 8.3 重构效果

| 维度 | 原始（SpdbVectorContainer） | 重构（InlineSkipList） |
|------|---------------------------|----------------------|
| 迭代器创建 | O(V) + 锁 + new V 个 SortHeapItem | O(1)，无锁 |
| Seek | O(V log V) 排序 + O(V log N) 二分 + O(V) 堆 | O(log N) |
| Next | O(log V) 堆更新 | O(1) |
| 析构 | O(V) 次 delete | O(1) |
| 并发读 | 锁竞争 | 无锁 |
| Scan 性能（mixgraph） | 2,908 ops/s | 2,576,344 ops/s (**886x**) |

---

## 9. 已知限制

1. **内存翻倍**：每个 key 在 hash table 和 skip list 各存一份，内存占用约为纯 SkipList 的 2 倍
2. **写入开销**：每次 `InsertKey` 执行 hash table 插入 + skip list CAS 插入 + memcpy，比纯 SkipList 多一次插入和一次拷贝
3. **hash_bucket_count 固定**：bucket 数量在创建时固定，不支持动态扩容；key 数量远超 bucket 数时链表变长，点查退化为 O(chain_length)
4. `CanHandleDuplicatedKey()` 返回 `false`：不在 memtable 层检测重复 key，由 RocksDB 上层处理

---

## 附录 A：文件变更完整 Diff

### A.1 接口变更（include/rocksdb/memtablerep.h）

```diff
+ // Returns true if the iterator is empty (no elements)
+ virtual bool IsEmpty() { return false; }

- virtual Iterator* GetIterator(Arena* arena = nullptr) = 0;
+ virtual Iterator* GetIterator(Arena* arena = nullptr,
+                              bool /*part_of_flush*/ = false) = 0;

+ // Return true if the current MemTableRep supports refreshing an existing
+ // iterator. Default: true
+ virtual bool IsRefreshIterSupported() const { return true; }
+
+ // Pre-create a MemTableRep in background before it is needed.
+ virtual MemTableRep* PreCreateMemTableRep() { return nullptr; }
+
+ // Post-create initialization.
+ virtual void PostCreateMemTableRep(MemTableRep*, const KeyComparator&,
+                                    Allocator*, const SliceTransform*,
+                                    Logger*) {}

+ extern MemTableRepFactory* NewHashSpdbRepFactory(
+     size_t bucket_count = 1000000, bool use_merge = true);
```

### A.2 工厂注册（table/plain/plain_table_factory.cc）

```diff
+ library.Register<MemTableRepFactory>(
+     AsRegex("HashSpdbRepFactory", "hash_spdb"),
+     [](const std::string& uri, unique_ptr<MemTableRepFactory>* guard, ...) {
+       auto colon = uri.find(":");
+       if (colon != std::string::npos) {
+         guard->reset(NewHashSpdbRepFactory(ParseSizeT(uri.substr(colon + 1))));
+       } else {
+         guard->reset(NewHashSpdbRepFactory());
+       }
+       return guard->get();
+     });
```

### A.3 现有 memtable 适配（4 个文件）

`hash_linklist_rep.cc`、`hash_skiplist_rep.cc`、`skiplistrep.cc`、`vectorrep.cc` 的 `GetIterator()` 签名统一增加 `bool /*part_of_flush*/ = false` 参数。

### A.4 stl_wrappers.h

```diff
+ inline bool operator()(const char* a, const Slice& b) const {
+   return compare_(a, b) < 0;
+ }
```
