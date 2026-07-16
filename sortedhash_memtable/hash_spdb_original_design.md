# RocksDB HashSpdb Memtable 原始实现详细设计文档

## 1. 概述

### 1.1 背景

HashSpdb（Hash Speedb）memtable 是由 Speedb Ltd. 开发的混合数据结构 memtable，旨在同时优化点查询（point lookup）和有序遍历（ordered scan）。其核心思想是采用 **双数据结构** 设计：

- **SpdbHashTable**：基于哈希桶的有序链表，提供 O(1) 平均时间复杂度的点查询
- **SpdbVectorContainer**：基于排序向量 + 归并堆的多段有序容器，提供有序遍历能力

该设计源自 Speedb 的商业 RocksDB 分支，在 RocksDB 6.26.1 基础上以 commit `0f4926728` 引入。

### 1.2 版本信息

- **RocksDB 版本**：6.26.1
- **引入 commit**：`0f4926728ddd95c76067a8e26d90c191edbbb50b`
- **基线 commit**：`4d57a393a` (Release 6.26.1)
- **代码来源**：Copyright (C) 2022-2023 Speedb Ltd.

### 1.3 变更统计

16 个文件，+1155 行 / -10 行。核心新增文件：

| 文件 | 行数 | 说明 |
|------|------|------|
| `memtable/hash_spdb_rep.cc` | 612 | HashSpdb memtable 核心实现（含 SpdbHashTable + SpdbVector 方法实现 + HashSpdbRep 类 + 工厂） |
| `memtable/spdb_sorted_vector.h` | 407 | 排序向量数据结构（SpdbVector + SpdbVectorContainer + SpdbVectorIterator + 归并堆） |
| `java/src/main/java/org/rocksdb/HashSpdbMemTableConfig.java` | 59 | Java 层配置类 |

---

## 2. 整体架构

### 2.1 架构总览

```
HashSpdbRep (继承 MemTableRep)
│
├── SpdbHashTable                          ← 点查询路径 (Get / Contains)
│   └── vector<BucketHeader> buckets_
│       └── BucketHeader
│           ├── RWMutex rwlock_              (读写锁)
│           ├── atomic<SpdbKeyHandle*> items_  (有序链表头)
│           └── atomic<uint32_t> elements_num_ (元素计数)
│               └── SpdbKeyHandle
│                   ├── atomic<SpdbKeyHandle*> next_  (链表指针)
│                   └── char key_[1]                    (key 数据)
│
└── shared_ptr<SpdbVectorContainer>        ← 有序遍历路径 (GetIterator / scan / flush)
    ├── list<SpdbVectorPtr> spdb_vectors_    (多段排序向量)
    │   └── SpdbVector
    │       ├── vector<const char*> items_     (key 指针数组)
    │       ├── atomic<size_t> n_elements_     (元素计数)
    │       ├── atomic<bool> sorted_           (排序标志)
    │       └── RWMutex add_rwlock_            (读写锁)
    │
    ├── atomic<SpdbVector*> curr_vector_      (当前写入向量)
    ├── SortThread (后台线程)                  (异步排序)
    │
    └── [迭代器创建时]
        ├── IterAnchors (list<SortHeapItem*>)  (各向量迭代器锚点)
        └── IterHeapInfo
            └── BinaryHeap<SortHeapItem*>       (归并堆)
```

### 2.2 数据流向

```
写入路径 (InsertKey):
  MemTable::Add()
    → Allocate(len) → 分配 SpdbKeyHandle (memtable arena)
    → 用户填充 key: [varint key_size][key][varint val_size][value]
    → InsertKey(handle)
        ├── SpdbHashTable::Add(spdb_handle)         ← 插入 hash table
        │   └── BucketHeader::Add() (WriteLock)
        │       └── 有序链表插入 (O(chain_length))
        │
        └── SpdbVectorContainer::Insert(key)        ← 插入排序向量
            ├── ReadLock spdb_vectors_add_rwlock_
            │   └── curr_vector_->Add(key)           (fetch_add + vector赋值)
            │       └── 如果 vector 已满或已排序 → 返回 false
            └── WriteLock spdb_vectors_add_rwlock_
                ├── 创建新 SpdbVector (switch_spdb_vector_limit_=10000)
                ├── MutexLock spdb_vectors_mutex_
                │   └── spdb_vectors_.push_back(new_vector)
                ├── curr_vector_ = new_vector
                ├── new_vector->Add(key)
                └── notify sort_thread_cv_           (唤醒后台排序线程)

点查询路径 (Get / Contains):
  MemTable::Get()
    → SpdbHashTable::Get()                           ← 只走 hash table
        └── BucketHeader::Get() (ReadLock)
            └── 有序链表线性扫描 (找到 >= key 的位置，回调)

有序遍历路径 (GetIterator):
  MemTableIterator 构造
    → GetIterator(arena, part_of_flush)
        └── new SpdbVectorIterator(spdb_vectors_cont_)
            └── InitIterator()
                ├── MutexLock spdb_vectors_mutex_     ← 获取锁
                ├── 如果当前 vector 非空 → 创建新 vector (切换写入)
                ├── 遍历所有已冻结的 vector → new SortHeapItem per vector
                └── SeekIter() 时:
                    ├── 每个 vector 调用 Sort()      (如果未排序则 std::sort)
                    ├── 每个 vector 调用 Seek()      (二分搜索)
                    └── 构建 BinaryHeap              (多路归并)
```

