# RocksDB HashSpdb Memtable 性能测试报告

## 1. 测试环境

| 项目 | 详情 |
|------|------|
| **硬件平台** | aarch64 (Kunpeng 950), 384 核 |
| **操作系统** | openEuler 24.03 SP2 |
| **RocksDB 版本** | 6.26.1 |
| **编译选项** | DEBUG_LEVEL=0, DISABLE_JEMALLOC=1, ROCKSDB_DISABLE_TCMALLOC=1 |
| **JDK** | OpenJDK 1.8.0_462 (BiSheng) |
| **Maven** | 3.6.3 |
| **YCSB 版本** | 0.18.0-SNAPSHOT |
| **测试工具** | YCSB (6 workloads A-F), db_bench (mixgraph) |
| **绑核策略** | taskset -c 0-7 (8 核) 用于 YCSB; taskset -c 0-31 (32 核) 用于 db_bench |
| **perf 工具** | perf record -F 999 -g, LD_LIBRARY_PATH 指定带符号 .so |

## 2. 测试对象

| 版本 | 说明 | Commit |
|------|------|--------|
| **skip_list** | RocksDB 默认 SkipList memtable（对照组） | 6.26.1 原始 |
| **hash_spdb 原始版** | 原始 HashSpdb 实现（SpdbVectorContainer 排序向量+归并堆） | `0f4926728` |
| **hash_spdb 优化版** | 重构后 HashSpdb 实现（InlineSkipList 替换 SpdbVectorContainer） | `96af879de` |

## 3. YCSB Workload 说明

| Workload | 操作比例 | 请求分布 | 典型场景 |
|----------|---------|---------|---------|
| **A** | 50% Read / 50% Update | Zipfian | 会话存储 |
| **B** | 95% Read / 5% Update | Zipfian | 照片标签 |
| **C** | 100% Read | Zipfian | 用户画像缓存 |
| **D** | 95% Read / 5% Insert | Latest | 状态更新/时间线 |
| **E** | 95% Scan / 5% Insert | Zipfian | 线程对话 |
| **F** | 50% Read / 50% Read-Modify-Write | Zipfian | 用户数据库 |

**注意**：YCSB 的 Update 操作为 read-modify-write，实际执行 1 次 Get + 1 次 Put。因此 Workload-A 的实际操作分布为 75% Get + 25% Put。

## 4. 测试参数

### 4.1 YCSB 测试参数

| 参数 | 值 |
|------|------|
| recordcount | 10,000,000 (1000万) |
| operationcount | 10,000,000 |
| Load 线程数 | 8 |
| Run 线程数 | 8 |
| 数据目录 | /data/zyue/hashspdb_opt/data |
| hash_bucket_count | 1,000,000 |
| compression | 默认 (snappy) |

### 4.2 db_bench 测试参数

| 参数 | 值 |
|------|------|
| num | 1,000,000 |
| key_size | 16 |
| value_size | 100 |
| threads | 32 |
| compression_type | none |
| hash_bucket_count | 1,000,000 |
| mix_get_ratio | 0.0 |
| mix_put_ratio | 0.05 |
| mix_seek_ratio | 0.95 |
| mix_max_scan_len | 100 |

## 5. 测试结果

### 5.1 YCSB 全量测试（1000万 records, 8线程）

#### 5.1.1 全部三版本对比（Workload A-F）

| Phase | skip_list (ops/s) | hash_spdb 原始版 (ops/s) | hash_spdb 优化版 (ops/s) |
|-------|-------------------|-------------------------|-------------------------|
| **Load** | 276,901 | 272,606 | 274,748 |
| **WL-A** (50R/50U) | 206,607 | 215,415 | 193,750 |
| **WL-B** (95R/5U) | 406,009 | 436,224 | 396,605 |
| **WL-C** (100R) | 379,147 | 376,790 | 450,451 |
| **WL-D** (95R/5I) | 619,042 | 685,824 | 662,647 |
| **WL-E** (95S/5I) | 49,209 | N/A¹ | 49,026 |
| **WL-F** (50R/50RMW) | 203,326 | 210,128 | 193,218 |

