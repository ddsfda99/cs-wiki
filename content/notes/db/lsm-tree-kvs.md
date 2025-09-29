---
title: "LSM-tree model for KVS"
date: 2025-09-22
---
# LSM-tree model for KVS
## 1. B+树
**基本概念**  
B+树是一种多路平衡查找树（multi-way balanced search tree）。
**B+树的结构特点**
（1）平衡性
B+树始终保持平衡，所有叶子节点在同一层。
查询任何数据，访问路径长度相同（避免树高不均带来的查询性能抖动）。
**优缺点**  
优点：读性能高，尤其是范围查询非常高效。  
缺点：需要随机写入。对于 HDD，随机写比顺序写慢几十倍；即使在 SSD 上，也会慢 3–10 倍。
**结论**：传统 B+树对写入型场景不友好，尤其在机械硬盘时代。

## 2. 日志结构化存储 (Log-structured Storage)

- **核心思想**：通过 **顺序写（append-only）** 避免随机写放大。  
- **方式**：将所有修改以日志的形式 **追加** 到文件末尾。

### 示例
1. **日志型 B+树（Copy-on-Write B+tree）**  
   修改叶子节点时，采用写时复制（COW）技术，避免在原地修改。
   
2. **Bitcask 架构**  
   - 基于内存哈希表 + 磁盘日志文件的 KV 存储。  
   - 应用：Berkeley DB、豆瓣 BeansDB、RamCloud。  
   - **特点**：  
     - 写操作只追加到日志文件。  
     - 内存保存哈希索引（file_id, value_pos, value_sz）。  
     - 文件大小有限，超过阈值会新建文件。  
     - 删除时写一个“墓碑记录”，不直接删除旧数据。  
     - 定期执行 **merge** 操作，清理冗余。

📌 **问题**：写入快，但范围查询不友好。

---

## 3. LSM-tree 模型

**Log-structured Merge-tree (LSM-tree)**：结合日志式写入与多层归并的存储结构。

- **应用**：Bigtable、HBase、LevelDB、RocksDB、TiDB、MongoDB、Cassandra
- **关键组件**：
  1. **WAL（Write-Ahead Log）**：写前日志，保证崩溃恢复。  
  2. **Memtable**：内存有序表（LevelDB 用 SkipList）。  
  3. **SSTable**：磁盘文件，只读有序。  
  4. **Compaction**：后台合并，去除冗余。  

---

## 4. SSTable 文件逻辑布局

SSTable（Sorted String Table）是一种 **不可变的有序存储文件**。

```

Data Block 1
Data Block 2
...
Meta Block 1
Meta Block 2
Meta Block Index
Index Block
Footer

```

- **数据存储区**：存放实际 KV 数据（有序）。  
- **数据管理区**：包含索引、元数据，加速查找。  
- **Footer**：保存元数据和索引的偏移。

特点：
- 数据有序，支持二分查找。  
- 通常配合 Bloom Filter 加速点查。  
- 文件不可修改，只能追加新文件。

---

## 5. 层次设计

- **L0 层**：文件范围可能重叠，查询需遍历多个文件。  
- **L1-L6 层**：同层文件范围不重叠，每层只需查一个文件。  
- **Compaction**：保证下层文件不重叠，提高查询性能。

## 6. 写操作流程

1. 追加写入 WAL（顺序写）。  
2. 插入 Memtable（跳表实现）。  
3. Memtable 达到阈值后，**flush** 成新的 SSTable。  
4. 后台触发 Compaction，合并旧文件，清理冗余。

---

## 7. 读操作流程

1. 查询 Memtable。  
2. 查询 Immutable Memtable（已冻结，待 flush）。  
3. 查询 L0 层 SSTables（可能多个）。  
4. 查询 L1-Ln，每层最多命中一个文件。

优化：
- **SSTable Cache**  
- **Bloom Filter / Ribbon Filter**

---

## 8. Compaction（压缩合并）

- **触发条件**：某层数据量达到阈值。  
- **流程**：
  1. 在 L 层选择一个 SSTable。  
  2. 在 L+1 找到重叠文件集合 Y。  
  3. 合并 L 层的相关文件 Z 和 Y。  
  4. 输出新文件到 L+1。  

- **类型**：
  - **Minor Compaction**：Memtable flush 到磁盘。  
  - **Major Compaction**：全量排序合并多个文件。

---

## 9. 优化点

- **读优化**
  - Block Cache / Row Cache
  - 减少层数
  - 使用 Bloom Filter 或 Ribbon Filter
  
- **写优化**
  - 减少写放大（WA = 实际写入量 / 原始数据量）
  - 控制 Compaction 速率  

📌 写放大示例：写 1KB 数据，系统实际写入 1MB。

---

## 10. LSM-tree vs B+树

| 特性        | LSM-tree | B+树 |
|-------------|----------|------|
| 写入性能    | 顺序写，写快 | 随机写，写慢 |
| 读取性能    | 跨层查找，依赖 filter/cache | 范围查好，点查快 |
| 删除/更新   | 写墓碑，后台清理 | 直接修改节点 |
| 空间效率    | 压缩率高 | 压缩有限 |
| 适用场景    | 写多读少，分布式 KV | 读多写少，事务型 DB |

---

## 11. 扩展阅读

- **研究热点**：LSM-tree 存储优化  
- 推荐：*LSM-based storage techniques: a survey*，VLDB Journal:contentReference[oaicite:1]{index=1}

---

# 总结

LSM-tree 是现代 KV 存储引擎的核心设计，特点是：
- **写快（顺序写 + flush）**
- **读性能可接受（索引 + Bloom Filter + Cache）**
- **后台 Compaction 保证数据有序**
- **适合写多读少、大规模数据场景**