---

## 3. 核心数据结构

### 3.1 SpdbKeyHandle

```cpp
struct SpdbKeyHandle {
  SpdbKeyHandle* GetNextBucketItem() {
    return next_.load(std::memory_order_acquire);
  }
  void SetNextBucketItem(SpdbKeyHandle* handle) {
    next_.store(handle, std::memory_order_release);
  }
  std::atomic<SpdbKeyHandle*> next_{nullptr};  // hash table 链表指针
  char key_[1];                                 // flexible array, key 数据
};
```

**设计要点**：
- 用于 SpdbHashTable 的有序链表节点
- `next_` 使用 `std::atomic` + acquire/release 内存序，支持并发读取
- `key_[1]` 是 C 语言风格的 flexible array member，实际分配大小为 `sizeof(SpdbKeyHandle) + len`，key 数据紧跟在 `next_` 之后
- `sizeof(SpdbKeyHandle)` = 16 bytes（aarch64），`offsetof(key_)` = 8 bytes
- key 数据格式为 RocksDB memtable 编码：`[varint32 internal_key_size][internal_key][varint32 value_size][value]`

### 3.2 BucketHeader

```cpp
struct BucketHeader {
  port::RWMutex rwlock_;                       // 读写锁
  std::atomic<SpdbKeyHandle*> items_{nullptr}; // 有序链表头指针
  std::atomic<uint32_t> elements_num_{0};      // 链表元素计数

  bool Contains(const char* check_key, const KeyComparator& comparator,
                bool needs_lock);
  bool Add(SpdbKeyHandle* handle, const KeyComparator& comparator);
  void Get(const LookupKey& k, const KeyComparator& comparator,
           void* callback_args, bool (*callback_func)(...), bool needs_lock);
};
```

**设计要点**：

- **每个 hash bucket 一个 BucketHeader**，链表按 internal key 有序排列
- **`elements_num_` 快速判空**：`Contains` 和 `Get` 在加锁前先检查 `elements_num_ == 0`，空桶直接返回，避免锁开销
- **`Add` 有序插入**：遍历链表找到插入位置（`comparator > 0` 时插入），链表保持按 key 升序。同时做重复检测（`comparator == 0` 返回 false）
- **`Get` 范围查询**：先跳过 `< key` 的节点，然后从 `>= key` 的位置开始回调。利用有序性可以提前终止（当 `comparator > 0` 且不再匹配时）
- **`needs_lock` 参数**：当 memtable 已 immutable（`IsReadOnly()`）时，不再有写入，可以无锁读取（`needs_lock=false`）

**Add 操作流程**：
```
1. WriteLock(rwlock_)
2. 从 items_ 头开始遍历
3. 对于每个节点 iter:
   - 如果 comparator(iter->key, handle->key) == 0 → 重复 key，返回 false
   - 如果 comparator(iter->key, handle->key) > 0 → 找到插入位置，break
   - 否则 prev = iter, iter = iter->next
4. handle->next = iter (插入到 iter 之前)
5. if prev: prev->next = handle, else: items_ = handle
6. elements_num_++
7. 返回 true
```

**Get 操作流程**：
```
1. (可选) ReadLock(rwlock_)
2. 从 items_ 头开始遍历
3. 跳过 comparator(iter->key, k.internal_key()) < 0 的节点
4. 从 >= k 的位置开始，对每个节点调用 callback_func
5. callback_func 返回 false 时停止
6. (可选) ReadUnlock(rwlock_)
```

### 3.3 SpdbHashTable

```cpp
struct SpdbHashTable {
  std::vector<BucketHeader> buckets_;

  SpdbHashTable(size_t n_buckets) : buckets_(n_buckets) {}

  bool Add(SpdbKeyHandle* handle, const KeyComparator& comparator);
  bool Contains(const char* key, const KeyComparator& comparator,
                bool needs_lock) const;
  void Get(const LookupKey& k, const KeyComparator& comparator,
           void* callback_args, bool (*callback_func)(...),
           bool needs_lock) const;

 private:
  static size_t GetHash(const Slice& user_key_without_ts);
  static Slice UserKeyWithoutTimestamp(const Slice& internal_key,
                                       const KeyComparator& compare);
  BucketHeader* GetBucket(const char* key, const KeyComparator& comparator) const;
  BucketHeader* GetBucket(const Slice& internal_key,
                          const KeyComparator& comparator) const;
};
```

**设计要点**：

- **固定大小 bucket 数组**：构造时分配 `n_buckets` 个 BucketHeader，默认 1,000,000（工厂默认 400,000）
- **Hash 函数**：`MurmurHash(user_key_without_ts, 0)`，只对 user key（去除 timestamp）做 hash，不包含 seqno。这意味着相同 user key 不同 seqno 的 entry 落在同一个 bucket
- **Timestamp 处理**：`UserKeyWithoutTimestamp` 从 internal key 中提取 user key 并剥离 timestamp，使用 `ExtractUserKeyAndStripTimestamp(internal_key, ts_sz)`
- **Bucket 定位**：`hash % buckets_.size()`
- **不支持动态扩容**：bucket 数量固定，当 key 数量远超 bucket 数时链表变长，点查退化为 O(chain_length)