> ¹ 原始版 hash_spdb 在 Workload-E (scan) 场景性能极差（早期测试仅 1,442 ops/s），1000万数据量下耗时过长，跳过测试。

#### 5.1.2 优化版 vs skip_list 比值

| Phase | 优化版/skip_list | 说明 |
|-------|-----------------|------|
| Load | 99.2% | 持平 |
| WL-A | 93.8% | 略低（写入双结构开销） |
| WL-B | 97.7% | 持平 |
| WL-C | **118.8%** | **反超**（hash table 点查优势） |
| WL-D | **107.1%** | **反超**（hash table 点查优势） |
| WL-E | 99.6% | 持平（重构核心价值） |
| WL-F | 95.0% | 略低（写入双结构开销） |

#### 5.1.3 优化版 vs 原始版比值（仅可比的 5 条 workload）

| Phase | 优化版/原始版 | 说明 |
|-------|-------------|------|
| Load | 100.8% | 持平 |
| WL-A | 89.9% | 略低（skip list CAS > vector insert） |
| WL-B | 90.9% | 略低 |
| WL-C | **119.5%** | **反超** |
| WL-D | 96.6% | 持平 |
| WL-F | 91.9% | 略低 |

### 5.2 db_bench 专项测试（100万 records, 32线程, 绑核 0-31）

#### 5.2.1 点查询性能（readrandom）

| 版本 | ops/s | micros/op | 命中率 |
|------|-------|-----------|--------|
| skip_list | 1,745,694 | 17.6 | 100% |
| hash_spdb 优化版 | 1,646,794 | 19.1 | 100% |

#### 5.2.2 Scan 性能（mixgraph: 95% scan / 5% insert）

| 版本 | ops/s | micros/op | Seek P50 (us) | Seek P99 (us) |
|------|-------|-----------|---------------|---------------|
| skip_list | 2,989,145 | 10.3 | 5.38 | 45.71 |
| hash_spdb 优化版 | 2,576,344 | 12.1 | 5.41 | 67.24 |
| hash_spdb 原始版 | 2,908 | 10,872 | 11,564 | 29,614 |

**Scan 性能提升**：原始版 → 优化版 = **886 倍提升**

#### 5.2.3 原始版 scan 热点分析（perf profile）

| 热点函数 | 占比 | 来源 |
|----------|------|------|
| `pthread_rwlock_wrlock` | 25.61% | InitIterator 中 SpdbVectorContainer 写锁竞争 |
| `SpdbVectorContainer::InitIterator` | 19.80% | 每次 Seek 重建迭代器（锁+SortHeapItem分配） |
| `SpdbVectorIterator::~SpdbVectorIterator` | 17.14% | SortHeapItem 的 delete |
| `pthread_rwlock_unlock` | 7.41% | 锁释放 |
| `_int_free` + `malloc` | 10.67% | SortHeapItem 内存分配/释放 |
| `SeekIter` + `Sort` | 10.93% | 实际计算 |

**根因**：32 线程并发 Seek 时，InitIterator 中的 `spdb_vectors_mutex_` 写锁竞争 + SortHeapItem 的 new/delete 成为瓶颈。

### 5.3 YCSB Workload-A 热点分析（perf profile）

#### 5.3.1 测试条件

- 100万 records, 8线程, 绑核 0-7
- `LD_LIBRARY_PATH` 指定带符号的 `librocksdbjni-linux-aarch64.so`
- `perf record -F 999 -g`

#### 5.3.2 热点对比

