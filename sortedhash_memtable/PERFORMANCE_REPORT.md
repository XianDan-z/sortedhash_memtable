# Hash Sorted Memtable (hash_spdb) 移植与性能测试报告

## 概要

将 Speedb 项目中的 Hash Sorted Memtable 优化独立提取并移植到 RocksDB v6.26.1，实现完整的 C++/Java/JNI 支持，并通过 db_bench 和 YCSB 进行全面的性能对比测试。

**源版本**: Speedb (基于 RocksDB 8.6.7)
**目标版本**: RocksDB v6.26.1
**测试平台**: aarch64, OpenJDK 1.8.0

---

## 补丁清单

| 编号 | 文件 | 说明 | 大小 |
|------|------|------|------|
| 0001 | `0001-hash-sorted-memtable.patch` | 原始补丁（从 Speedb 提取） | 36K |
| 0002 | `0002-rocksdb-hash-spdb-core.patch` | RocksDB C++ 核心实现 + API 适配 | 42K |
| 0003 | `0003-rocksdb-hash-spdb-jni.patch` | RocksDB Java/JNI 绑定 | 6.4K |
| 0004 | `0004-ycsb-memtable-config.patch` | YCSB Memtable 类型可配置 | 4.8K |

## 代码修改详情

### RocksDB 修改（共 16 个文件，+1155/-10）

| 类别 | 文件 | 修改内容 |
|------|------|---------|
| **核心新增** | `memtable/spdb_sorted_vector.h` | SpdbVector / SpdbVectorContainer / SpdbVectorIterator — 排序向量 + 堆归并迭代器 |
| **核心新增** | `memtable/hash_spdb_rep.cc` | SpdbHashTable + HashSpdbRep + HashSpdbRepFactory — 哈希桶 + 排序 memtable |
| **API 扩展** | `include/rocksdb/memtablerep.h` | +28/-1: 添加 PreCreateMemTableRep/PostCreateMemTableRep/IsRefreshIterSupported/IsEmpty/NewHashSpdbRepFactory；GetIterator 增加 part_of_flush 参数 |
| **构建** | `CMakeLists.txt`, `src.mk` | +2: 添加 hash_spdb_rep.cc 编译项 |
| **配置注册** | `table/plain/plain_table_factory.cc` | +14: library.Register(AsRegex("HashSpdbRepFactory", "hash_spdb")) |
| **工具适配** | `memtable/stl_wrappers.h` | +3: Compare 添加 Slice 重载 |
| **签名适配** | `memtable/skiplistrep.cc`, `vectorrep.cc`, `hash_*_rep.cc` | 6处: GetIterator 签名同步 |
| **测试适配** | `test_util/testutil.cc`, `db/db_memtable_test.cc` | 2处: GetIterator 签名同步 |
| **Java/JNI** | `java/.../HashSpdbMemTableConfig.java` | 新增: Java API 封装类 |
| **Java/JNI** | `java/rocksjni/memtablejni.cc` | +23: JNI 桥接 NewHashSpdbRepFactory |
| **Java/JNI** | `java/CMakeLists.txt` | +2: 注册 HashSpdbMemTableConfig |

### YCSB 修改（2 个文件，+46）

| 文件 | 修改内容 |
|------|---------|
| `rocksdb/pom.xml` | 添加 htrace-core4 运行时依赖 |
| `RocksDBClient.java` | 新增 PROPERTY_ROCKSDB_MEMTABLE/MEMTABLE_BUCKETS 属性；新增 getMemTableConfig() 支持 5 种 memtable 类型 |

### API 适配说明

| 原 Speedb API | RocksDB v6.26.1 | 适配方式 |
|:---|:---|:---|
| `port::RWMutexWr` | 不存在 | → `port::RWMutex` |
| `PreCreateMemTableRep()` | 不存在 | → 添加到基类（默认返回 nullptr） |
| `PostCreateMemTableRep()` | 不存在 | → 添加到基类（默认 no-op） |
| `IsRefreshIterSupported()` | 不存在 | → 添加到基类（默认 true） |
| `Iterator::IsEmpty()` | 不存在 | → 添加到基类（默认 false） |
| `ArenaTracker::ArenaStats` | 不存在 | → 去除引用，单参数 Allocate |
| `opts_list[0] ==` | `library.Register(AsRegex())` | → 适配至新注册模式 |

---

## 性能测试结果

### 1. db_bench 微基准（单线程，100万 KV）