### 3.4 SpdbVector

```cpp
class SpdbVector {
 public:
  using Vec = std::vector<const char*>;
  using Iterator = Vec::iterator;

  // 构造：预分配 switch_spdb_vector_limit_ 大小的 vector
  SpdbVector(size_t switch_spdb_vector_limit);

  bool Add(const char* key);     // 原子追加
  bool Sort(const KeyComparator&); // std::sort 排序
  Iterator SeekForward(const KeyComparator&, const Slice* seek_key);  // 二分搜索 >=
  Iterator SeekBackword(const KeyComparator&, const Slice* seek_key); // 二分搜索 <=
  Iterator Seek(const KeyComparator&, const Slice* seek_key, bool up);
  bool Next(Iterator& iter);  // ++iter
  bool Prev(Iterator& iter);  // --iter

 private:
  Vec items_;                         // key 指针数组
  std::atomic<size_t> n_elements_;    // 实际元素数
  std::atomic<bool> sorted_;          // 是否已排序
  std::list<SpdbVectorPtr>::iterator iter_;  // 在 container 列表中的位置
  port::RWMutex add_rwlock_;          // 读写锁
};
```

**设计要点**：

- **预分配数组**：构造时预分配 `switch_spdb_vector_limit_`（默认 10000）个 `const char*` 槽位
- **无锁追加**：`Add` 使用 `n_elements_.fetch_add(1)` 原子递增获取写入位置，然后 `items_[location] = key`。这是无锁的，多线程可以并发写入不同位置
- **写满切换**：当 `location >= items_.size()` 时返回 false，触发 container 创建新 vector
- **写后即冻结**：一旦 `Sort()` 被调用，`sorted_` 设为 true，后续 `Add` 检测到 `sorted_` 后直接返回 false。排序后的 vector 成为不可变快照
- **Sort 操作**：获取 `WriteLock`，执行 `std::sort`，设置 `sorted_`。有 double-check（先无锁检查 `sorted_`，再加锁检查）
- **SeekForward**：使用 `std::lower_bound` 二分搜索找到第一个 `>= seek_key` 的元素。有两个快速路径：如果 `seek_key` 小于首元素则返回 `begin()`，大于末元素则返回 `end()`
- **SeekBackword**：使用 `std::lower_bound` 找到第一个 `>= seek_key` 的位置，然后 `--ret` 得到最后一个 `<= seek_key` 的位置

**Add 操作流程**：
```
1. ReadLock(add_rwlock_)          ← 读锁，与 Sort 的 WriteLock 互斥
2. if sorted_ → return false      ← 已排序，不可写入
3. location = n_elements_.fetch_add(1, relaxed)  ← 原子获取写入位置
4. if location < items_.size():
     items_[location] = key       ← 写入
     return true
   else:
     return false                 ← 已满
```

**Sort 操作流程**：
```
1. if sorted_.load(acquire) → return true    ← 快速路径，已排序
2. WriteLock(add_rwlock_)                     ← 写锁，阻止并发 Add
3. if n_elements_ == 0 → return false
4. if sorted_.load(relaxed) → return true    ← double-check
5. num_elements = min(n_elements_, items_.size())
6. items_.resize(num_elements)                ← 截断未写满的部分
7. std::sort(items_.begin(), items_.end(), Compare(comparator))
8. sorted_.store(true, release)               ← 标记已排序
9. return true
```

### 3.5 SortHeapItem

```cpp
class SortHeapItem {
 public:
  SpdbVectorPtr spdb_vector_;              // 指向所属 SpdbVector
  SpdbVector::Iterator curr_iter_;         // 当前迭代位置

  bool Valid() const;                      // iter != end
  const char* Key() const;                 // *curr_iter_
  bool Next() { return spdb_vector_->Next(curr_iter_); }
  bool Prev() { return spdb_vector_->Prev(curr_iter_); }
};
```

**设计要点**：
- 每个 SortHeapItem 对应一个 SpdbVector 的当前迭代位置
- 在 `InitIterator` 时通过 `new SortHeapItem(vector, vector->End())` 创建
- 在 `SeekIter` 时更新 `curr_iter_` 为 seek 后的位置
- 在 `~SpdbVectorIterator` 时通过 `delete` 销毁
- **性能问题**：每次创建迭代器都 new V 个 SortHeapItem（V = vector 数量），销毁时 delete V 个。高并发 scan 时这是主要瓶颈

### 3.6 IterHeapInfo / BinaryHeap

```cpp
using IterHeap = BinaryHeap<SortHeapItem*, IteratorComparator>;

class IterHeapInfo {
  std::unique_ptr<IterHeap> iter_heap_;    // 归并堆
  const KeyComparator& comparator_;

  void Reset(bool up_iter_direction);      // 重建堆（new IterHeap）
  const char* Key() const;                 // 堆顶 key
  SortHeapItem* Get();                     // 获取堆顶 item
  void Update(SortHeapItem* sort_item);    // 替换/弹出堆顶
  void Insert(SortHeapItem* sort_item);    // 插入堆
};
```