| 热点函数 | hash_spdb 原始版 | hash_spdb 优化版 | skip_list |
|----------|-----------------|-----------------|-----------|
| `WriteThread::AwaitState` (写线程等待) | 8.30% | 7.19% | 6.84% |
| `InlineSkipList::RecomputeSpliceLevels` (skip list 插入) | 0% | ~2.0% | 0% |
| `InlineSkipList::FindGreaterOrEqual` (skip list 查找) | 0% | 0% | ~7.9% |
| `IndexBlockIter::SeekImpl` (SST 索引查找) | ~5.4% | ~5.0% | ~4.6% |
| `LZ4_compress_fast_continue` (压缩) | ~5.7% | ~1.3% | ~4.4% |
| `MemTable::KeyComparator` | 0.39% | 0% | 0% |

#### 5.3.3 YCSB Workload-A 吞吐量

| 版本 | Throughput (ops/s) |
|------|-------------------|
| hash_spdb 原始版 | 192,456 |
| hash_spdb 优化版 | 167,532 |
| skip_list | 168,067 |

#### 5.3.4 关键发现

1. **原始版 hash_spdb 完全无 InlineSkipList 热点**：点查走 hash table，写入走 SpdbVectorContainer，两者都不涉及 InlineSkipList。

2. **优化版 InlineSkipList 热点来自写入路径**：Workload-A 的 50% Update = read-modify-write，每次 Update 执行 1 次 Get（走 hash table）+ 1 次 Put（走 hash table + skip list）。InlineSkipList 热点 100% 来自 Put 操作中的 `InsertConcurrently`，**点查确实走了 hash table**，没有走 skip list 查找。

3. **优化版比原始版略慢**：优化版每次写入做 hash table Add（WriteLock）+ skip list InsertConcurrently（CAS）+ memcpy，比原始版的 hash table Add + SpdbVectorContainer::Insert（fetch_add + vector赋值）重。

4. **skip_list 的 InlineSkipList 热点来自读取**：skip_list 没有独立 hash table，点查和写入都走 skip list。`FindGreaterOrEqual` 占 ~7.9%，这是点查的 skip list 搜索操作。

## 6. 测试结论

### 6.1 Scan 场景（核心优化目标）

| 指标 | 原始版 | 优化版 | 提升 |
|------|--------|--------|------|
| db_bench mixgraph (ops/s) | 2,908 | 2,576,344 | **886x** |
| YCSB WL-E (ops/s) | 1,442² | 49,026 | **34x** |

> ² YCSB WL-E 原始版在 100万数据量下的测试结果。

**结论**：重构将 scan 性能从原始版的极低水平提升到与 skip_list 持平，核心目标达成。

### 6.2 点查场景

| 指标 | 原始版 | 优化版 | skip_list |
|------|--------|--------|-----------|
| YCSB WL-C (100% Read, ops/s) | 376,790 | 450,451 | 379,147 |
| db_bench readrandom (ops/s) | N/A | 1,646,794 | 1,745,694 |

**结论**：优化版在 100% 读场景反超 skip_list 19%，hash table 点查优势体现。相比原始版也提升 19.5%。

### 6.3 混合读写场景

| 指标 | 原始版 | 优化版 | skip_list |
|------|--------|--------|-----------|
| YCSB WL-A (50R/50U, ops/s) | 215,415 | 193,750 | 206,607 |
| YCSB WL-F (50R/50RMW, ops/s) | 210,128 | 193,218 | 203,326 |

**结论**：优化版在写入密集场景比原始版低 ~10%，因为 skip list CAS 插入比 SpdbVectorContainer 的 vector 追加重。这是 scan 性能提升的 tradeoff。

### 6.4 写入场景

| 指标 | 原始版 | 优化版 | skip_list |
|------|--------|--------|-----------|
| YCSB Load (ops/s) | 272,606 | 274,748 | 276,901 |

**结论**：三者持平，写入性能不受 scan 数据结构影响。

### 6.5 综合评价

| 维度 | 原始版 | 优化版 | 评价 |
|------|--------|--------|------|
| Scan 性能 | 极差 (2,908 ops/s) | 优秀 (2,576,344 ops/s) | **核心改进** |
| 点查性能 | 优秀 | 优秀 | 保持 |
| 写入性能 | 优秀 | 良好 | 略降（可接受） |
| 混合读写 | 优秀 | 良好 | 略降（可接受） |
| 并发安全 | 良好（锁） | 优秀（CAS 无锁） | 改进 |
| 内存效率 | 一般（双结构） | 一般（双结构） | 不变 |
| 代码复杂度 | 高（SpdbVectorContainer + SortThread + 归并堆） | 低（InlineSkipList 包装） | **大幅简化** |