| 场景 | SkipList (基线) | HashSpdb | 提升 |
|:---|:---|:---|:---|
| **fillrandom** (写入) | 320K ops/s | **645K ops/s** | **+102%** |
| **readrandom** (点查) | 208K ops/s | **259K ops/s** | **+25%** |
| **seekrandom** (范围扫描) | 96.5K ops/s | 97.1K ops/s | +0.6% |
| **overwrite** (更新) | 333K ops/s | **654K ops/s** | **+96%** |

### 2. db_bench 多线程（4 线程）

| 场景 | SkipList | HashSpdb | 提升 |
|:---|:---|:---|:---|
| **fillrandom** | 399K ops/s | **436K ops/s** | +9.3% |
| **readwhilewriting** | 345K ops/s | **428K ops/s** | **+24%** |

### 3. YCSB 标准工作负载（20万条记录，20万次操作）

| 工作负载 | 模式 | SkipList (ops/s) | HashSpdb (ops/s) | 差异 |
|:---|:---|:---|:---|:---|
| **A** | 50%读 + 50%写 | 45,808 | **60,150** | **+31%** |
| **B** | 95%读 + 5%写 | 54,200 | **66,711** | **+23%** |
| **C** | 100%读 | 60,204 | **68,073** | **+13%** |
| **D** | 95%读 + 5%插入 | 61,050 | **78,094** | **+28%** |
| **E** | 95%扫描 + 5%插入 | 10,032 | **535** | **-95%** ⚠️ |
| **F** | 50%读 + 50%RMW | 41,718 | **54,719** | **+31%** |

---

## 架构分析

### HashSpdb 核心设计

```
  Insert(key)
      │
      ├─► SpdbHashTable::Add()
      │     ├── MurmurHash(user_key) → bucket index
      │     └── BucketHeader: 排序单向链表插入 (WriteLock)
      │
      └─► SpdbVectorContainer::Insert()
            ├── curr_vector→Add(key)  [无锁 atomic append]
            └── vector 满 → 创建新 vector → notify sort_thread
      
  Get(key)
      │
      └─► SpdbHashTable::Get()
            ├── MurmurHash → bucket
            └── 桶内有序链表遍历 (提前终止: cmp_res > 0)
      
  Iterator:
      └─► SpdbVectorIterator
            ├── 每个 SpdbVector 内部已排序 (std::sort)
            ├── 多 vector 通过 MinHeap 归并
            └── 支持双向遍历
```

### 性能优势来源

| 优化点 | 机制 | 效果 |
|:---|:---|:---|
| 哈希桶定位 | O(1) 直接定位桶，避免全局跳表 O(log N) | 点查 +25% |
| 无锁插入 | Vector atomic append，写锁仅保护桶级链表 | 写入 +102% |
| 桶内有序链表 | cmp > 0 时提前终止，避免全桶扫描 | 点查命中加速 |
| 后台排序 | 独立 sort_thread，插入不阻塞在排序上 | 写入不降速 |
| 二分查找 | 排序后的 vector 支持 O(log n) lower_bound | Seek 定位高效 |

### 性能劣势场景

| 场景 | 原因 | 影响 |
|:---|:---|:---|
| 范围扫描 (YCSB-E) | 多 vector 归并需重建 MinHeap，每个 Next() 做 replace_top() | **-95%** |
| 高并发写入 | 桶级单向链表竞争加剧 | 多线程提升收窄至 9% |
| 迭代器方向反转 (Prev↔Next) | 需要 ReverseDirection() 重建堆 | 交替遍历性能差 |

---

## 适用场景建议

| 场景 | 推荐 | 原因 |
|:---|:---|:---|
| 写密集型 (时序数据、日志) | ✅ HashSpdb | 写入性能翻倍 |
| 点查型 (KV 缓存、Session) | ✅ HashSpdb | 点查 +25% |
| 混合读写 (OLTP) | ✅ HashSpdb | YCSB A/F +30% |
| 范围扫描型 (分析查询) | ❌ SkipList | HashSpdb 扫描性能退化 95% |
| 全表遍历 | ❌ SkipList | 堆归并开销不可接受 |

---

## 使用方式

### C++ 程序化

```cpp
#include "rocksdb/memtablerep.h"
options.memtable_factory.reset(NewHashSpdbRepFactory(400000, true));
```

### 配置字符串

```
memtable_factory=hash_spdb:400000
```

### YCSB

```bash
./bin/ycsb.sh run rocksdb -p rocksdb.memtable=hash_spdb -p rocksdb.memtable.buckets=400000
```