**设计要点**：
- 使用 RocksDB 内置的 `BinaryHeap` 实现多路归并
- `IteratorComparator` 根据方向（up/down）定义堆序
- `Reset` 每次创建新的 `IterHeap`（`unique_ptr::reset`），方向变化时必须重建
- `Update` 替换堆顶：如果 item 仍有有效数据则 `replace_top`，否则 `pop`
- **性能问题**：`Reset` 每次 `new IterHeap` 涉及内存分配，频繁 Seek 时开销大

### 3.7 SpdbVectorContainer

```cpp
class SpdbVectorContainer {
  port::RWMutex spdb_vectors_add_rwlock_;           // 写入读写锁
  port::Mutex spdb_vectors_mutex_;                  // vector 列表互斥锁
  std::list<SpdbVectorPtr> spdb_vectors_;           // 排序向量列表
  std::atomic<SpdbVector*> curr_vector_;            // 当前写入向量
  const KeyComparator& comparator_;
  const size_t switch_spdb_vector_limit_;           // 10000
  std::atomic<bool> immutable_;                     // 只读标志
  std::atomic<size_t> num_elements_;                // 总元素数
  bool use_merge_;                                  // 合并模式标志
  port::Thread sort_thread_;                        // 后台排序线程
  std::mutex sort_thread_mutex_;                    // 排序线程互斥锁
  std::condition_variable sort_thread_cv_;          // 排序线程条件变量
};
```

**设计要点**：

#### 3.7.1 写入流程 (Insert)

```
Insert(key):
1. num_elements_.fetch_add(1)
2. ReadLock(spdb_vectors_add_rwlock_)
   → InternalInsert(key): curr_vector_->Add(key)
   → 如果成功，返回
3. WriteLock(spdb_vectors_add_rwlock_)       ← 升级为写锁
   → InternalInsert(key): 再次尝试（可能其他线程已创建新 vector）
   → 如果仍失败：
     a. MutexLock(spdb_vectors_mutex_)
     b. new SpdbVector(switch_spdb_vector_limit_)
     c. spdb_vectors_.push_back(new_vector)
     d. curr_vector_ = new_vector
     e. notify_sort_thread = true
     f. InternalInsert(key)  ← 写入新 vector
4. if notify_sort_thread → sort_thread_cv_.notify_one()
```

**锁层次**：
- `spdb_vectors_add_rwlock_`：保护 `curr_vector_` 的切换，读锁允许并发写入当前 vector
- `spdb_vectors_mutex_`：保护 `spdb_vectors_` 列表的修改（push_back）
- 两把锁嵌套使用：先获取 `spdb_vectors_add_rwlock_` WriteLock，再获取 `spdb_vectors_mutex_`

#### 3.7.2 后台排序线程 (SortThread)

```
SortThread():
  unique_lock(sort_thread_mutex_)
  sort_iter_anchor = spdb_vectors_.begin()

  loop:
    wait(sort_thread_cv_)           ← 等待通知
    if immutable_ → break

    last = prev(spdb_vectors_.end())
    if last == sort_iter_anchor → continue  ← 没有新 vector

    for each vector from sort_iter_anchor to last:
      vector->Sort(comparator_)     ← 排序

    sort_iter_anchor = last         ← 更新已排序位置
```

**设计要点**：
- 后台线程在容器构造时启动，析构时（`MarkReadOnly` 后）退出
- 每次有新 vector 创建时通过 `notify_one` 唤醒
- 对所有尚未排序的 vector 执行 `Sort()`
- `Sort()` 内部有 `sorted_` 检查，已排序的 vector 不会重复排序

#### 3.7.3 迭代器初始化 (InitIterator)

```
InitIterator(iter_anchor, part_of_flush):
1. if IsEmpty(part_of_flush) → return false
2. last_iter = curr_vector_->GetVectorListIter()
3. if not immutable:
     if not last_iter->IsEmpty():
       a. MutexLock(spdb_vectors_mutex_)     ← 获取锁
       b. new SpdbVector(switch_spdb_vector_limit_)
       c. spdb_vectors_.push_back(new_vector)
       d. curr_vector_ = new_vector           ← 切换写入目标
       e. notify_sort_thread = true
     else:
       --last_iter                            ← 当前 vector 为空，回退一个
4. ++last_iter                                ← 排除当前（可能仍在写入的）vector
5. InitIterator(iter_anchor, begin(), last_iter):
   for each vector from begin to last:
     new SortHeapItem(vector, vector->End())
     iter_anchor.push_back(item)
6. if notify_sort_thread → sort_thread_cv_.notify_one()
7. return true
```

**设计要点**：
- **冻结当前 vector**：如果当前写入 vector 非空，创建新 vector 并切换 `curr_vector_`。旧 vector 被冻结（不再有新写入），可以安全排序和遍历
- **`part_of_flush` 参数**：控制 `IsEmpty` 的行为。当 `use_merge_=false` 且非 flush 场景时，返回空（不参与迭代）。这是为了支持 merge 操作时的特殊语义
- **锁竞争**：`spdb_vectors_mutex_` 在 `InitIterator` 和 `Insert` 中都会获取。32 线程并发 scan 时，每次 Seek 都创建新迭代器，导致严重锁竞争