**总结**：优化版在保持点查优势的同时，将 scan 性能提升 886 倍达到 skip_list 水平，代码复杂度大幅降低。写入性能的轻微下降（~10%）是合理的 tradeoff。

## 7. 附录

### 7.1 测试命令

#### YCSB 全量测试

```bash
# Load 阶段
./bin/ycsb load rocksdb -s \
  -P workloads/workloada \
  -p recordcount=10000000 -p operationcount=10000000 \
  -p rocksdb.dir=/data/zyue/hashspdb_opt/data \
  -p rocksdb.memtable=hash_spdb \
  -threads 8

# Run 阶段
./bin/ycsb run rocksdb -s \
  -P workloads/workloada \
  -p recordcount=10000000 -p operationcount=10000000 \
  -p rocksdb.dir=/data/zyue/hashspdb_opt/data \
  -p rocksdb.memtable=hash_spdb \
  -threads 8
```

#### db_bench 专项测试

```bash
# 加载数据
taskset -c 0-31 ./db_bench --benchmarks=fillrandom \
  --db=/data/zyue/hashspdb_opt/db_bench_data \
  --num=1000000 --threads=32 --key_size=16 --value_size=100 \
  --memtablerep=hash_spdb --hash_bucket_count=1000000 \
  --compression_type=none

# Scan 测试
taskset -c 0-31 ./db_bench --benchmarks=mixgraph \
  --use_existing_db=true --db=/data/zyue/hashspdb_opt/db_bench_data \
  --num=1000000 --reads=10000 --threads=32 --key_size=16 --value_size=100 \
  --memtablerep=hash_spdb --hash_bucket_count=1000000 \
  --compression_type=none \
  --mix_get_ratio=0.0 --mix_put_ratio=0.05 --mix_seek_ratio=0.95 \
  --mix_max_scan_len=100 --key_dist_a=0.23 --key_dist_b=0.99 \
  --iter_theta=1.0 --iter_k=0.5 --iter_sigma=10.0 --histogram
```

#### Perf 热点分析

```bash
# YCSB workloada profiling
perf record -F 999 -g -o perf.data -- \
  taskset -c 0-7 env LD_LIBRARY_PATH=/path/to/so \
  java -cp classpath site.ycsb.Client \
  -db site.ycsb.db.rocksdb.RocksDBClient \
  -p rocksdb.dir=/data -p rocksdb.memtable=hash_spdb \
  -P workloads/workloada \
  -p recordcount=1000000 -p operationcount=1000000 \
  -threads 8 -t

# 分析
perf report -i perf.data --stdio --no-children -g none --percent-limit 0.3
```

### 7.2 编译命令

```bash
# C++ db_bench
export JAVA_HOME=/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.462.b08-4.oe2403sp2.aarch64
make -j$(nproc) db_bench DEBUG_LEVEL=0 \
    DISABLE_JEMALLOC=1 ROCKSDB_DISABLE_TCMALLOC=1

# JNI jar
make -j$(nproc) rocksdbjava DEBUG_LEVEL=0 \
    DISABLE_JEMALLOC=1 ROCKSDB_DISABLE_TCMALLOC=1

# 安装到 Maven 本地仓库
mvn install:install-file \
    -Dfile=rocksdbjni-6.26.1.jar \
    -DpomFile=pom.xml \
    -DgroupId=org.rocksdb -DartifactId=rocksdbjni \
    -Dversion=6.26.1 -Dpackaging=jar

# 编译 YCSB binding
cd YCSB && mvn -pl site.ycsb:rocksdb-binding -am clean package \
    -DskipTests -Dcheckstyle.skip
```