#### 3.7.4 Seek 操作 (SeekIter)

```
SeekIter(iter_anchor, iter_heap_info, seek_key, up_direction):
1. iter_heap_info->Reset(up_direction)        ← 重建归并堆
2. for each item in iter_anchor:
     a. if item->spdb_vector_->Sort(comparator_):  ← 确保已排序
        item->curr_iter_ = item->spdb_vector_->Seek(comparator_, seek_key, up_direction)
        if item->Valid():
          iter_heap_info->Insert(item)         ← 插入归并堆
```

**设计要点**：
- 每次 Seek 都调用 `Sort()` 确保所有 vector 已排序。`Sort()` 内部有 `sorted_` 快速检查，已排序的 vector 不会重复排序
- 每个 vector 执行 `Seek()`（二分搜索）定位到 `>= seek_key`（或 `<=`）的位置
- 所有有效结果插入 `BinaryHeap`，堆顶是最小（或最大）key
- `Reset` 每次创建新的 `BinaryHeap`，涉及内存分配

### 3.8 SpdbVectorIterator

```cpp
class SpdbVectorIterator : public MemTableRep::Iterator {
  shared_ptr<SpdbVectorContainer> spdb_vectors_cont_holder_;  // 持有容器引用
  SpdbVectorContainer* spdb_vectors_cont_;                    // 裸指针
  IterAnchors iter_anchor_;                                   // SortHeapItem 列表
  IterHeapInfo iter_heap_info_;                               // 归并堆
  bool up_iter_direction_;                                    // 方向
  bool is_empty_;                                             // 空标志
};
```

**生命周期**：

```
构造:
  InitIterator(iter_anchor_, part_of_flush)
  → 获取锁、冻结当前 vector、创建 SortHeapItem per vector

Seek(key):
  Reset(true)                    → 重建归并堆 (new BinaryHeap)
  InternalSeek(&key)             → SeekIter: Sort + binary search + 堆构建

Next():
  if 方向为 down → ReverseDirection(true): 重新 Seek
  Advance():
    sort_item = heap.top()
    sort_item->Next()             → ++iter
    heap.Update(sort_item)        → replace_top 或 pop

~析构():
  for each item in iter_anchor_:
    delete item                  ← 释放所有 SortHeapItem
```

**性能特征**：

| 操作 | 时间复杂度 | 说明 |
|------|-----------|------|
| 构造 | O(V) + 锁 | V = vector 数量，每个 vector 创建一个 SortHeapItem |
| Seek | O(V log V) + O(V log N) + O(V) | V 个 vector 各排序 + 二分 + 堆构建 |
| Next | O(log V) | 堆顶替换 |
| 析构 | O(V) | delete V 个 SortHeapItem |

**关键问题**：由于 `IsRefreshIterSupported()` 返回 `false`，RocksDB 每次 Seek 都销毁旧迭代器、创建新迭代器。上述所有开销在每次 Seek 时重复执行。

### 3.9 HashSpdbRep

```cpp
class HashSpdbRep : public MemTableRep {
  SpdbHashTable spdb_hash_table_;                        // 点查数据结构
  std::shared_ptr<SpdbVectorContainer> spdb_vectors_cont_; // 有序遍历数据结构
};

// 工厂标志
bool IsInsertConcurrentlySupported() const override { return true; }
bool CanHandleDuplicatedKey() const override { return true; }
bool IsRefreshIterSupported() const override { return false; }  // ← 关键
```

**`IsRefreshIterSupported() = false` 的含义**：

RocksDB 的 `MemTableRepFactory` 接口在原始 6.26.1 中没有 `IsRefreshIterSupported()` 方法。此方法是 HashSpdb 引入的新接口（在 `include/rocksdb/memtablerep.h` 中新增）。当返回 `false` 时，RocksDB 上层在每次 Seek 操作时都会销毁旧 memtable 迭代器并创建新迭代器，而不是复用现有迭代器。

这是原始实现 scan 性能极差的**根本原因**——每次 Seek 都触发完整的 `InitIterator`（锁 + 冻结 vector + 创建 SortHeapItem）+ `SeekIter`（排序 + 二分 + 堆构建）+ 析构（delete SortHeapItem）。

---

## 4. PreCreate / PostCreate 机制

HashSpdb 引入了 `PreCreateMemTableRep()` / `PostCreateMemTableRep()` 接口，支持 memtable 的预创建：

```cpp
MemTableRep* HashSpdbRepFactory::PreCreateMemTableRep() {
  // 在后台线程提前创建，不含 comparator/allocator
  return new HashSpdbRep(nullptr, options_.hash_bucket_count);
}

void HashSpdbRepFactory::PostCreateMemTableRep(
    MemTableRep* switch_mem, const KeyComparator& compare,
    Allocator* allocator, ...) {
  // 在需要时完成初始化
  static_cast<HashSpdbRep*>(switch_mem)
      ->PostCreate(compare, allocator, options_.use_merge);
}
```

**流程**：
1. `PreCreateMemTableRep()` 在后台线程调用，创建 `HashSpdbRep` 但不初始化 `SpdbVectorContainer`（因为 comparator/allocator 尚未确定）
2. 当 RocksDB 需要新 memtable 时，取出预创建的对象，调用 `PostCreateMemTableRep()` 完成初始化
3. `PostCreate` 设置 `allocator_` 并创建 `SpdbVectorContainer`

**设计目的**：减少 memtable 切换时的延迟。SpdbVectorContainer 构造时会启动排序线程，预创建可以提前完成这些工作。

---

## 5. use_merge 机制

```cpp
struct HashSpdbRepOptions {
  size_t hash_bucket_count;
  bool use_merge;    // 默认 true
};
```

`use_merge` 控制 `SpdbVectorContainer::IsEmpty(part_of_flush)` 的行为：

```cpp
bool IsEmpty(bool part_of_flush) const {
  return num_elements_.load() == 0 || (!use_merge_ && !part_of_flush);
}
```

- `use_merge=true`（默认）：`IsEmpty` 只检查元素数。所有 vector 都参与迭代器
- `use_merge=false`：非 flush 场景下 `IsEmpty` 返回 true，迭代器为空。仅 flush 时才参与迭代

**设计目的**：当 `use_merge=false` 时，SpdbVectorContainer 的数据不参与普通读取的迭代器（只有 hash table 提供点查）。仅在对 flush 的迭代器中才包含 SpdbVectorContainer 数据。这适用于某些不需要有序遍历的场景，可以跳过 vector 的排序和归并开销。

---

## 6. 并发安全设计

### 6.1 写入并发

| 操作 | 锁 | 粒度 |
|------|-----|------|
| `BucketHeader::Add` | `rwlock_` WriteLock | per-bucket |
| `SpdbVector::Add` | `add_rwlock_` ReadLock | per-vector |
| `SpdbVectorContainer::Insert` (新建 vector) | `spdb_vectors_add_rwlock_` WriteLock + `spdb_vectors_mutex_` | 全局 |
| `SpdbVectorContainer::Insert` (正常写入) | `spdb_vectors_add_rwlock_` ReadLock | 全局读 |

**写入路径分析**：
- 正常情况下（vector 未满），`Insert` 只需 `ReadLock`（允许并发写入不同位置）
- vector 满时，升级为 `WriteLock`，创建新 vector（`MutexLock` 保护列表修改）
- `BucketHeader::Add` 的 `WriteLock` 是 per-bucket 粒度，不同 bucket 无竞争

### 6.2 读取并发

| 操作 | 锁 | 说明 |
|------|-----|------|
| `BucketHeader::Get/Contains` | `rwlock_` ReadLock | 多线程可并发读同一 bucket |
| `SpdbVector::Sort` | `add_rwlock_` WriteLock | 与 `Add` 互斥 |
| `SpdbVectorContainer::InitIterator` | `spdb_vectors_mutex_` | 与 `Insert` 的新建 vector 路径竞争 |

### 6.3 `needs_lock` 机制

```cpp
// HashSpdbRep::Contains
bool Contains(const char* key) const {
  return spdb_hash_table_.Contains(key, GetComparator(),
                                   !spdb_vectors_cont_->IsReadOnly());
}
```

- `IsReadOnly() = false`（活跃 memtable）→ `needs_lock = true` → 加 ReadLock
- `IsReadOnly() = true`（immutable memtable）→ `needs_lock = false` → 无锁读取

**前提**：`MarkReadOnly()` 后不再有 `Add` 操作，因此无锁读取是安全的。

---

## 7. HashSpdbRepFactory

### 7.1 工厂配置

```cpp
class HashSpdbRepFactory : public MemTableRepFactory {
  HashSpdbRepOptions options_;  // {hash_bucket_count, use_merge}

  // 默认值
  explicit HashSpdbRepFactory(
      size_t hash_bucket_count = 400000,  // 工厂默认 400K
      bool use_merge = true);

  // 接口标志
  bool IsInsertConcurrentlySupported() const override { return true; }
  bool CanHandleDuplicatedKey() const override { return true; }
  bool IsRefreshIterSupported() const override { return false; }
};
```

**注意**：工厂构造函数默认 `hash_bucket_count = 400000`，但 `NewHashSpdbRepFactory` 函数签名默认 `bucket_count = 1000000`。通过 Options 文件或 `hash_spdb:N` 语法可以指定。

### 7.2 工厂注册

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

支持通过 `MemTableRepFactory::CreateFromString` 按以下格式匹配：
- `HashSpdbRepFactory`（类名）
- `hash_spdb`（昵称）
- `hash_spdb:400000`（昵称 + bucket_count）

---

## 8. 性能特征分析

### 8.1 时间复杂度

| 操作 | 时间复杂度 | 说明 |
|------|-----------|------|
| 点查 `Get` | O(1) + O(chain) | hash 定位 + 桶内链表遍历 |
| `Contains` | O(1) + O(chain) | 同上 |
| 写入 `InsertKey` | O(1) + O(chain) + O(1) | hash 插入 + vector append |
| 迭代器构造 | O(V) + 锁 | V = vector 数量 |
| `Seek` | O(V·sort + V·logN + V) | V 个 vector 排序 + 二分 + 堆构建 |
| `Next` | O(log V) | 堆顶替换 |
| 迭代器析构 | O(V) | delete V 个 SortHeapItem |

### 8.2 已知性能问题

#### 8.2.1 Scan 性能极差

**根因**：`IsRefreshIterSupported() = false` → 每次 Seek 重建迭代器

**32 线程并发 scan 的热点分布**（db_bench mixgraph, 1M keys）：

| 热点 | 占比 | 说明 |
|------|------|------|
| `pthread_rwlock_wrlock` | 25.61% | `InitIterator` 中 `spdb_vectors_mutex_` 写锁竞争 |
| `SpdbVectorContainer::InitIterator` | 19.80% | 锁 + 冻结 vector + 创建 SortHeapItem |
| `SpdbVectorIterator::~SpdbVectorIterator` | 17.14% | delete SortHeapItem |
| `pthread_rwlock_unlock` | 7.41% | 锁释放 |
| `_int_free` + `malloc` | 10.67% | SortHeapItem 内存分配/释放 |
| `SeekIter` + `Sort` | 10.93% | 实际计算 |

**Scan 吞吐量**：2,908 ops/s（对比 skip_list 的 2,989,145 ops/s，差 1000 倍）

#### 8.2.2 锁竞争

`spdb_vectors_mutex_` 在以下路径中竞争：
- `Insert`：新建 vector 时获取（写入路径）
- `InitIterator`：冻结当前 vector 时获取（每次 Seek）

32 线程并发时，InitIterator 的 mutex 竞争成为主要瓶颈。

#### 8.2.3 内存分配开销

每次 Seek：
- `new` V 个 `SortHeapItem`（V = 10000 条/vector → 1M keys = 100 vectors）
- `new` 1 个 `BinaryHeap`（`IterHeapInfo::Reset`）
- `delete` V 个 `SortHeapItem`（析构）

1M keys / 10000 per vector = 100 vectors → 每次 Seek 100 次 new + 100 次 delete。

### 8.3 优势场景

| 场景 | 性能 | 原因 |
|------|------|------|
| 点查（WL-C 100% Read） | 优秀 | hash table O(1) 查找 |
| 写入（Load） | 优秀 | vector append O(1) + hash 有序插入 |
| 混合读写（WL-A/B/D/F） | 优秀 | 点查走 hash table，写入走 vector append |
| Scan（WL-E 95% Scan） | **极差** | 迭代器重建开销 + 锁竞争 |

---

## 9. 接口变更

### 9.1 MemTableRep 接口变更

```cpp
// include/rocksdb/memtablerep.h

class MemTableRep {
  class Iterator {
+   virtual bool IsEmpty() { return false; }  // 新增
  };

- virtual Iterator* GetIterator(Arena* arena = nullptr) = 0;
+ virtual Iterator* GetIterator(Arena* arena = nullptr,
+                              bool /*part_of_flush*/ = false) = 0;  // 新增参数
};

class MemTableRepFactory {
+ virtual bool IsRefreshIterSupported() const { return true; }  // 新增
+ virtual MemTableRep* PreCreateMemTableRep() { return nullptr; }  // 新增
+ virtual void PostCreateMemTableRep(...) {}  // 新增
};

+ extern MemTableRepFactory* NewHashSpdbRepFactory(
+     size_t bucket_count = 1000000, bool use_merge = true);
```

### 9.2 现有 memtable 适配

4 个现有 memtable 实现的 `GetIterator()` 签名统一增加 `bool /*part_of_flush*/ = false` 参数：
- `memtable/skiplistrep.cc` (SkipListRep)
- `memtable/hash_linklist_rep.cc` (HashLinkListRep)
- `memtable/hash_skiplist_rep.cc` (HashSkipListRep)
- `memtable/vectorrep.cc` (VectorRep)

测试代码 `db/db_memtable_test.cc` 和 `test_util/testutil.cc` 中的 Mock 也同步适配。

### 9.3 stl_wrappers.h

```cpp
struct Compare : private Base {
  inline bool operator()(const char* a, const char* b) const { ... }
+ inline bool operator()(const char* a, const Slice& b) const {
+   return compare_(a, b) < 0;
+ }
};
```

新增 `operator()(const char*, const Slice&)` 重载，用于 `SpdbVector::SeekForward/SeekBackword` 中的 `std::lower_bound`。

---

## 10. Java/JNI 绑定

### 10.1 HashSpdbMemTableConfig.java

```java
public class HashSpdbMemTableConfig extends MemTableConfig {
  public static final int DEFAULT_BUCKET_COUNT = 1000000;

  public HashSpdbMemTableConfig() { bucketCount_ = DEFAULT_BUCKET_COUNT; }
  public HashSpdbMemTableConfig setBucketCount(final long count);
  public long bucketCount();

  @Override
  protected long newMemTableFactoryHandle() {
    return newMemTableFactoryHandle(bucketCount_);
  }
  private native long newMemTableFactoryHandle(long bucketCount);
}
```

### 10.2 JNI 桥接 (memtablejni.cc)

```cpp
jlong Java_org_rocksdb_HashSpdbMemTableConfig_newMemTableFactoryHandle(
    JNIEnv* env, jobject, jlong jbucket_count) {
  auto s = JniUtil::check_if_jlong_fits_size_t(jbucket_count);
  if (s.ok()) {
    return reinterpret_cast<jlong>(
        NewHashSpdbRepFactory(static_cast<size_t>(jbucket_count)));
  }
  IllegalArgumentExceptionJni::ThrowNew(env, s);
  return 0;
}
```

注意：JNI 层只传递 `bucket_count`，`use_merge` 参数使用工厂构造函数默认值 `true`。

---

## 11. 内存管理

### 11.1 内存分配

所有内存通过 RocksDB 的 `Allocator`（通常是 Arena）分配：

| 分配点 | 大小 | 用途 |
|--------|------|------|
| `HashSpdbRep::Allocate` | `sizeof(SpdbKeyHandle) + len` | hash table 节点 + key data |
| `SpdbVectorContainer` 构造 | `sizeof(SpdbVectorContainer)` | 容器（含后台线程） |
| `SpdbVector` 构造 | `switch_spdb_vector_limit_ * sizeof(const char*)` | vector 预分配 |
| `SortHeapItem` (每次 Seek) | `sizeof(SortHeapItem)` × V | 迭代器锚点（heap 分配） |
| `BinaryHeap` (每次 Seek) | 动态 | 归并堆（heap 分配） |

### 11.2 内存生命周期

- `SpdbKeyHandle` 和 `SpdbVector::items_` 中的 key 指针都指向 Arena 内存，随 memtable 释放时统一回收
- `SpdbVectorContainer` 通过 `shared_ptr` 管理，迭代器持有引用计数
- `SortHeapItem` 和 `BinaryHeap` 在 heap 上分配，迭代器析构时释放

### 11.3 内存开销

相比纯 SkipList，HashSpdb 额外内存开销：
- 每个 key 多一份 `SpdbKeyHandle` 头部（16 bytes）
- `SpdbHashTable` 的 bucket 数组（`bucket_count * sizeof(BucketHeader)`）
- `SpdbVectorContainer` 的 vector 列表（每个 vector 预分配 10000 × 8 bytes = 80KB）
- 每次迭代的 SortHeapItem + BinaryHeap（临时开销）

---

## 附录 A：完整类关系图

```
MemTableRepFactory
  └── HashSpdbRepFactory
        ├── options_: HashSpdbRepOptions {hash_bucket_count, use_merge}
        ├── CreateMemTableRep() → new HashSpdbRep(...)
        ├── PreCreateMemTableRep() → new HashSpdbRep(nullptr, bucket_count)
        └── PostCreateMemTableRep() → HashSpdbRep::PostCreate(...)

MemTableRep
  └── HashSpdbRep
        ├── spdb_hash_table_: SpdbHashTable
        │     └── buckets_: vector<BucketHeader>
        │           └── BucketHeader
        │                 ├── rwlock_: RWMutex
        │                 ├── items_: atomic<SpdbKeyHandle*>
        │                 └── elements_num_: atomic<uint32_t>
        │                       └── SpdbKeyHandle {next_, key_[]}
        │
        ├── spdb_vectors_cont_: shared_ptr<SpdbVectorContainer>
        │     ├── spdb_vectors_: list<shared_ptr<SpdbVector>>
        │     │     └── SpdbVector {items_, n_elements_, sorted_, add_rwlock_}
        │     ├── curr_vector_: atomic<SpdbVector*>
        │     ├── sort_thread_: Thread (后台排序)
        │     └── InitIterator() / SeekIter() / Insert()
        │
        ├── cmp_: const KeyComparator&
        └── immutable_: atomic<bool>

MemTableRep::Iterator
  └── SpdbVectorIterator
        ├── spdb_vectors_cont_holder_: shared_ptr<SpdbVectorContainer>
        ├── iter_anchor_: list<SortHeapItem*>
        │     └── SortHeapItem {spdb_vector_, curr_iter_}
        ├── iter_heap_info_: IterHeapInfo
        │     └── iter_heap_: unique_ptr<BinaryHeap<SortHeapItem*>>
        ├── up_iter_direction_: bool
        └── is_empty_: bool
```

## 附录 B：文件结构

```
memtable/
├── hash_spdb_rep.cc          # 612 行
│   ├── SpdbKeyHandle          (struct)
│   ├── BucketHeader           (struct)
│   ├── SpdbHashTable          (struct)
│   ├── SpdbVector 方法定义     (Add, Sort, SeekForward, SeekBackword, Seek)
│   ├── SpdbVectorContainer 方法定义 (InternalInsert, Insert, InitIterator, SeekIter, SortThread)
│   ├── HashSpdbRep            (class, 继承 MemTableRep)
│   ├── HashSpdbRepOptions     (struct)
│   └── HashSpdbRepFactory     (class, 继承 MemTableRepFactory)
│
└── spdb_sorted_vector.h       # 407 行
    ├── SpdbVector             (class)
    ├── SortHeapItem           (class)
    ├── IteratorComparator     (class)
    ├── IterHeap               (typedef BinaryHeap<...>)
    ├── IterHeapInfo           (class)
    ├── IterAnchors             (typedef list<SortHeapItem*>)
    ├── SpdbVectorContainer    (class)
    └── SpdbVectorIterator     (class, 继承 MemTableRep::Iterator)
```
